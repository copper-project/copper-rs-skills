---
name: copper-ron-config
description: >-
  Deep schema reference for `copperconfig.ron`. Use when reading, writing, or
  debugging a copper-rs config file: every top-level key, every per-task/per-cnx/per-bridge
  field, `ComponentConfig` accessor rules (`get::<T>` vs `get_value::<T>`), and the exact
  validation errors the parser emits. All claims cite
  `core/cu29_runtime/src/config.rs`. For the runtime architecture and task-graph mental
  model see `copper-arch`; for `fmtron` formatting and the `just fmt` cycle see
  `copper-workflow`; for the "how should a new config field feel" taste rules
  (enums via `Deserialize`, no stringly parsing) see `copper-api-flavor`.
---

# copper-rs — RON config schema reference

`copperconfig.ron` is a compile-time input to `#[copper_runtime(config = "...")]`. The
parser and graph model live in `core/cu29_runtime/src/config.rs` (~6008 lines). This
skill is the field-by-field reference into that file. It does **not** cover how the
macro consumes the config (see `copper-macro-debug`), how the runtime executes it
(see `copper-arch`), or how to format it (see `copper-workflow`).

Authority: `config.rs` is the source of truth. When the wiki/book disagrees, the code
wins (see `copper-arch` skill's authority order).

## Top-level grammar (`CuConfigRepresentation`, `config.rs:2195`)

| Key | Type | Required | Default | Purpose | Ref |
|---|---|---|---|---|---|
| `tasks` | `Vec<Node>` | no | `[]` | task/source/sink declarations | `config.rs:2197` |
| `cnx` | `Vec<SerializedCnx>` | no | `[]` | connections between nodes (and bridge channels) | `config.rs:2200, 1052` |
| `resources` | `Vec<ResourceBundleConfig>` | no | `[]` | HAL resource bundles bound to tasks/bridges | `config.rs:2198, 988` |
| `bridges` | `Vec<BridgeConfig>` | no | `[]` | external-world bridges (ROS, Zenoh, protocol adapters) | `config.rs:2199, 1002` |
| `logging` | `LoggingConfig` | no | see below | slab/section sizing, keyframes, codecs | `config.rs:2207, 1798` |
| `monitor` / `monitors` | `MonitorConfig` or `Vec<MonitorConfig>` | no | none | monitoring tasks; singular alias accepted | `config.rs:2206, 1768` |
| `runtime` | `RuntimeConfig` | no | none | rate limiting + thread pools | `config.rs:2208, 1853` |
| `missions` | `Vec<MissionsConfig>` | no | single implicit `default` | multi-mission support | `config.rs:2209, 2023` |
| `includes` | `Vec<IncludesConfig>` | no | `[]` | modular composition of sibling RON files | `config.rs:2210, 2029` |

A valid graph needs at least one node. `type:` strings are Rust paths resolved at the
**binary crate root** — not by RON introspection.

## Task node (`Node`, `config.rs:663`)

| Key | Type | Required | Default | Notes | Ref |
|---|---|---|---|---|---|
| `id` | `String` | **yes** | — | unique in the graph | `:664` |
| `type` | `String` | no | — | fully-qualified Rust path of the task struct | `:668` |
| `kind` | `TaskKind` | no | inferred | `source`\|`task`\|`sink` (aliases `src`, `regular`/`cutask`, `snk`). Inference uses graph connectivity | `:673, 617–624, 1540–1604` |
| `config` | `ComponentConfig` | no | `{}` | node-level RON config; see accessors below | `:677` |
| `logging` | `NodeLogging` | no | `enabled=true` | per-node: `enabled`, `codec`, `codecs: {msg_type: codec}`, `handle_content` in `all`/`touched_only`/`none` | `:705, 544–600` |
| `missions` | `Vec<String>` | no | all missions | omit ⇒ task participates in every mission | `:684` |
| `background` | `BackgroundConfig` | no | `false` | `true` for default pool, or `(pool: "name")` for a named pool | `:693, 651` |
| `resources` | `HashMap<String,String>` | no | none | local name → `"bundle_id.field"` binding | `:681` |
| `run_in_sim` | `bool` | no | **false** for sources/sinks, ignored for regular tasks | when `false`, source/sink is stubbed under sim | `:701, 810` |

`kind` inference errors (grep-able strings): `"is declared as kind 'source' but has
incoming connections"` (`:1540`), `"is declared as kind 'task' but has no incoming
connections"` (`:1543`), `"is declared as kind 'sink' but has outgoing or NC
outputs"` (`:1546`), `"has no declared inputs or outputs, so Copper cannot infer"`
(`:1602`).

## Connection (`SerializedCnx`, `config.rs:1052`)

| Key | Type | Required | Notes | Ref |
|---|---|---|---|---|
| `src` | `String` | **yes** | task `id`, or `"bridge_id/rx_channel_id"` | `:1054` |
| `dst` | `String` | **yes** | task `id`, `"bridge_id/tx_channel_id"`, or `"__nc__"` for "output intentionally unused" | `:1055, 1061` |
| `msg` | `String` | **yes** | fully-qualified payload Rust path | `:1056` |
| `missions` | `Vec<String>` | no | omit ⇒ shared across all missions | `:1057, 1143–1166` |

Bridge endpoint parsing lives at `config.rs:1100–1124` (`parse_endpoint`). `Rx` channels
must appear as `src`, `Tx` as `dst`; the reverse is an error
(`:977 "channel '{}' is Tx and cannot act as a source"`, `:981 "channel '{}' is Rx
and cannot act as a destination"`). `__nc__` works only with task destinations, not
bridge channels (`:1182`). Cnx order is preserved from RON for deterministic execution
(`order: usize`, `config.rs:1079`).

## Logging (`LoggingConfig`, `config.rs:1798`)

| Field | Default | Constraint | Ref |
|---|---|---|---|
| `enable_task_logging: bool` | **true** | master switch — off ⇒ empty logs | `:1801` |
| `slab_size_mib: Option<u64>` | unbounded | — | `:1812` |
| `section_size_mib: Option<u64>` | unbounded | must be `≤ slab_size_mib` | `:1816`, err `:3349–3355` |
| `keyframe_interval: Option<u32>` | **100** | interval **in CopperLists**, not wall time | `:1791, 1823` |
| `copperlist_count: Option<usize>` | unbounded | must be `> 0` if set | `:1808`, err `:3341–3346` |
| `codecs: Vec<LoggingCodecSpec>` | `[]` | each `(id, type, config?)`; ids must be unique | `:1827, 1843`, err `:3358–3365` |

`keyframe_interval` counts copperlists — at 1 Hz that's once per 100 s; account for
`runtime.rate_target_hz` when tuning.

## Monitor / Resources / Bridges / Missions / Includes

Monitor (`MonitorConfig`, `config.rs:1768`): `(type: "...", config?: {...})`. Top-level
key is `monitor:` or `monitors:` (both accepted, `:2203`).

Resources (`ResourceBundleConfig`, `config.rs:988`):
`(id, provider, config?, missions?)`. Tasks/bridges bind fields via
`resources: { "local": "bundle_id.field" }`. Fields must match the provider's public
API — validated only at codegen, not at RON parse.

Bridges (`BridgeConfig`, `config.rs:1002`):
`(id, type, config?, resources?, missions?, run_in_sim?, channels: [...])`. Channel
entries are `Rx(id, route?, config?)` or `Tx(id, route?, config?)`
(`BridgeChannelConfigRepresentation`, `:910`). `run_in_sim` defaults **true** for
bridges (`:1026`) — opposite of sources/sinks. Undeclared channel names are rejected
with `"Bridge '{}' does not declare a channel named '{}'"` (`:969`).

Missions (`MissionsConfig`, `config.rs:2023`): `(id: "name")`. Any node/cnx/resource
without a `missions:` list participates in **every** mission. When a node/cnx repeats
across includes, mission sets are **unioned** (`:1143–1166`), not overwritten.

Includes (`IncludesConfig`, `config.rs:2029`):
`(path, params?, missions?)`. `{{key}}` in the included file is textually substituted
from `params` (`:3394–3402`). Paths starting with `/` are absolute; otherwise relative
to the parent config (`:3419–3429`). Included tasks/cnx are appended to the parent
(`:2279–2363`).

## `ComponentConfig` accessors

Every `config: {...}` block deserializes to a `ComponentConfig` map (`config.rs:115`).
Reach for these three in order of increasing complexity:

- **`config.get::<T>("key")? -> Option<T>`** (`:90`) — scalars only (`bool`, integers,
  floats, `String`). Uses `TryFrom<&Value>`. Canonical use: `cu_ratelimit/src/lib.rs:69`.
- **`config.get_value::<T>("key")? -> Option<T>`** (`:115–133`) — anything
  `T: DeserializeOwned`. This is the **only** correct route for enums and nested
  structs. Canonical use: `cu_peer_range_accumulator/src/lib.rs:130`
  (`SampleRetentionConfig`), `cu_peer_triangulation/src/lib.rs:220`.
- **`config.deserialize_into::<T>()`** (`:136–154`) — rare; treat the whole
  ComponentConfig as one flat struct.

**Never write a `match s.as_str() { "reuse_last" => ... }` parser** — that violates the
API-flavor rule (see `copper-api-flavor` skill, rule 2). Enum fields must be real
Rust enums with `#[derive(Deserialize)]` and read via `get_value`.

## Validation errors (grep-able strings)

Every message below is emitted verbatim; grep in `config.rs` to find the site.

- Graph shape: `"Task '{}' is declared as kind 'source' but has incoming connections..."`
  (`:1540`), `"...kind 'sink' but has outgoing or NC outputs..."` (`:1546`),
  `"has no declared inputs or outputs, so Copper cannot infer"` (`:1602`).
- Connection targets: `"Source node not found: {}"` (`:2304`),
  `"Destination node not found: {}"` (`:2310`),
  `"NC destination '{}' does not support bridge channels..."` (`:1182`).
- Bridges: `"Bridge '{}' does not declare a channel named '{}'"` (`:969`),
  `"channel '{}' is Tx and cannot act as a source"` (`:977`),
  `"channel '{}' is Rx and cannot act as a destination"` (`:981`).
- Sizing: `"CopperList count cannot be zero..."` (`:3341`),
  `"Section size...cannot be larger than slab size..."` (`:3349`),
  `"Duplicate logging codec id..."` (`:3358`).
- Runtime block: `"Runtime rate target cannot be zero..."` (`:3376`),
  `"exceeds ... 1_000_000 Hz"` (`:3382`), `"Thread pool id cannot be empty"` (`:1968`),
  `"Thread pool '{}' must have at least 1 thread"` (`:1975`),
  `"real-time priority {} is out of range"` (`:1983`),
  `"niceness {} is out of range"` (`:1991`),
  `"has an empty affinity list"` (`:2002`).

## RON foot-guns

1. **`type:` is resolved at the binary crate root.** A re-export in a sibling
   `lib.rs` may not be visible; the macro reads the RON path as-is.
2. **Enum variants are snake_case.** Serde aliases exist for some (see `TaskKind`
   `:617–624`), but the canonical form is `kind: source`, `background: (pool: "vision")`.
3. **`__nc__` only works for task destinations** — not bridge channels (`:1182`).
4. **`section_size_mib ≤ slab_size_mib` is asymmetric** — reversing is an error
   (`:3349`).
5. **`run_in_sim` defaults differ by node type:** `false` for sources/sinks
   (`:811`), **`true`** for bridges (`:1026`). Regular tasks ignore the field entirely.
6. **`keyframe_interval` counts copperlists, not milliseconds.** With
   `runtime.rate_target_hz: 1`, `keyframe_interval: 100` → one keyframe every 100 s.
7. **Missions merge, they don't override.** A cnx repeated across includes ends up
   in the **union** of both mission lists (`:1143`).
8. **Never hand-format RON.** `just fmt` runs `fmtron` and is enforced in
   `fmt-check` (see `copper-workflow`). Diffs from manual edits will fail CI.
9. **`monitor:` and `monitors:` are aliases** — either one parses, but a mixed file
   with both is confusing to readers. Pick one per repo.
10. **Resource-binding field names are validated at codegen, not at RON parse.** A
    typo in `resources: { pin: "board.gpi04" }` compiles the RON fine and dies later
    in the macro.

## Examples

Minimal (`examples/cu_min_baremetal/copperconfig.ron`):

```ron
(
    tasks: [
        ( id: "src", type: "tasks::FortyKSrc" ),
        ( id: "dst", type: "tasks::FortyKSink" ),
    ],
    cnx: [ ( src: "src", dst: "dst", msg: "crate::tasks::DoraPayload" ) ],
    logging: ( keyframe_interval: 10000, section_size_mib: 1 ),
)
```

Missions + resources + bridge (excerpt from
`examples/cu_resources_test/copperconfig.ron`):

```ron
(
    missions: [ (id: "A"), (id: "B") ],
    resources: [
        ( id: "board_a", provider: "crate::resources::BoardBundle",
          config: { "label": "alpha-bus", "offset": 10 }, missions: ["A"] ),
    ],
    tasks: [
        ( id: "sensor_a", type: "tasks::SensorTask", missions: ["A"],
          resources: { "counter": "board_a.counter", "bus": "board_a.bus" } ),
    ],
    bridges: [
        ( id: "stats_a", type: "bridges::StatsBridge", missions: ["A"],
          resources: { "bus": "board_a.bus" },
          channels: [ Rx(id: "stats_rx"), Tx(id: "stats_tx") ] ),
    ],
    cnx: [
        ( src: "sensor_a", dst: "stats_a/stats_tx",
          msg: "bridges::BusReading", missions: ["A"] ),
        ( src: "stats_a/stats_rx", dst: "inspector_a",
          msg: "bridges::BusReading", missions: ["A"] ),
    ],
)
```
