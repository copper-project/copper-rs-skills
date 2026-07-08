---
name: copper-rs-api-flavor
description: >-
  How to shape new user-facing traits and adapters in the copper-rs (`cu29`) codebase so
  they match the maintainer's design taste. Use when proposing or reviewing any new
  task-like trait, adapter over `CuTask`/`CuSrcTask`/`CuSinkTask`, RON-configured
  enum/policy field, or any API that touches the hot path. Complements `copper-rs`
  (architecture) and `copper-rs-coding-style` (formatting/naming/logging).
---

# copper-rs API flavor — how new task APIs must feel

The copper-rs runtime has a **very specific taste** for what a task-facing API looks like.
New traits or adapters that don't match it will get rejected on aesthetic grounds even
when the logic is correct. This skill distills that taste into hard rules, each with
the exact codebase reference that established it.

Authority: `core/cu29_runtime/src/cutask.rs` for the trait shapes; existing
`components/tasks/*` for canonical usage; `core/cu29_runtime/src/config.rs` for config.

## The five non-negotiables

### 1. Outputs are `&mut` handles, never return values

The runtime already owns the output slot in the copperlist. Task code **writes into it**;
it does not construct-then-return a payload. This avoids a copy on every cycle and keeps
the RT path zero-alloc.

Look at every existing task (`cutask.rs:498, 572–577, 650`):

```rust
// CuSrcTask
fn process<'o>(&mut self, ctx: &CuContext,
               new_msg: &mut Self::Output<'o>) -> CuResult<()>;
// CuTask
fn process<'i, 'o>(&mut self, ctx: &CuContext,
                   input: &Self::Input<'i>,
                   output: &mut Self::Output<'o>) -> CuResult<()>;
// CuSinkTask
fn process<'i>(&mut self, ctx: &CuContext,
               input: &Self::Input<'i>) -> CuResult<()>;
```

Inputs come in as `&CuMsg<T>` (or a tuple of refs). Outputs go out as `&mut CuMsg<T>`
written via `output.set_payload(...)` / `output.clear_payload()` / mutating fields in
place. Return value is always `CuResult<()>`.

**Rule for new traits.** This applies to **every** method that produces payload state,
including read-only-looking accessors. A `fn best(&self) -> Self::Output` still fails
the rule — even though the receiver is `&self`, the return value is owned, so the
adapter has to clone into the output slot. Rewrite it to write into a caller-owned
buffer: `fn write_best(&self, out: &mut Self::Output)` — or accept a `&mut Self::Output`
directly on the step method that produced the state. No method on a hot-path trait should
own-return payload data.

Concretely, for an anytime/iterative-refinement trait: don't write

```rust
fn best(&self) -> Self::Output;                 // BAD — forces a copy
```

Write instead something like

```rust
fn write_best(&self, out: &mut Self::Output);   // GOOD — in-place, zero copy
```

and have the adapter's `CuTask::process` pass `output.payload_mut()` (initialised via
`set_payload(Default::default())` if needed) through to the user code.

### 2. Enum-shaped config uses `get_value` + `serde::Deserialize`, NEVER stringly parsed

`ComponentConfig` (`core/cu29_runtime/src/config.rs:115`) has two accessors:

- `get::<T>(key)` — scalars (`bool`, integers, floats, `String`) via `TryFrom<&Value>`.
- `get_value::<T>(key)` — anything `T: DeserializeOwned`, deserialised out of the RON
  value. This is how structured config works in this codebase
  (`cu_peer_triangulation/src/lib.rs:220`, `cu_peer_range_accumulator/src/lib.rs:130`,
  `cu_instance_overrides_demo/src/main.rs:30`).

**Rule.** A policy/mode field must be a real Rust enum with `Deserialize` derived, read
via `get_value`:

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum OnOverload {
    #[default] ReuseLast,
    Fail,
}

