# copper-rs skills

Agent skills for working effectively in [**copper-rs**](https://github.com/copper-project/copper-rs)
— the Rust robotics runtime + SDK (the `cu29` crates). They encode the repo's
architecture, house coding style, and the CI-aligned build/test/debug workflow so AI
agents (and humans) can be productive without re-deriving it every time.

## Skills overview

| Skill | When it applies |
|---|---|
| [`copper-arch`](./copper-arch/SKILL.md) | **Architecture & task authoring.** Mental model, workspace map, anatomy of a Copper app, the `CuSrcTask`/`CuTask`/`CuSinkTask` traits, `copperconfig.ron` graphs, resources, missions, and the runtime's design biases (static-over-dynamic, sacred real-time path, std/no_std). Start here when orienting or writing app code. |
| [`copper-coding-style`](./copper-coding-style/SKILL.md) | **House style.** Naming (`Cu` prefix), import grouping, `CuResult`/`CuError` error handling, std/no_std discipline, the interning `debug!/info!/warning!/error!` logging macros (`{}`-only), doctests, recurring clippy gotchas, payload trait bounds, and test layout. |
| [`copper-workflow`](./copper-workflow/SKILL.md) | **Build, test & debug.** The CI-aligned `just` targets (`pr-check`/`lint`/`test`/`std-ci`/`nostd-ci`/`api-check`), environment setup, and a quick unified-log debugging ladder. |
| [`copper-api-flavor`](./copper-api-flavor/SKILL.md) | **API taste rules.** Five non-negotiables for shaping new user-facing traits and adapters (in-place `&mut` outputs, enums via `Deserialize`+`get_value`, no cached inputs, payload/`Tov` placement, `CuTask` lifecycle mirror). |
| [`copper-macro-debug`](./copper-macro-debug/SKILL.md) | **Proc-macro debugging.** What `#[copper_runtime]` emits, common macro errors, when to `cargo expand`. Compile-time only. |
| [`copper-ron-config`](./copper-ron-config/SKILL.md) | **RON schema reference.** Every top-level key, per-node/per-cnx/per-bridge field, `ComponentConfig` accessor rules (`get::<T>` vs `get_value::<T>`), verbatim validation errors, RON foot-guns. |
| [`copper-component-design`](./copper-component-design/SKILL.md) | **Component recipes.** Skeletons and lifecycle patterns for authoring a source / task / sink / bridge, anchored to real crates under `components/`. |
| [`copper-debug-replay`](./copper-debug-replay/SKILL.md) | **Runtime debug/replay deep dive.** `cu29_export::run_cli` subcommands, `sim_mode`+resim flow, the `debug.v1` remote-debug RPC over Zenoh, Python offline iterators, failure modes. |

Skills are cross-referenced: each `SKILL.md` points to the others by name so the right
one is pulled in for a given task.

## Compatibility

Tested with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and
[OpenAI Codex](https://openai.com/index/openai-codex/). Each `SKILL.md` follows
the [agentskills.io](https://agentskills.io/specification.md) spec, so any
skills-aware agent listed by `npx skills add --help` should work.

## Install

> **Note (pre-release):** the `copper-project/copper-rs-skills` repo is currently
> private. Every install path below requires read access on your GitHub account
> — either an SSH key on `github.com`, or a token via `gh auth login`. Once the
> repo is public the `owner/repo` shorthands work with no auth.

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

### Manual

Clone once, then symlink the skill directories into your agent's skills folder.

```bash
git clone git@github.com:copper-project/copper-rs-skills.git
SKILLS=(copper-arch copper-coding-style copper-workflow copper-api-flavor \
        copper-macro-debug copper-ron-config copper-component-design copper-debug-replay)

# Claude Code — user-global
mkdir -p "$HOME/.claude/skills"
for s in "${SKILLS[@]}"; do ln -s "$PWD/copper-rs-skills/$s" "$HOME/.claude/skills/$s"; done

# Codex — per-project (run inside the project you want the skills in)
mkdir -p ./.agents/skills
for s in "${SKILLS[@]}"; do ln -s "$PWD/copper-rs-skills/$s" "./.agents/skills/$s"; done
```

For per-project Claude Code, symlink into `<repo>/.claude/skills/` instead.
Restart open agent sessions after installing so the new skills are discovered.

## Recommended workflow

1. Land in the repo → `copper-arch` orients you (architecture, where to start).
2. **Before you sketch a new user-facing trait, adapter, or config field**, read
   `copper-api-flavor` — the maintainer enforces these five rules on the diff, so it's
   much cheaper to bake them in than to unpick them at review.
3. Writing or reviewing Rust → `copper-coding-style` keeps it on-house-style.
4. Editing `copperconfig.ron` → `copper-ron-config` is the schema reference.
5. Implementing a source / task / sink / bridge → `copper-component-design`.
6. Compile-time macro grief → `copper-macro-debug`.
7. Building, testing, or chasing a bug in a recorded run → `copper-workflow` for the
   `just` targets, `copper-debug-replay` for the deep extract/resim/remote-debug flow.

Skills auto-activate from their `description`; you can also invoke one explicitly.

## Maintaining these skills

These are derived from the copper-rs codebase and its root `AGENTS.md`. **Authority order
when sources disagree: code > rustdoc/API > wiki/book narrative.** When the runtime,
config schema, `just` targets, or conventions change, update the matching `SKILL.md` here.
Keep each skill's directory name in sync with its `name:` frontmatter. See
[CONTRIBUTING.md](./CONTRIBUTING.md).

## License

Apache-2.0 — see [LICENSE](./LICENSE).
