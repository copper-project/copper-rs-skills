---
name: copper-arch
description: >-
  Architecture and task-authoring guide for the copper-rs robotics runtime/SDK (the
  `cu29` crates). Use when orienting in the repo or writing/reviewing app code:
  tasks/sources/sinks/bridges, the `#[copper_runtime]` macro, `copperconfig.ron`
  graphs, resources, missions, and the runtime's design constraints. For code style
  see the `copper-coding-style` skill; for build/test/logging/replay commands see
  the `copper-workflow` skill.
---

# copper-rs — architecture & task authoring

`copper-rs` is a Rust robotics **runtime + SDK** built around declarative task graphs
in `copperconfig.ron`, compiled into a concrete runtime by the `#[copper_runtime(...)]`
proc macro. Mental model: **"a game engine for robots, where the scene graph is a task
graph and the engine is generated from RON config at compile time."**

Core value props that must be preserved when changing runtime behavior:
**deterministic execution + bit-for-bit replay**, **unified binary logging** (`.copper`),
**sub-microsecond, zero-alloc real-time path**, and a **mixed `std`/`no_std` workspace**
(desktop → SBC → bare-metal MPU, plus browser/wasm).

Companion skills: **`copper-coding-style`** (naming, errors, no_std, logging macros,
tests) and **`copper-workflow`** (the `just` commands, logging/debugging/replay,
environment setup). The repo root `AGENTS.md` is the authoritative, longer companion —
read it for the full file map and debugging playbook. **Authority order when sources
disagree: code > rustdoc/API > wiki/book narrative.** Default branch is `master`.

## Workspace map

- `core/` — runtime + proc-macro crates. The ones that matter most:
  - `core/cu29` — primary user-facing crate; its **prelude re-exports nearly everything**
    app authors touch (`use cu29::prelude::*;`). `lib.rs` is `#![no_std]`-aware.
  - `core/cu29_runtime` — the heart. Key modules: `config.rs` (RON schema + graph),
    `curuntime.rs` (`CuRuntime`, `compute_runtime_plan` — the topological execution plan
    the macro embeds), `cutask.rs` (task traits + `CuMsg`), `cubridge.rs`, `app.rs`
    (`CuApp`/builder), `resource.rs` (HAL wiring), `simulation.rs`/`app_sim.rs` (sim +
    resim), `replay.rs`, `remote_debug.rs` (`remote-debug` feature),
    `monitoring.rs`, `reflect.rs` (`reflect` feature).
  - `core/cu29_derive` — the `#[copper_runtime]` proc macro (`src/lib.rs`); also
    `bundle_resources`, `resources`, `safety_case`, and `gen_cumsgs!`.
  - `core/cu29_export` — log reader CLI (`run_cli`), MCAP/JSON/CSV export, Python API
    (behind `python` feature). `core/cu29_unifiedlog` — the `.copper` slab format.
  - Other core: `cu29_clock` (RobotClock), `cu29_traits`, `cu29_value`, `cu29_log*`
    (interned text logs), `cu29_soa_derive`, `cu29_units` (uom-based), `cu29_intern_strs`.
- `components/` — reusable building blocks, grouped by role:
  `sources/`, `sinks/`, `tasks/`, `bridges/`, `payloads/`, `monitors/`, `codecs/`,
  `res/` (platform HAL bundles), `libs/`, `testing/`.
- `examples/` — **the best executable documentation.** Many features are clearer here
  than in rustdoc. Canonical orientation set: `examples/cu_caterpillar/` (minimal, has
  the CI determinism regression test), plus `cu_flight_controller` and `cu_rp_balancebot`
  which live in the sibling repo `copper-project/extra-examples`.
- `support/` — CI helpers, Docker, embedded crate lists, and `cargo_cunew` (the
  `cargo cunew` bootstrap tool). **Templates live at `support/cargo_cunew/templates/`**
  (`cu_project` = single crate, `cu_full` = multi-crate workspace) — *not* a top-level
  `templates/` dir despite some doc references.
- `vendor/` — patched deps (e.g. `soft_ratatui`, via `[patch.crates-io]`).

`default-members` is only the core, platform-neutral crates. Full-workspace builds pull
in hardware/GUI-heavy crates and need extra system deps (see the `copper-workflow` skill).

## Anatomy of a Copper app

Read an app in this order: `Cargo.toml` (features/deps) → `copperconfig.ron` →
`#[copper_runtime(config=...)]` entrypoint in `main.rs` → task/bridge/resource impls →
`logreader.rs`/`resim.rs` if logs/replay matter → mission bindings if missions exist.

`src/main.rs` — the macro generates the app type, builder, and execution plan:
```rust
use cu29::prelude::*;

#[copper_runtime(config = "copperconfig.ron")]
struct MyApp {}

fn main() {
    let mut app = MyApp::builder()
        .with_log_path("logs/app.copper", Some(SLAB_SIZE))   // builder owns log setup
        .expect("logger")
        .build()
        .expect("app");
    app.run().unwrap();   // run() comes from the CuApp trait
}
```

`copperconfig.ron` — declarative graph (parsed at compile time by `config.rs`):
```ron
(
    tasks: [
        ( id: "src",  type: "tasks::MySource" ),
        ( id: "proc", type: "tasks::MyTask", config: {"pin": 4}, logging: (enabled: true) ),
    ],
    cnx: [ ( src: "src", dst: "proc", msg: "my_payloads::Frame" ) ],
    logging: ( enable_task_logging: true, slab_size_mib: 16, section_size_mib: 8, keyframe_interval: 100 ),
    monitor: ( type: "cu_consolemon::CuConsoleMon" ),
)
```
Notes: `type` is the fully-qualified Rust path to the impl. `enable_task_logging`
defaults `true`; `section_size_mib` must be ≤ `slab_size_mib`; be conservative with sizes
on small-memory targets. Missions (per-node `missions: [...]`) generate mission-specific
builders/modules. Always `just dag` (renders the graph) to sanity-check config edits.