let policy = cfg.get_value::<OnOverload>("on_overload")?.unwrap_or_default();
```

Do **not** write a hand-rolled `match s { "reuse_last" | "ReuseLast" => ... }` parser.
The maintainer will flag both the string-based dispatch ("stringly type thing") and any
"either casing works" fallback ("we are defining right now the API, why does it need to
be fuzzy?"). Pick one canonical serialised spelling — snake_case — via serde attributes
and let the deserializer be the sole source of truth. No dual naming, no free-form parse.

### 3. Inputs stay in the copperlist — don't make tasks cache them

The copperlist already holds the input message for the whole cycle. Any per-cycle work
that needs the input should **receive it as a parameter**, not require the task to stash
it in `self` between calls.

If you're designing a multi-step user trait (e.g. `base(input)` then a loop of
`refine()`), give `refine` the input too, or refactor so the adapter drives the loop
with the input in scope. Forcing the user to write

```rust
struct MyThing { cached_input: Option<Input>, ... }
```

is a design smell in this codebase — the runtime already stores it, doing it a second
time doubles memory and invites staleness bugs.

### 4. Payload traits, Tov, and bounds live where the codebase already puts them

**Payload derives.** The mandatory set is exactly what `CuMsgPayload` requires
(`cutask.rs:28–46`):

```rust
#[derive(Default, Debug, Clone, Encode, Decode, Serialize, Deserialize, Reflect)]
```

`PartialEq` is **not** required by `CuMsgPayload` — many payloads add it because it's
useful in tests or for equality checks (`cu_ads7883/src/lib.rs:51`,
`cu_peer_range_accumulator/src/lib.rs:13`), and plenty skip it (`cu_ahrs/src/lib.rs:46`,
`cu_pid` payloads). Add it if your payload wants it; don't advertise it as part of the
contract.

**Task-struct derives.** Every task struct derives `Reflect`. The extra attribute
`#[reflect(no_field_bounds, from_reflect = false, type_path = false)]` is needed for
**generic wrappers and structs with opaque/FFI fields** — see `cu_ratelimit/src/lib.rs:8`
(generic `<T>`), `cu_python_task/src/lib.rs:620` (PyO3 handles), `cu_mpu9250`, `cu_bmi088`,
`cu_dps310`, `cu_gnss_ublox` (hardware/driver fields that don't implement `Reflect`).
Simple non-generic task structs get away with a plain `#[derive(..., Reflect)]` —
`cu_pid/src/lib.rs:19` is the canonical example. Use `#[reflect(ignore)]` on individual
fields that don't derive `Reflect`.

**`Tov` lives on the `CuMsg` envelope, not in the payload.** Propagate it via
`output.tov = input.tov;` or seed from the clock: `new_msg.tov = ctx.now().into();`
(`cu_ads7883/src/lib.rs:164`, `cu_ahrs/src/lib.rs:349–403`). Don't design payload
wrappers that smuggle timestamps back into the data — that's what the envelope is for.

**Bounds on associated payload types.** Use `CuMsgPayload` as-is. No ad-hoc weakening
("just `Default + Clone` is enough for my case"), no ad-hoc strengthening (`Send + Sync`,
`'static`, extra trait bounds) unless the feature genuinely needs it and you can point at
the specific method that requires it. `CuMsgPayload` itself carries no `Send/Sync/'static`
bound; the task traits don't either. Piling them on "just in case" narrows what payloads
users can plug in and drifts the API from every other task in the tree.

### 5. Mirror the `CuTask` lifecycle exactly

`new`, `start`, `preprocess`, `process`, `postprocess`, `stop`, plus `Freezable`
(`freeze`/`thaw`). Every user-facing task trait — including adapters — must expose
the same lifecycle with the same signatures (`&mut self, ctx: &CuContext`, returning
`CuResult<()>`). Don't invent new lifecycle names; don't drop `Freezable`. `freeze`/
`thaw` default to no-op which is fine for stateless work — see `cutask.rs:412–425`.

## The taste check (before you send the PR)

Ask, honestly:

1. Does any method **return** payload by value? → rewrite to `&mut out`.
2. Does any config field get parsed from a string by hand? → switch to
   `Deserialize` + `get_value`.
3. Does the trait force the user to **cache** an input the copperlist already holds?
   → pass it in each time, or restructure the adapter loop.
4. Do payload derives cover the `CuMsgPayload` bound (no ad-hoc weaker set)? Does the
   task struct derive `Reflect` (with the `no_field_bounds/from_reflect/type_path`
   attribute only if it's generic or holds opaque/FFI fields)? Is `Tov` on the envelope,
   not in the payload? Bounds on associated payload types stay at `CuMsgPayload` unless
   a specific method demands more?
5. Is the lifecycle 1:1 with `CuTask`, with `Freezable`? → if not, align it.

Only after all five are clean does the API "feel like copper-rs". Reviews from the
maintainer (`gbin`) enforce these on the diff line, not in prose — so it's cheaper
to bake them in from the first sketch.

## Failure exemplar

PR #1180 (the anytime-task PR) is the concrete example these rules were extracted
against. Its first-round review flagged, in order: `best(&self) -> Output` (rule 1),
`refine` not receiving input (rule 3), the `OnOverload::parse` string-match with
`"reuse_last" | "ReuseLast"` (rule 2, both "stringly typed" and "fuzzy"). Anything
new should not repeat those shapes.
