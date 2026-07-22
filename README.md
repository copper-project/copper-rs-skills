# copper-rs skills

Agent skills for working effectively in [**copper-rs**](https://github.com/copper-project/copper-rs)
— the Rust robotics runtime + SDK (the `cu29` crates). They encode the repo's
architecture, house coding style, and the CI-aligned build/test/debug workflow so AI
agents (and humans) can be productive without re-deriving it every time.

> Table of contents:
> **[Compatibility](#compatibility)** · **[Install](#install)** · **[Skills](#skills)** · **[Recommended workflow](#recommended-workflow)** · **[Contributing](#contributing)**

## Compatibility

Tested with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and
[OpenAI Codex](https://openai.com/index/openai-codex/). Each `SKILL.md` follows
the [agentskills.io](https://agentskills.io/specification.md) spec, so any
skills-aware agent listed by `npx skills add --help` should work.

## Install

### Via the `skills` installer (recommended, cross-agent)

Works for both Claude Code and Codex — pick the target(s) from the interactive
menu (`--agent claude-code`, `--agent codex`, or `--agent '*'`).

Public-repo form:

```bash
npx skills add copper-project/copper-rs-skills
```

Private-repo form (clone with SSH, install from the local path):

```bash
git clone git@github.com:copper-project/copper-rs-skills.git
npx skills add "$PWD/copper-rs-skills"
```

The installer prompts for skill selection, scope (`-g` for user-global,
otherwise per-project), and link method (symlink by default, `--copy` to copy).
Update later with `npx skills update`; remove with `npx skills remove`.

### As a Claude Code plugin (Claude only)

This repo is also a Claude Code plugin marketplace
(`.claude-plugin/marketplace.json`).

```
/plugin marketplace add copper-project/copper-rs-skills
/plugin install copper-rs-skills@copper-rs-skills
```

Private-repo variant: clone first, then pass the local path to
`/plugin marketplace add`.

Prefer not to use `npx`? Clone the repo and symlink each `SKILL.md` folder into
your agent's skills directory.

## Skills

The skills are grouped by what you're doing. Each `SKILL.md` cross-references the
others by name so the right one is pulled in for a given task.

### Orientation

[`copper-arch`](./copper-arch/SKILL.md) — the mental model. Generated runtimes,
`copperconfig.ron` task graphs, component roles, `CuMsg` flow, resources,
missions, logging, and deterministic replay. **Start here** when learning how a
Copper application fits together.

[`copper-workflow`](./copper-workflow/SKILL.md) — how the repo is built and
tested. The CI-aligned `just` targets (`pr-check`/`lint`/`test`/`std-ci`/
`nostd-ci`/`api-check`), environment setup, and a quick unified-log debugging
ladder. Use whenever `just` is involved or a run has produced a `.copper` log.

### Writing code

[`copper-core-dev`](./copper-core-dev/SKILL.md) — core runtime/SDK development.
The internal crate map, runtime design constraints, hot-path and std/no_std rules,
deterministic-replay impact, and concrete starting points for changes under `core/`.
Use this when changing Copper itself rather than building an application on it.

[`copper-coding-style`](./copper-coding-style/SKILL.md) — house style. Naming
(`Cu` prefix), import grouping, `CuResult`/`CuError` error handling, std/no_std
discipline, the interning `debug!/info!/warning!/error!` logging macros
(`{}`-only), doctests, recurring clippy gotchas, payload trait bounds, test
layout.

[`copper-api-flavor`](./copper-api-flavor/SKILL.md) — non-negotiables for
shaping new user-facing traits and adapters (in-place `&mut` outputs, enums via
`Deserialize`+`get_value`, no cached inputs, payload/`Tov` placement, no
`Option`/nested-enum branching on the hot path, `CuTask`
lifecycle mirror). **Read this before you sketch a new trait, adapter, or
config field** — the maintainer enforces these on the diff.

[`copper-component-design`](./copper-component-design/SKILL.md) — recipes for
implementing a source / task / sink / bridge. Skeletons and lifecycle patterns
anchored to real crates under `components/`.

### Config & macros

[`copper-ron-config`](./copper-ron-config/SKILL.md) — deep schema reference for
`copperconfig.ron`: every top-level key, per-node/per-cnx/per-bridge field,
`ComponentConfig` accessor rules (`get::<T>` vs `get_value::<T>`), verbatim
validation errors, RON foot-guns.

[`copper-macro-debug`](./copper-macro-debug/SKILL.md) — proc-macro debugging.
What `#[copper_runtime]` emits, common macro errors, when to `cargo expand`.
Compile-time only.

### Debug & replay

[`copper-debug-replay`](./copper-debug-replay/SKILL.md) — the runtime-debug
playbook. `cu29_export::run_cli` subcommands, `sim_mode`+resim flow, the
`debug.v1` remote-debug RPC over Zenoh, Python offline iterators, common
failure modes. Reach for this whenever the reflex is to add `println!` and
rerun.

## Recommended workflow

1. Land in the repo → `copper-arch` explains the execution model.
2. Changing runtime or SDK internals → `copper-core-dev` maps the code and constraints.
3. **Before you sketch a new user-facing trait, adapter, or config field**, read
   `copper-api-flavor` — the maintainer enforces these rules on the diff, so it's
   much cheaper to bake them in than to unpick them at review.
4. Writing or reviewing Rust → `copper-coding-style` keeps it on-house-style.
5. Editing `copperconfig.ron` → `copper-ron-config` is the schema reference.
6. Implementing a source / task / sink / bridge → `copper-component-design`.
7. Compile-time macro grief → `copper-macro-debug`.
8. Building, testing, or chasing a bug in a recorded run → `copper-workflow` for the
   `just` targets, `copper-debug-replay` for the deep extract/resim/remote-debug flow.

Skills auto-activate from their `description`; you can also invoke one explicitly.

## Contributing

See [AGENTS.md](./AGENTS.md) for the skill-file spec and editorial ground rules,
and [CONTRIBUTING.md](./CONTRIBUTING.md) for the change process.

## License

Apache-2.0 — see [LICENSE](./LICENSE).
