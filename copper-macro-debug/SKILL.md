---
name: copper-macro-debug
description: >-
  Debug the `#[copper_runtime]` proc macro in copper-rs (`cu29_derive`): read
  its compile errors, decide when to run `cargo expand` / `just expand-runtime`,
  understand what the macro emits around the user's `App` struct, and recognise
  the recurring foot-guns (missing `Reflect`/`Freezable`, RON-parse-as-macro-error,
  `sim_mode` divergence, feature-gated lifecycle hooks). Use whenever the
  compiler error mentions `#[copper_runtime]`, points at the `App` struct line,
  or names a symbol that only exists in expanded code. For everyday task/RON
  authoring see `copper-arch`; for `just` commands see `copper-workflow`.
---

# copper-rs macro debugging

`#[copper_runtime(config = "copperconfig.ron", sim_mode = false)]` is the single
proc-macro that turns a declarative RON task graph plus a user-written marker
struct into a concrete `App` with `new`, `run`, `start_all_tasks`, the typed
copperlist, the resource bundle, and the per-step calls into every task's
lifecycle methods. The macro lives in `core/cu29_derive/src/lib.rs` (~9.4k lines)
and **executes at compile time** — it reads the RON file from disk, resolves
every task/source/sink/bridge type, computes the execution plan, and emits Rust.

This skill is about the failure modes specific to that macro. For "what does
the macro do conceptually" see `copper-arch` (the architecture skill).

## When to invoke

Trigger on any of these signals:

- A compile error span points at `#[copper_runtime(...)]` or the line below it.
- An error message contains `to_compile_error`, `proc-macro panicked`,
  `cannot find type ... in this scope` where the missing symbol is something
  like `CuStampedDataSet`, `CuList`, `App`, `CuTasks`, `__copper_runtime_*`.
- A trait error names a method the user never wrote (`start`, `preprocess`,
  `process`, `postprocess`, `stop`, `freeze`, `thaw`) on the `App` struct.
- The compiler complains about `Reflect`, `Freezable`, `Encode`, `Decode`,
  `bincode`, or `Serialize`/`Deserialize` on a task type the user just added.
- Behaviour differs between a normal binary and a `sim_mode = true` binary
  built from the same config.

## Mental model of what the macro emits

The macro takes the *marker* `struct App;` (or `pub struct App { ... }`) and
rewrites it into a full runtime. After expansion the type contains:

- `copper_runtime: CuRuntime<...>` — the engine (always added).
- One typed field per task in the RON `tasks: [...]` list.
- A generated `CuStampedDataSet` (the concrete copperlist payload type).
- A generated `CuTasks` tuple over every task.
- A generated `impl App { fn new(...) -> CuResult<Self>; fn run(...) -> ...;
  fn start_all_tasks(...); fn stop_all_tasks(...); ... }`.
- Per-step blocks that call `task.preprocess(ctx)?; task.process(...)?;
  task.postprocess(ctx)?;` in execution order.
- Optional `rtsan` / `memory_monitoring` scope guards around each task step,
  emitted **conditionally on cargo features** (`feature = "rtsan"`,
  `feature = "memory_monitoring"`). Default builds elide them.

The user's source therefore does *not* contain — but the compiler thinks does:
`App::new`, `App::run`, `App::start_all_tasks`, `CuStampedDataSet`,
`CuTasks`, every per-task field name. **If the IDE/jump-to-definition fails on
one of these, you are looking for generated code; expand the macro.**

## How to expand

`just expand-runtime pkg=<crate> bin=<bin> [features=feat1,feat2]` runs
`cargo +stable expand -p <crate> --bin <bin>`. Output is thousands of lines;
pipe through the repo's `support/agent/agent-expand.sh` (if present) to get
just the section around a symbol of interest, otherwise pipe to `less` and
search.

When you need an even narrower slice, `grep -n` for one of the generated symbol
names listed above to land near the interesting region — `fn run(`,
`impl App`, `CuStampedDataSet`, or `task.process(`.

## Recurring failure modes

### 1. "Cannot find type `Foo` in this scope" inside generated code

The macro resolves task `type: "my_crate::Foo"` strings from RON by literally
parsing them as Rust paths. Causes:

- The RON `type` string is misspelled or the crate isn't a path-dependency of
  the binary's crate. **Fix in RON, not in code.**
- The crate is feature-gated and the relevant feature is off for this build.
  Try the build with the right `--features`.
- A re-export the user added in `lib.rs` doesn't apply because the macro
  resolves at the binary crate root, not via re-exports of a sibling crate.

### 2. "the trait bound `MyTask: Reflect` is not satisfied"

