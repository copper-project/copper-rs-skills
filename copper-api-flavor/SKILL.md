---
name: copper-api-flavor
description: >-
  How to shape new user-facing traits and adapters in the copper-rs (`cu29`) codebase so
  they match the maintainer's design taste. Use when proposing or reviewing any new
  task-like trait, adapter over `CuTask`/`CuSrcTask`/`CuSinkTask`, RON-configured
  enum/policy field, or any API that touches the hot path. Complements
  `copper-core-dev` (runtime design constraints), `copper-arch` (execution model),
  and `copper-coding-style` (formatting/naming/logging).
---

# copper-rs API flavor — how new task APIs must feel

Every rule below exists because of a concrete ownership, real-time, or replay
constraint in the runtime — none is a style preference. Each section states the
constraint first, then the API rule that follows from it, with the codebase references
that established it.

Authority: `core/cu29_runtime/src/cutask.rs` for the trait shapes; existing
`components/tasks/*` for canonical usage; `core/cu29_runtime/src/config.rs` for config.

## The non-negotiables

### 1. The runtime owns output memory — write through `&mut`

The runtime allocates and owns each output slot as part of the `CopperList`, and it
controls that memory's lifecycle through processing, logging, and reuse. A task receives
temporary mutable access so it can fill the slot in place; it does not take ownership of
the slot or return a separately owned result. This avoids an extra payload copy or clone
on every cycle and keeps the RT path zero-alloc.

Look at the three `process` signatures in `cutask.rs`:

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

**API consequence.** No method on a hot-path trait should own-return payload data —
including read-only-looking accessors. Even with a `&self` receiver, an owned return
value forces the adapter to clone it into the output slot:

```rust
fn best(&self) -> Self::Output;                 // BAD — forces a copy
fn write_best(&self, out: &mut Self::Output);   // GOOD — in-place, zero copy
```

The adapter's `CuTask::process` passes `output.payload_mut()` (initialised via
`set_payload(Default::default())` if needed) through to the user code.

### 2. Strong typing over stringly typing — an enum is an index, a string is a parser

This is a general API principle, not just a config rule: any closed set of choices — a
policy, a mode, a variant selector — is a Rust enum, never a `String`. The mechanics
decide it. An enum compiles down to a discriminant — an index — that is cheap to store,
copy, and `match` on, allocation-free on the hot path, and exhaustiveness-checked, so a
missed variant is a compile error. A stringly-typed value allocates, dispatches by
runtime string comparison, can't be exhaustiveness-checked, fails at runtime instead of
compile time, and invites fuzzy aliases.

RON config is where this bites most often, because config is part of the public API.
Letting serde deserialize a Rust enum keeps the type and its accepted serialized
spellings in one declaration, with one error path for unknown values. A hand-written
string parser duplicates that schema and tends to grow aliases that make the API fuzzy.

`ComponentConfig` (`core/cu29_runtime/src/config.rs`) has two accessors:

- `get::<T>(key)` — scalars (`bool`, integers, floats, `String`) via `TryFrom<&Value>`.
- `get_value::<T>(key)` — anything `T: DeserializeOwned`, deserialised out of the RON
  value. This is how structured config works in this codebase
  (`cu_peer_triangulation/src/lib.rs`, `cu_peer_range_accumulator/src/lib.rs`,
  `cu_instance_overrides_demo/src/main.rs`).

**API consequence.** A policy/mode field must be a real Rust enum with `Deserialize`
derived, read via `get_value`:

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
In review the maintainer flags both halves of that pattern: the stringly-typed dispatch
itself, and the accept-several-spellings leniency — a new API is being defined from
scratch, so there is no legacy input to be lenient toward. Pick one canonical serialised
spelling — snake_case — via serde attributes and let the deserializer be the sole source
of truth. No dual naming, no free-form parse.

### 3. The `CopperList` owns per-cycle inputs — borrow instead of caching

The `CopperList` already holds the input message for the whole cycle. Any per-cycle work
that needs the input should borrow that single source of truth. Caching it in the task
duplicates runtime-owned data, consumes memory, and creates a stale copy that can outlive
the cycle it came from.

**API consequence.** Receive the input as a parameter instead of requiring the task to
stash it in `self` between calls.

For a multi-step trait whose steps all run within one cycle, give every step the
input, or refactor so the adapter drives the loop with the input in scope — don't
push duplicate ownership onto the user
(`struct MyThing { cached_input: Option<Input>, ... }`).

