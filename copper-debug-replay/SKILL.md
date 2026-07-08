---
name: copper-debug-replay
description: >-
  Runtime-debug playbook for copper-rs. Use whenever the reflex is to add
  `println!`, rebuild, and rerun — copper already recorded every message; the
  right move is extract-copperlists, resim (deterministic replay), or the
  remote-debug protocol. Covers the full CLI surface of `cu29_export::run_cli`,
  the `#[copper_runtime(sim_mode = true)]` replay flow, the `debug.v1` RPC over
  Zenoh, the Python offline API, and the common failure modes (empty logs,
  slab discovery, divergence, `CuStampedDataSet` mismatch, keyframe interval).
  For build/format/`just` targets see `copper-workflow`; for compile-time
  macro errors see `copper-macro-debug`; for the RON side of a debug config
  see `copper-ron-config`.
---

# copper-rs — runtime debug & replay

The mental model: **a copper app already logs every message and every keyframe**.
If you're about to `println!` a value to see what's happening, stop — extract the
recorded copperlists, replay them deterministically, or attach the remote-debug
protocol to a running or replay-backed app. This skill is the deep reference for
those three tools. For the quick escalation ladder see `copper-workflow`; here
we cover the exact CLI surfaces, code entry points, and failure modes.

Sources: `core/cu29_export/src/lib.rs`, `core/cu29_unifiedlog/src/{lib,memmap}.rs`,
`core/cu29_runtime/src/{replay,app_sim,simulation,remote_debug}.rs`,
`examples/cu_caterpillar/src/{logreader,resim,determinism}.rs`.

**Line-number citations below are anchors, not addresses** — they drift as these files
evolve. The durable hooks are the *symbol names* (`ReplayArgs`, `SimOverride`,
`run_cli`, `struct_log_iterator_unified`, `MAIN_MAGIC`) and the *quoted error/protocol
strings* (`"log family not found"`, `debug.v1`, `MissingSession`). Grep those if a
`:NNN` misses.

## Escalation ladder (deep dive)

Cheapest → richest. Reach for the first that answers the question.

1. **Extract from the recorded log** — payload snapshots (JSON/CSV/MCAP), text logs,
   integrity checks. No rebuild, no rerun.
2. **Resim** — full deterministic replay of the recorded log through the app
   (`sim_mode = true`). Use when you need "what would this task have done given
   this exact input?" and to catch non-determinism.
3. **Remote-debug (`debug.v1` RPC)** — arbitrary state inspection and cursor
   navigation, either on a live app or a replay-backed session. Feature-gated
   (`remote-debug`).
4. **Python offline** (`cu29_export`, `python` feature) — programmatic analysis
   over the whole log with `struct_log_iterator_unified` /
   `copperlist_iterator_unified`.

## The logreader binary

Each app builds its own logreader binary. Template:
`examples/cu_caterpillar/src/logreader.rs`.

```rust
use cu29_export::run_cli;
use cu29::gen_cumsgs;

gen_cumsgs!("copperconfig.ron");        // compile-time: generates cumsgs::CuStampedDataSet

fn main() {
    run_cli::<cumsgs::CuStampedDataSet>().expect("logreader CLI failed");
}
```

The `gen_cumsgs!` macro emits the app-specific decode type from the same RON the app
was recorded with. **Both must match** — see the "wrong `CuStampedDataSet`" failure mode.

### `run_cli` subcommands (`cu29_export::run_cli`, `core/cu29_export/src/lib.rs:120–225`)

Invoked as `logreader <unifiedlog_base> <subcommand> [flags]`:

