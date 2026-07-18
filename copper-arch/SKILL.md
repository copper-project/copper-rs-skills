---
name: copper-arch
description: >-
  Architecture guide for the copper-rs robotics runtime/SDK (the `cu29` crates).
  Use when learning the execution model or reading and designing Copper applications:
  compile-time task graphs, generated runtimes, component roles, `CuMsg` data flow,
  resources, missions, unified logging, and deterministic replay. For implementing a
  source/task/sink/bridge see `copper-component-design`; for changing runtime internals
  see `copper-core-dev`; for the full RON schema see `copper-ron-config`.
---

# copper-rs architecture

`copper-rs` is a Rust robotics **runtime + SDK** built around declarative task graphs
in `copperconfig.ron`. The `#[copper_runtime(...)]` proc macro turns that static graph
into a concrete application runtime at compile time.

Mental model: **a game engine for robots, where the scene graph is a task graph and the
engine is generated from RON config.** Components are the systems, `CuMsg<T>` values are
the data flowing between them, and a CopperList is the recorded state of one execution
cycle.

The architecture is built to preserve deterministic execution and bit-for-bit replay,
unified binary logging (`.copper`), a zero-allocation real-time path, and deployment from
desktop hosts down to `no_std` embedded targets. When changing the implementation behind
those guarantees, use `copper-core-dev`.

## System model

1. **Describe the graph.** `copperconfig.ron` declares tasks, bridges, typed
   connections, resources, logging, monitoring, and optional missions.
2. **Generate the runtime.** `#[copper_runtime(config = "...")]` parses and validates
   that graph at compile time, then generates the app type, builder, message layout, and
   execution plan.
3. **Build the app.** The generated builder wires component instances to resources and
   configures logging.
4. **Run the plan.** The runtime calls components in topological order. Messages stay in
   runtime-owned CopperList slots; tasks read input handles and write into output handles.
5. **Record and replay.** Enabled task messages and interned text logs share the unified
   `.copper` log. Keyframes capture component state through `Freezable`, allowing resim
   to resume deterministically.

The main implementation anchors are:

- `core/cu29_runtime/src/config.rs` — RON schema and graph validation.
- `core/cu29_runtime/src/curuntime.rs` — `CuRuntime` and
  `compute_runtime_plan`.
- `core/cu29_runtime/src/cutask.rs` — task traits, `CuMsg`, and `Freezable`.
- `core/cu29_runtime/src/cubridge.rs` — bidirectional bridge model.
- `core/cu29_runtime/src/app.rs` — `CuApplication` lifecycle traits.
- `core/cu29_derive/src/lib.rs` — `#[copper_runtime]` code generation.
- `core/cu29_unifiedlog/` — `.copper` slab storage.

## Anatomy of a Copper app

Read an app in this order: `Cargo.toml` (features and dependencies) →
`copperconfig.ron` → the `#[copper_runtime]` entrypoint → component and resource
implementations → `logreader.rs`/`resim.rs` if logs or replay matter → mission bindings
if missions exist.

The macro generates the app type, builder, and execution plan:

```rust
use cu29::prelude::*;

#[copper_runtime(config = "copperconfig.ron")]
struct MyApp {}

fn main() {
    let mut app = MyApp::builder()
        .with_log_path("logs/app.copper", Some(SLAB_SIZE))
        .expect("logger")
        .build()
        .expect("app");
    app.run().unwrap();
}
```

The matching graph is declarative and typed:

```ron
(
    tasks: [
        (id: "src", type: "tasks::MySource"),
        (id: "proc", type: "tasks::MyTask", config: {"pin": 4}),
    ],
    cnx: [(src: "src", dst: "proc", msg: "my_payloads::Frame")],
    logging: (enable_task_logging: true),
)
```

`type` is the fully qualified Rust path to the component implementation. Connections
define both graph edges and payload types. Use `copper-ron-config` for the complete schema
and validation rules, and run `just dag` after graph edits.

## Component roles and message flow

All three task roles implement `Freezable + Reflect` and operate on runtime-owned
messages:

- `CuSrcTask` has no input and writes sensor/origin data to `Output`.
- `CuTask` reads `Input` (one message or a `CuMsgPack`) and writes `Output` for compute,
  filtering, or fusion.
- `CuSinkTask` reads `Input` and has no output; it usually drives an actuator or other
  terminal endpoint.
- `CuBridge` owns one bidirectional transport and exposes typed Rx/Tx channels. Config
  addresses a channel as `"bridge_id/channel_id"`.

Messages are `CuMsg<T>` envelopes containing an optional payload, metadata, and
time-of-validity (`tov`). `input_msg!` and `output_msg!` describe the handles a component
receives; components do not allocate or return new message envelopes on each cycle.
Resources such as device handles are injected through `Resources<'r>` and bound from the
graph rather than discovered dynamically.

Use `copper-component-design` for implementation skeletons, lifecycle hooks, config
access, resource declarations, and `Freezable` recipes. Use `copper-api-flavor` before
designing a new task-facing trait or adapter.

## Missions, resources, and replay

- **Missions** select declared subsets or variants of a graph. Per-node `missions: [...]`
  entries generate mission-specific modules and builders without introducing runtime
  service discovery.
- **Resources** keep platform/HAL ownership outside component logic and inject typed
  handles during construction. `examples/cu_resources_test/` is the canonical model.
- **Replay** combines recorded CopperLists with keyframed component state. A stateful
  component must freeze every field that can affect later outputs; otherwise resim can
  diverge after a keyframe.

For extracting logs and running resim, use `copper-debug-replay`. For build and graph
rendering commands, use `copper-workflow`.

## Architectural constraints to keep in mind

- The topology is static and compile-time-generated; do not assume tasks or connections
  appear dynamically.
- The runtime owns message storage; components borrow inputs and mutate output slots.
- The real-time path is allocation-free and sensitive to copies, serialization, and
  blocking I/O.
- Shared runtime surfaces may need to work under both `std` and `no_std`.
- Determinism includes component state, message ordering, timestamps, and replay behavior.

These are the mental-model constraints an app or component author needs. The actionable
rules and code map for modifying the runtime live in `copper-core-dev`.