Every task type must derive `Reflect` (in addition to `Default + Debug`). The
copper-rs Reflect derive lives in `cu29_runtime::reflect`; importing
`cu29::prelude::*` brings it in. Missing `Reflect` shows up only when the
macro expands and tries to register the task. Same applies to `Freezable` —
even *stateless* tasks need the trait (use `impl_default_freeze!`).

### 3. "the trait bound `MyPayload: Encode` is not satisfied" (or `Decode`, `Serialize`, `DeserializeOwned`)

Payloads must satisfy the full `CuMsgPayload` bound set: `Default + Debug +
Clone + Encode + Decode(()) + Serialize + DeserializeOwned + Reflect`. If any
one is missing the error points at the *macro-emitted* copperlist type, not at
the payload definition. Re-derive the missing trait on the payload type.

### 4. RON parse errors surface as macro errors

The macro reads the RON file with `read_to_string` and calls the runtime
parser. Bad RON does **not** produce a friendly parse-error pointer at the RON
file — it produces a `compile_error!` at the `#[copper_runtime(...)]` span
saying e.g. `unexpected ident`. Always re-format the RON with `fmtron` (or
`support/agent/agent-ron-fmt.sh`) before assuming a code-level bug.

### 5. `proc-macro panicked`

The macro has several `panic!`/`unwrap_or_else(|| panic!(...))` sites for
"impossible" cases — missing output packs, unknown task type, codec resolution
failures. The panic message is informative; read it. Common root causes:

- A task ID referenced in `cnx:` is not in `tasks:`.
- A bridge's `from`/`to` references a non-existent endpoint.
- A `payload_type` in `cnx:` cannot be resolved as a Rust path (same root cause
  as #1, different symptom).
- A resource bundle in RON references a HAL resource not registered for the
  active platform.

### 6. `sim_mode = true` produces a different shape

When `sim_mode = true`, the macro emits a runtime that **does not own real
tasks**; instead each task slot is a `SimTask` callback the user provides at
runtime. The generated `App::new` signature gains a `sim_callback` parameter.
Two consequences:

- A real-mode and sim-mode binary built from the same RON will have
  *different* `App::new` arities. The sim binary is a separate `main.rs`.
- `ignore_resources = true` is **only** legal with `sim_mode = true` — the
  macro emits a `compile_error!` otherwise.

### 7. Lifecycle hooks "missing" in expanded code

The macro can elide per-step hooks for certain feature combinations:

- `rtsan` feature off → no `ScopedSanitizeRealtime` guard inserted.
- `memory_monitoring` feature off → no `__cu_alloc_scope` open/close.
- `log-level-*` features below the call's level → `debug!`/`info!` macro calls
  compile to nothing.

If you're chasing "why doesn't this instrumentation fire," check the feature
flags before assuming a code bug.

### 8. `subsystem = "..."` vs flat config

A `subsystem` argument means the RON is a *strict-multi-Copper* config and
the macro will only embed the named sub-graph. If the RON is a flat config,
omitting `subsystem` is correct; passing `subsystem` to a flat config is a
hard error. If the RON is multi-Copper, omitting `subsystem` is a hard error.

## Debugging recipe

1. **Read the error span first.** If it points at `#[copper_runtime(...)]`
   the macro itself rejected something — usually RON-resolution or a missing
   derive on a task/payload type listed below.
2. **If a symbol is missing**, decide: is it a user symbol (fix the user code),
   a copper-rs prelude symbol (add `use cu29::prelude::*;`), or a
   macro-generated symbol (the macro didn't emit it because of an earlier
   error or a feature flag)?
3. **If the message says `proc-macro panicked`**, the message body is the bug
   report — read it, then check the RON for the referenced ID.
4. **If you still can't tell what's going on, expand.** `just expand-runtime
   pkg=<bin-crate> bin=<bin> features=<features>` and search for the symbol
   that's giving trouble. The expanded `impl App { ... }` is the source of
   truth.
5. **Reformat RON with `fmtron` before re-running** the build — a tab-vs-space
   change after manual editing is enough to throw the macro.

## What this skill is *not*

- Not the RON schema reference — see `copper-ron-config` for every key/field and
  the exact validation errors the parser emits.
- Not a guide to the runtime's execution model — see `copper-arch`.
- Not a guide to implementing sources/tasks/sinks/bridges — see
  `copper-component-design`.
- Not the debug/replay runtime playbook — see `copper-debug-replay` for
  extract-copperlists, resim, and the `debug.v1` remote-debug protocol.
- Not a guide to logging macros (`debug!`/`info!` syntax restrictions) — see
  `copper-coding-style`.