| Subcommand | Flags | What it does | Ref |
|---|---|---|---|
| `extract-copperlists` | `--export-format {json,csv}` | dump every recorded copperlist as JSON or CSV | `:125, 246, 527` |
| `extract-text-log <log_index>` | positional path (usually `target/debug/cu29_log_index`) | reconstruct interned text logs | `:123` |
| `fsck` | `--verbose`, `--dump-runtime-lifecycle` | verify log integrity; optional lifecycle event dump | `:130` |
| `log-stats` | `--output`, `--config`, `--mission` | stats bundle for DAG rendering | `:138` |
| `export-mcap` | `--output`, `--progress`, `--quiet` | convert to MCAP (`mcap` feature) | `:151` |
| `mcap-info` | `--schemas`, `--sample-messages` | inspect MCAP metadata (`mcap` feature) | `:164` |

Canonical justfile recipes (`examples/cu_caterpillar/justfile`): `just cl` (extract
copperlists), `just logreader` (text log), `just fsck`, `just mcap`.

### Unified log slab format

`core/cu29_unifiedlog/src/lib.rs`:

- `MAIN_MAGIC = [0xB4, 0xA5, 0x50, 0xFF]` ("BRASS OFF"), `SECTION_MAGIC = [0xFA, 0x57]`,
  `UNIFIED_LOG_FORMAT_VERSION = 1` (`:39–47`).
- Section header carries a `UnifiedLogType` (`CopperList`, `StructuredLogLine`,
  `RuntimeLifecycle`, `Keyframe`) (`:76–84`).
- Files are slabs: `app.copper` base + `app_0.copper`, `app_1.copper`, ... The
  builder discovers them from the **base** path (`memmap.rs:121–125`). Always pass
  the base — never `_0` — to any tool.

## Resim (deterministic replay)

Second binary in the app, next to `main.rs`:

```rust
#[copper_runtime(config = "copperconfig.ron", sim_mode = true)]
struct MyAppReSim {}
```

`sim_mode = true` opens a callback surface where each `SimStep` can be intercepted:
`SimOverride::ExecutedBySim` (inject a recorded output), `ExecuteByRuntime` (let the
task run), or `Errored`. Canonical example:
`examples/cu_caterpillar/src/resim.rs:22–146`.

### CLI contract (`ReplayArgs`, `core/cu29_runtime/src/replay.rs:55–68`)

```rust
#[arg(long)] pub log_base:        Option<PathBuf>,   // recorded log base
#[arg(long)] pub replay_log_base: Option<PathBuf>,   // replay output base (template supported)
#[arg(long)] pub debug_base:      Option<String>,   // if set, become a remote-debug server
```

**CLI args only — no env-var fallback.** Defaults are only applied when the flag is
absent (`replay.rs:119–127`). Do not add parallel `COPPER_LOG_BASE`-style env vars.

Prefer `cu29::replay::ReplayCli` directly, or embed `ReplayArgs` with
`#[command(flatten)]` when the binary has its own extra flags (missions, etc.).

Helpers in `replay.rs`:

- `ensure_log_family_exists(log_base)` — verifies the slab set exists (`:134–149`).
- `first_slab_path(log_base)` — resolves `_0.copper` from a base (`:193–206`).
- `per_session_replay_log_base(template, labels)` — unique output paths using
  `NEXT_REPLAY_SESSION_ID` and sanitized labels (`:208–236`).
- `remove_log_family(log_base)` — deletes base + `_0`, `_1`, ... (`:151–191`).

### Keyframes and `Freezable`

Keyframes snapshot every task's `Freezable::freeze` output at
`logging.keyframe_interval` copperlists (see `copper-ron-config`). Replay resumes from a
keyframe by calling `thaw` on every task. If a stateful task doesn't implement `freeze`
properly, replay from any non-zero copperlist will diverge — this is why every
stateful task in `copper-component-design` implements it.

Sim-only helpers on the runtime (`core/cu29_runtime/src/app_sim.rs:1–27`):

- `set_forced_keyframe_timestamp(ts)` — pin the mock clock to a captured ts.
- `lock_keyframe(kf)` — restore task state from a specific keyframe.
- `captures_keyframe(culistid)` — check if the current step captures a keyframe.

### Callback pattern (from `examples/cu_caterpillar/src/resim.rs`)