**The carve-out — work that outlives the copperlist.** This rule holds only while
the copperlist outlives the work. `CuAnytimeTask`
(`core/cu29_runtime/src/cutask_anytime.rs`) deliberately gives `refine()` **no**
input: in background placements refinement quanta outlive the copperlist that
carried the input, so a borrow cannot legally reach `refine()`, and `base()` is
*required* to capture into the task's own per-job state everything refinement
will need. Capturing derived per-job state there is the correct pattern, not a
violation — the anti-pattern is stashing the raw input "just in case" when a
borrow would do.

### 4. Payload contracts keep storage, tooling, and replay uniform

`CuMsgPayload` defines the capabilities the runtime can rely on when it initializes,
records, restores, and reflects messages. The `CuMsg` envelope similarly gives the
runtime one payload-independent place for time-of-validity and metadata. Recreating those
contracts in each new API would fragment the runtime's message model.

**API consequence.** Reuse `CuMsgPayload` as-is, keep `Tov` on the `CuMsg` envelope, and
add only the bounds that a specific method requires.

**Payload derives.** The mandatory set is exactly what `CuMsgPayload` requires
(`cutask.rs`):

```rust
#[derive(Default, Debug, Clone, Encode, Decode, Serialize, Deserialize, Reflect)]
```

`PartialEq` is **not** required by `CuMsgPayload` — many payloads add it because it's
useful in tests or for equality checks (`cu_ads7883/src/lib.rs`,
`cu_peer_range_accumulator/src/lib.rs`, `cu_ahrs/src/lib.rs`), and others skip it
(`cu_pid` payloads). Add it if your payload wants it; don't advertise it as part of the
contract.

