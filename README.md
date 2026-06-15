# copper-rs skills

Agent skills for working effectively in [**copper-rs**](https://github.com/copper-project/copper-rs)
— the Rust robotics runtime + SDK (the `cu29` crates). They encode the repo's
architecture, house coding style, and the CI-aligned build/test/debug workflow so AI
agents (and humans) can be productive without re-deriving it every time.

## Skills overview

| Skill | When it applies |
|---|---|
| [`copper-rs`](./copper-rs/SKILL.md) | **Architecture & task authoring.** Mental model, workspace map, anatomy of a Copper app, the `CuSrcTask`/`CuTask`/`CuSinkTask` traits, `copperconfig.ron` graphs, resources, missions, and the runtime's design biases (static-over-dynamic, sacred real-time path, std/no_std). Start here when orienting or writing app code. |
| [`copper-rs-coding-style`](./copper-rs-coding-style/SKILL.md) | **House style.** Naming (`Cu` prefix), import grouping, `CuResult`/`CuError` error handling, std/no_std discipline, the interning `debug!/info!/warning!/error!` logging macros (`{}`-only), doctests, recurring clippy gotchas, payload trait bounds, and test layout. |
| [`copper-rs-workflow`](./copper-rs-workflow/SKILL.md) | **Build, test & debug.** The CI-aligned `just` targets (`pr-check`/`lint`/`test`/`std-ci`/`nostd-ci`/`api-check`), environment setup, and the unified-log debugging ladder: extract copperlists → resim (deterministic replay) → remote-debug API → Python export. |

The three are cross-referenced: each `SKILL.md` points to the others by name so the right
one is pulled in for a given task.

## Install

### Via the `skills` installer (recommended)

```
npx skills add copper-project/copper-rs-skills
```

Select the skills you want, choose a scope (global or per-project), and a link method.

### As a Claude Code plugin

This repo is a Claude Code plugin marketplace (`.claude-plugin/marketplace.json`). Add it,
then install the plugin:

```
/plugin marketplace add copper-project/copper-rs-skills
/plugin install copper-rs-skills@copper-rs-skills
```

### Manual

Copy (or symlink) the skill directories into your skills folder:

```bash
# user-global
git clone https://github.com/copper-project/copper-rs-skills
ln -s "$PWD/copper-rs-skills/copper-rs"              ~/.claude/skills/copper-rs
ln -s "$PWD/copper-rs-skills/copper-rs-coding-style" ~/.claude/skills/copper-rs-coding-style
ln -s "$PWD/copper-rs-skills/copper-rs-workflow"     ~/.claude/skills/copper-rs-workflow

# or per-project: symlink into <repo>/.claude/skills/ instead
```

## Recommended workflow

1. Land in the repo → the `copper-rs` skill orients you (architecture, where to start).
2. Writing or reviewing Rust → `copper-rs-coding-style` keeps it on-house-style.
3. Building, testing, or chasing a bug in a recorded run → `copper-rs-workflow` has the
   `just` targets and the unified-log debugging ladder.

Skills auto-activate from their `description`; you can also invoke one explicitly.

## Maintaining these skills

These are derived from the copper-rs codebase and its root `AGENTS.md`. **Authority order
when sources disagree: code > rustdoc/API > wiki/book narrative.** When the runtime,
config schema, `just` targets, or conventions change, update the matching `SKILL.md` here.
Keep each skill's directory name in sync with its `name:` frontmatter. See
[CONTRIBUTING.md](./CONTRIBUTING.md).

## License

Apache-2.0 — see [LICENSE](./LICENSE).
# copper-rs-skills