## Task traits (`core/cu29_runtime/src/cutask.rs`)

Three roles, all `Freezable + Reflect`. `Freezable` is the keyframe/replay hook — a task
with internal state must implement `freeze`/`thaw` (bincode); stateless tasks get the
empty default impl. `#[derive(Reflect)]` is required.

```rust
#[derive(Default, Reflect)]
struct MySource { state: bool }
impl Freezable for MySource {
    fn freeze<E: Encoder>(&self, e: &mut E) -> Result<(), EncodeError> { Encode::encode(&self.state, e) }
    fn thaw<D: Decoder>(&mut self, d: &mut D)  -> Result<(), DecodeError> { self.state = Decode::decode(d)?; Ok(()) }
}
impl CuSrcTask for MySource {
    type Resources<'r> = ();
    type Output<'m> = output_msg!(MyPayload);
    fn new(cfg: Option<&ComponentConfig>, res: Self::Resources<'_>) -> CuResult<Self> { Ok(Self::default()) }
    fn process(&mut self, ctx: &CuContext, out: &mut Self::Output<'_>) -> CuResult<()> {
        out.set_payload(MyPayload { .. });
        out.tov = Tov::Time(ctx.now());     // time-of-validity from the RobotClock
        Ok(())
    }
}
```
- `CuSrcTask` — no input, produces `Output` (sensors / data origins).
- `CuTask` — `type Input<'m> = input_msg!(A, B)` (a `CuMsgPack`) → `type Output<'m>`
  (compute/algorithms). Read inputs via `input.payload()` (`Option<&T>`).
- `CuSinkTask` — `Input` only, no output (actuators / final destinations).
- **Bridges** (`CuBridge`, `cubridge.rs`) — the runtime's only bidirectional node.
  A bridge owns one transport (serial port, Zenoh session, CAN bus) and exposes
  typed Rx/Tx channels declared via the `rx_channels!`/`tx_channels!` macros; the
  RON side lists them as first-class nodes distinct from `tasks:`, and cnxs
  address individual channels as `"bridge_id/channel_id"`. Skeleton and taste
  rules live in `copper-component-design`.
- Optional lifecycle hooks beyond `process`: `start`, `stop`, `preprocess`,
  `postprocess`. `new(config, resources)` builds the task; `config` is the node's RON
  `config: {...}` as a `ComponentConfig` (use `.get::<T>("key")`).
- Messages flow as `CuMsg<T>` envelopes (payload + metadata + `tov`). Use the
  `input_msg!`/`output_msg!` macros, not raw `CuMsg` types. Resources (HAL handles) are
  injected via `Resources<'r>` and bound in config — see `examples/cu_resources_test/`.
- Payload trait bounds (`CuMsgPayload`) and the canonical derive live in the
  `copper-coding-style` skill.

## Design biases — uphold these in any change

- **Static over dynamic.** A robot is a static thing. Prefer types, compile-time wiring,
  and proc-macro/codegen enforcement. Avoid runtime string-keyed lookup/service discovery,
  dynamic topology, or "components appear later" patterns when the set is known at compile
  time. Treat generated code as a feature, not something to bypass with runtime glue.
- **No hidden env-var APIs.** Prefer constants. If something is genuinely user-tunable,
  ask whether it belongs in RON config — but don't over-configure; gratuitous config
  fields are a design smell. Don't add a config field without a real user need.
- **Real-time path is sacred.** Never allocate on the RT stack/path; be cautious about
  startup allocation (no_std budgets are tiny). Prefer preallocation, fixed capacity,
  compile-time sizing, zero-copy/borrow over copies. **Any new alloc, copy, serialization
  pass, or latency hit on the RT path is a design-level regression** — don't add it
  silently; require explicit approval; feature-gate offline/debug-only costs out of the
  hot path. Use `just rtsan-smoke` to check for RT violations.
- **`std` vs `no_std` is a real axis.** Don't assume host APIs everywhere. When changing
  shared crates/traits/macros, consider no_std/embedded impact immediately and run
  `just nostd-ci` when practical. Platform priority: Linux > macOS > don't-break-no_std >
  Windows (minimal; Windows CI uses a reduced `core`-only scope).
- **Don't paper over problems** with hacks that hide a deeper design/runtime issue or
  create spaghetti. Use `cargo expand` (`just expand-runtime`) when macro behavior is unclear.

## Where to start for common tasks

- Runtime generation / macro: `core/cu29_derive/src/lib.rs`, `cu29_runtime/src/config.rs`,
  `cu29_runtime/src/curuntime.rs` — and `cargo expand` the output.
- Task API / message flow / lifecycle: `cu29_runtime/src/{cutask,cubridge,app}.rs`.
- Resources / HAL: `cu29_runtime/src/resource.rs`, `examples/cu_resources_test/`.
- Logging / export / replay: `core/cu29_export/`, `core/cu29_unifiedlog/`,
  `examples/cu_caterpillar/src/{logreader,resim}.rs` (see `copper-workflow`).
- Templates / DX: `support/cargo_cunew/templates/{cu_project,cu_full}/`.
- Embedded / no_std: `support/ci/embedded_crates.py`, `components/res/cu_micoairh743/`,
  the `cu_rp2350_skeleton` example, `.github/workflows/reusable-embedded.yml`.