**Task-struct derives.** Every task struct derives `Reflect`. The extra attribute
`#[reflect(no_field_bounds, from_reflect = false, type_path = false)]` is needed for
**generic wrappers and structs with opaque/FFI fields** — see `cu_ratelimit/src/lib.rs`
(generic `<T>`), `cu_python_task/src/lib.rs` (PyO3 handles), `cu_mpu9250`, `cu_bmi088`,
`cu_dps310`, `cu_gnss_ublox` (hardware/driver fields that don't implement `Reflect`).
Simple non-generic task structs get away with a plain `#[derive(..., Reflect)]` —
`cu_pid/src/lib.rs` is the canonical example. Use `#[reflect(ignore)]` on individual
fields that don't derive `Reflect`.

**`Tov` lives on the `CuMsg` envelope, not in the payload** (`pub tov: Tov` in
`cutask.rs`). Don't design payload wrappers that smuggle timestamps back into the
data — that's what the envelope is for. For when to propagate vs stamp `tov` inside a
component, see `copper-component-design`.

**Bounds on associated payload types.** Use `CuMsgPayload` as-is. No ad-hoc weakening
("just `Default + Clone` is enough for my case"), no ad-hoc strengthening (`Send + Sync`,
`'static`, extra trait bounds) unless the feature genuinely needs it and you can point at
the specific method that requires it. `CuMsgPayload` itself carries no `Send/Sync/'static`
bound; the task traits don't either. Piling them on "just in case" narrows what payloads
users can plug in and drifts the API from every other task in the tree.

### 5. Preserve the lifecycle so the runtime can drive every task uniformly

The lifecycle is not a naming convention — each phase is a hook the generated runtime
calls at a distinct moment for a distinct reason (`cutask.rs`, identical across all
three task traits):

- `new(config, resources)` — construction from RON config and declared resources,
  before the loop runs; the one phase with no `CuContext` (no clock, no copperlist).
- `start` / `stop` — the paired bracket around the running period: acquire/release
  what the task holds while live. `stop` guarantees `process` won't be called again
  until the next `start`.
- `preprocess` / `postprocess` — best-effort per-cycle hooks that the generated loop
  packs before the copperlist is created and after it is closed out
  (`cu29_derive/src/lib.rs`). They exist so prep work and non-time-critical
  bookkeeping (stats etc.) stay out of the critical path.
- `process` — the hot path, the only phase stamped into the message's
  `metadata.process_time` window; keep it as short as possible.
- `freeze`/`thaw` (`Freezable`, `cutask.rs`) — task-state snapshot/restore the
  runtime invokes at keyframe intervals (`curuntime.rs`) so deterministic replay
  can restore a stateful task mid-log.

Because the runtime drives every task through exactly these hooks, an adapter that
renames or omits a phase cannot substitute cleanly for `CuTask`; skipping `Freezable`
silently leaves state outside keyframes, so resim diverges (see `copper-arch`).

**API consequence.** Mirror `new`, `start`, `preprocess`, `process`, `postprocess`,
`stop`, plus `Freezable` (`freeze`/`thaw`). Every user-facing task trait — including
adapters — must expose the same lifecycle with the same signatures (`&mut self,
ctx: &CuContext`, returning `CuResult<()>`). Don't rename or drop the bracket
phases; don't drop `Freezable`. A trait may split the `process` slot into several
bounded step methods when its execution model demands it — `CuAnytimeTask`'s
`base`/`refine` is the merged precedent — but the surrounding
`new`/`start`/`preprocess`/`postprocess`/`stop` + `Freezable` bracket stays 1:1.
Mirroring is cheap: everything except `new`/`process` (and `freeze`/`thaw`)
defaults to a no-op, which is fine for stateless work.

### 6. Every `Option` on the hot path is a branch — decide at compile time what can be decided at compile time

Hot-path methods run every tick — `process`, or the step methods of a task-like
trait such as `CuAnytimeTask` (a standalone peer of `CuTask`, not an adapter over
it: an *anytime* task computes a coarse `base` result, then `refine`s it until
told to stop). Each `Option` and each extra enum layer in their signatures is
an `if` the CPU evaluates and the branch predictor tracks on every single tick — even
when 99% of the time the answer is "nothing there". Stack a few and you pay several
predicted branches per tick for no work. Both consequences below were established in
the `CuAnytimeTask` review (copper-rs PR #1205).

**One layer of signaling.** A hot-path method returns `CuResult<Status>`:
`Err(CuError)` means a real failure, a plain status variant means a controlled
outcome (converged, aborted, ...). Never nest the two layers:

```rust
fn refine(&mut self) -> CuResult<Option<Status>>;   // BAD — two ways to say "nothing"
enum Status { Aborted(Option<CuError>), /* … */ }   // BAD — an error hiding in a variant
fn refine(&mut self) -> CuResult<Status>;           // GOOD — one layer
```

A controlled stop is a variant, not an error; a failure is `Err`, not a variant.

**Generics instead of `Option` fields.** The maintainer's standing rule: anything
that *can* be decided at compile time *must* be decided at compile time. When a
capability varies by *task type* — not per tick — encode it in the type system, not
in a runtime `Option`.
The anytime trait's quality report is the canonical example: instead of
`Option<Quality>` in the status, the status is generic —

```rust
pub enum AnytimeStatus<Q> {
    Improved(Q),
    Converged(Q),
    Aborted,
}

// on the trait:
type Quality: Copy + PartialOrd;   // a normalized ratio, or () if the task can't score
```

Tasks that can score their result use a quality type; tasks that can't use `()`.
Monomorphization compiles the branch away, and misconfiguration (a quality target on
a `()` task) becomes a compile error instead of a runtime check.

## The taste check (before you send the PR)

Ask, honestly:

1. Does any method **return** payload by value? → rewrite to `&mut out`.
2. Is any closed set of choices a string instead of an enum — in config or anywhere
   else in the API? → make it an enum; config fields read it via `Deserialize` +
   `get_value`.
3. Does the trait force the user to **cache** an input the copperlist already holds?
   → pass it in each time, or restructure the adapter loop.
4. Do payload derives cover the `CuMsgPayload` bound (no ad-hoc weaker set)? Does the
   task struct derive `Reflect` (with the `no_field_bounds/from_reflect/type_path`
   attribute only if it's generic or holds opaque/FFI fields)? Is `Tov` on the envelope,
   not in the payload? Bounds on associated payload types stay at `CuMsgPayload` unless
   a specific method demands more?
5. Is the lifecycle 1:1 with `CuTask`, with `Freezable`? → if not, align it.
6. Does any hot-path signature stack `Option`s or nest error/status layers?
   → one `CuResult<Status>` layer: `Err(CuError)` for failure, plain variants for
   controlled outcomes. Does a runtime `Option` encode a per-task-type capability?
   → move it into a generic/associated type so it's resolved at compile time.

Only after all of these are clean does the API "feel like copper-rs". Reviews from the
maintainer (`gbin`) enforce these on the diff line, not in prose — so it's cheaper
to bake them in from the first sketch.