```rust
let mut sim_callback = move |step: default::SimStep| -> SimOverride {
    use default::SimStep::*;
    match step {
        Src(Process(_, output)) => {
            *output = msgs.get_src_output().clone();   // inject recorded output verbatim
            SimOverride::ExecutedBySim
        }
        _ => SimOverride::ExecuteByRuntime,             // let the task actually run
    }
};
copper_app.run_one_iteration(&mut sim_callback)?;
```

### Determinism regression harness

`cu_caterpillar` ships the `determinism_record_and_resim` test (feature
`determinism_ci`). CI runs it single-threaded with `COPPER_DETERMINISM_ITERS=N` to
replay N times and byte-compare the resulting log against the original. If you
change execution/replay behavior, expect to keep it green. See
`examples/cu_caterpillar/src/determinism.rs` and `just determinism` in the crate's
justfile.

## Remote-debug (`debug.v1`)

Protocol: `core/cu29_runtime/src/remote_debug.rs`. Feature: `remote-debug`.
Transport: **Zenoh** with these defaults (`:274–277`):

- Local Unix-socket endpoint derived from `paths.base`.
- Multicast/gossip scouting disabled.
- Shared memory enabled, threshold 1 byte, pool 4 MiB.

### Topic layout (base `debug_base`, `:22–31`)

- Request: `{base}/rpc/request`
- Response: `{base}/rpc/reply/{client_id}`
- Events: `{base}/events/{cursor,run,watch,health}`

### Envelope (`:50–70`)

Request: `api="debug.v1"`, `request_id`, `session_id?`, `method`, `params`,
`reply_to`. Response: `request_id` (echoed), `ok`, `result?`, `error?`, `cursor_rev`,
`resolved_at?`.

### Method surface (`:113–182`)

| Namespace | Methods (contract) |
|---|---|
| `session.*` | `open` (no session), `close`, `capabilities`, `cancel(op_id)` |
| `nav.*` | `seek(target, resolve?)`, `step(delta)`, `run_until(target, max_steps?, timeout_ms?)`, `replay(at?, limit?)` |
| `timeline.*` | `get_cursor()`, `get_cl(at?, include_payloads?)`, `list(from, to, page?)` |
| `logs.*` | `strings()`, `list(source?, offset?, limit?)` |
| `schema.*` | `get_stack()`, `list_types(filter?)`, `get_type(type_path, format?)`, `get_outputs()` |
| `state.*` | `inspect(path?, at?, depth?)`, `read(path?, at?, format="json")`, `search(query, at?)`, `watch.open(path, at?, mode?)`, `watch.close(watch_id)` |
| `health.*` | `ping()` (session optional), `stats()` |

Cursor `cursor_rev` (`:73–82`) is a monotonic counter: `nav.seek/step/replay` and
state-mutating calls bump it; queries preserve it via a temporary restore. Session
states: `NoSession → OpenCursorUnset → OpenCursorSet → ServerStopping`.

Error codes (`:210–224`): `InvalidApi`, `UnknownMethod`, `MissingSession`,
`SessionNotFound`, `SessionLimitReached`, `InvalidParams`, `ResolveFailed`,
`SeekFailed`, `StepFailed`, `ReplayFailed`, `RunUntilFailed`, `GetClFailed`,
`TimelineListFailed`, `CursorFailed`, `StateFailed`, `PathNotFound`, `TypeNotFound`.

### Serving

`cu29::replay::serve_remote_debug(debug_base, log_base, app_factory, build_callback,
time_of)` (`replay.rs:238–269`, `#[cfg(feature = "remote-debug")]`). Runs a
replay-backed Zenoh server on the topic base. The `App` type must implement
`CuSimApplication + ReflectTaskIntrospection + CurrentRuntimeCopperList<P>`.

Session limits (`remote_debug.rs:89–94`): 15-minute idle timeout, 64 max active
sessions, stale eviction on new requests / poll timeouts.

Example client + server: `examples/cu_remote_debug_session/`.

## Python offline analysis

