---
name: copper-workflow
description: >-
  Build, test, and debugging workflow for the copper-rs (`cu29`) repo. Use when running
  CI-aligned `just` targets (pr-check/lint/test/std-ci/nostd-ci/api-check), setting up the
  environment to make those pass, rendering the DAG, expanding the runtime macro, or
  debugging a running/recorded app via the unified `.copper` log: extracting copperlists,
  resim (deterministic replay), the remote-debug API, or the Python export. For
  architecture see `copper-arch`; for code style see `copper-coding-style`.
---

# copper-rs — build, test & debugging workflow

Prefer `just` targets over ad-hoc cargo invocations — they mirror
`.github/workflows/general.yml`. Run `just fmt` after every edit pass. Before pushing, run
the narrowest fitting target; default to `just pr-check`. Default branch is `master`.

## Build & verification

| Command | Purpose |
|---|---|
| `just` / `just pr-check` | full local PR gate: `fmt` → `lint` → `api-check` → `test` |
| `just fmt` | format Rust (`cargo fmt`), TOML (`taplo`), **RON (`fmtron`)**, + prek hygiene hooks |
| `just lint` | `fmt-check` + `typos` + `clippy-std` + `clippy-nostd` (clippy is `--deny warnings`) |
| `just test` | `nextest` std (excludes embedded) + `nextest --no-default-features` |
| `just std-ci [mode=debug\|release\|cuda-release]` | mirror the std CI path (incl. caterpillar determinism test + template generation) |
| `just nostd-ci` | mirror embedded/no_std CI (clippy/build no-default-features + rp2350 thumbv8m + embedded crates) |
| `just api-check` / `just api-update` | check / refresh the V1 public-API snapshot (`cargo-public-api`, nightly) |
| `just dag [mission=]` | render the working-dir `copperconfig.ron` graph |
| `just expand-runtime pkg=<crate> bin=<bin> [features=]` | `cargo expand` the generated runtime |
| `just rtsan-smoke pkg= bin= [args=]` | run an app under the realtime sanitizer |
| `just docs` | build wiki + rustdoc locally |

Tooling: builds need **Rust 1.95+** (edition 2024). `just` targets self-check for required
tools and print the `cargo install` line if missing (`taplo-cli`, `fmtron`, `prek`,
`cargo-nextest`, `cargo-llvm-cov`, `cargo-public-api@0.51.0`, `cargo-expand`). The public-API
check needs the **nightly** toolchain. Per-directory `justfile`s under `examples/`,
`components/`, etc. `import "../../justfile"` and add app-specific targets (`cl`, `resim`,
`mcap`, ...). The workspace is large/hardware-heavy: **package-focused verification
(`-p <crate>`) is often more pragmatic than a full-workspace run.**

### Environment setup (required for `pr-check`/`std-ci` to pass)
Without these, the checks fail on *environmental* (not code) errors — clippy-std/test pull
GUI/GStreamer/pcap crates and python-task tests:
- apt (mirrors `support/docker/Dockerfile.ubuntu`): `libudev-dev libpcap-dev libglib2.0-dev
  libssl-dev clang libclang-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
  libgstreamer-plugins-bad1.0-dev libwayland-dev libxkbcommon-dev`
- python: `pip3 install --break-system-packages cbor2` (for `cu-python-task` process-mode tests).

## Logging & debugging — the unified log is the primary tool

**Do not add ad-hoc `println!`/custom text dumps to inspect message values.** Copper already
records task messages as CopperLists when task logging is on — extract them. For temporary
instrumentation use the structured macros `debug!`/`info!`/`warning!`/`error!` (interned text
logs; see the `copper-coding-style` skill for their `{}`-only syntax), never bespoke
text formats.

Debugging escalation ladder (cheapest first):
1. **Extract recorded data** — the app's logreader binary (built on `cu29_export::run_cli`):
   `extract-copperlists` (message payloads, `--export-format json|csv`),
   `extract-text-log <string-index>` (structured logs; index is usually
   `target/debug/cu29_log_index`), `fsck`, `export-mcap`. See
   `examples/cu_caterpillar/src/logreader.rs` + its `justfile` (`just cl`, `just logreader`,
   `just mcap`). The logreader uses `gen_cumsgs!("copperconfig.ron")` to generate the
   decode type for that config.
2. **Resim (deterministic replay)** — replay a recorded log back through the app. It's just
   the normal macro in sim mode: `#[copper_runtime(config=..., sim_mode = true)]`. Keyframes
   (via `freeze`/`thaw`) let replay resume from frozen task state. See
   `examples/cu_caterpillar/src/resim.rs` + `just resim`. CLI contract for replay binaries:
   `--log-base` (recorded log), `--replay-log-base` (output), `--debug-base` (switches to
   replay-backed remote-debug server). Prefer `cu29::replay::ReplayCli`, or `ReplayArgs` with
   `#[command(flatten)]` when the binary has extra CLI (missions etc.).
   **CLI args only — do not add parallel env-var fallbacks for the launch contract.**
3. **Remote debug API** (`core/cu29_runtime/src/remote_debug.rs`, feature `remote-debug`) —
   arbitrary state inspection across the timeline: `nav.seek/step/replay`, `timeline.get_cl`,
   `state.inspect/read/search`, `schema.get_type/get_payload_map`. See
   `examples/cu_remote_debug_session/`.
4. **Python** (`cu29_export`, `python` feature) for programmatic offline analysis:
   `struct_log_iterator_unified/bare` — preferred over scraping rebuilt text.

If message data is missing from a log, check config first: global
`logging: (enable_task_logging: true)` and node-level `logging: (enabled: true)`.

## Other gotchas

- **RON files are formatted with `fmtron`** (run via `just fmt`); a `.bak` is produced and
  auto-deleted. Don't hand-format RON. TOML is formatted with `taplo`. There is also a RON
  format check (`support/ci/check_ron_format.sh`) in `fmt-check`.
- The `.copper` log is written as slabs: a base `app.copper` plus `app_0.copper`,
  `app_1.copper`... Tools take the *base* path and find the slabs.
- `build.rs` in apps exports `LOG_INDEX_DIR` (`OUT_DIR`) so the interned-string index can
  be reconstructed. Many examples write logs under their own `logs/` dir.
- Feature flags on `cu29` worth knowing: `std`, `signal-handler`, `textlogs`, `units`
  (default set), plus `reflect`, `remote-debug`, `safety-ids`, `parallel-rt` (requires std),
  `rtsan`, `async-cl-io`, `high-precision-limiter`, `sysclock-perf`, `defmt` (no_std logging).
- `cargo deny` advisory ignores live in `.config/deny.toml`; typo allowlist in
  `.config/_typos.toml`. Update those rather than disabling the checks.
- Docs sync: when a change in `core/` alters documented behavior, the maintainer expects
  follow-up PRs to the sibling `../copper-rs.wiki` and `../copper-rs-book` checkouts.