Feature: `cu29_export` with `python`. Module name: `libcu29_export`
(`core/cu29_export/src/lib.rs:899`).

Iterators (`:738–841`):

- `struct_log_iterator_bare(path) -> (iter, strings)` — standalone log.
- `struct_log_iterator_unified(path) -> (iter, strings)` — unified log.
- `copperlist_iterator_unified(path)` — requires
  `register_copperlist_python_type::<P>()` first.
- `runtime_lifecycle_iterator_unified(path)` — mission start/stop, panic, shutdown.

Typed entrypoints (`:66–92`):

- `copperlist_iterator_unified_typed_py::<P>(path, py)` — seeds effective config from
  the log then registers the payload type.
- `runtime_lifecycle_iterator_unified_py(path, py)`.

`PyCuLogEntry` accessors (`:844–896`): `.ts()`, `.msg_index()`, `.culistid()`,
`.component_id()`, `.task_index()`, `.paramname_indexes()`, `.params()`.
`copperlist_to_py` (`:924–951`) yields a namespace with `id`, `state`, `messages[]`
each carrying `task_id`, `tov`, `metadata`, `payload`.

## Common failure modes

| Symptom | Root cause | Fix |
|---|---|---|
| `extract-copperlists` returns empty | `logging.enable_task_logging: false` (or node-level `logging: (enabled: false)`) | flip it on in RON — see `copper-ron-config` |
| logreader "log family not found" | passed `foo_0.copper` instead of base `foo.copper`, or `_N` slab missing | pass the **base** path; `ls` the dir to confirm the family |
| replay diverges from record | non-deterministic source (real `Instant::now`, live clock), config drift, missing `freeze/thaw` on a stateful task | use `RobotClock::mock` in both runs, keep RON identical, verify `Freezable` covers all state; see `copper-component-design` |
| logreader panics decoding payloads | `gen_cumsgs!` config path doesn't match the app's recorded config | point the macro at the exact same RON; if missions differ, pass `mission =` |
| replay skips or re-executes wrong tasks | `SimStep` match arms miss a task or default to `ExecuteByRuntime` when they should inject | audit the callback against `default::SimStep` variants |
| keyframe cost dominates | `keyframe_interval` too small for the rate | raise it; remember it counts copperlists, not ms (`copper-ron-config`) |
| remote-debug client "MissingSession" | forgot `session.open` before any nav/state call | open first; `health.ping` is the only session-optional method |
| "MissionStarted event found but mission mismatch" | replaying an app-recorded log under the wrong mission | pass the correct `--mission` (or `assert_unified_log_mission(...)` in the test) |

## Decision matrix

| Question | Tool |
|---|---|
| "What was the value of task X's output at cl N?" | `extract-copperlists --export-format json`, or `remote-debug timeline.get_cl` for live |
| "Did task X even run?" | `fsck --dump-runtime-lifecycle`, or `extract-text-log` grep |
| "Why does task X compute Y here?" | resim + inject inputs, step through with `remote-debug nav.step` |
| "Is this non-determinism?" | resim + `COPPER_DETERMINISM_ITERS=N`, byte-compare logs |
| "How does state Z evolve over the run?" | `remote-debug state.watch` on the path |
| "Bulk analysis / statistics" | Python `copperlist_iterator_unified_typed_py` |
| "Perf regression breakdown" | `export-mcap` → external viewer, or `remote-debug health.stats` |

## Never do (in a debug context)

- **Add `println!` / `eprintln!` for temporary instrumentation.** Use the interned
  `debug!/info!/warning!/error!` macros (see `copper-coding-style`) so the output
  ends up in the same unified log the extractor already reads.
- **Add env-var fallbacks to the replay CLI.** `replay.rs:119–127` explicitly
  applies defaults only when the flag is absent.
- **Point the logreader at `foo_0.copper`.** Always the base.
- **Assume `sim_mode = true` alone gives determinism** — it enables the callback
  surface. Determinism also requires `RobotClock::mock`, stable RON, and
  `Freezable` coverage on every stateful task.
