## Project structure

Each skill lives in its own directory at the repo root:

- `copper-arch/` — architecture & task authoring
- `copper-coding-style/` — house code style
- `copper-workflow/` — build, test, debug commands
- `copper-api-flavor/` — taste rules for new user-facing traits
- `copper-macro-debug/` — `#[copper_runtime]` compile-time issues
- `copper-ron-config/` — `copperconfig.ron` schema reference
- `copper-component-design/` — source/task/sink/bridge recipes
- `copper-debug-replay/` — runtime debug/replay/remote-debug

Each directory contains a `SKILL.md`. That file is the skill.

`.claude-plugin/` holds the Claude Code plugin/marketplace manifests
(`plugin.json`, `marketplace.json`) — Claude-specific packaging only. Other
agents load the same `SKILL.md` files directly via the [agentskills.io
spec](https://agentskills.io/specification.md).

## Skill file format

Every `SKILL.md` must satisfy the agentskills.io spec:

- YAML frontmatter with a `name:` field that **exactly matches the directory
  name**, and a non-empty `description:` written so an agent can decide from it
  alone whether to activate the skill.
- At least one top-level Markdown heading (`# ...`).

The `description` is the discovery mechanism — lead with what the skill is,
then the concrete triggers ("Use when …").

## Editorial ground rules

**Authority order when sources disagree: code > rustdoc/API > wiki/book
narrative.** Verify claims against source before writing them down; cite
concrete paths (`core/cu29_runtime/src/cutask.rs`) and `just` targets rather
than vague guidance.

**One concern per skill.** If content fits two, put it in the most specific
one and cross-reference by skill name.

**Keep it tight.** Prefer the smallest set of high-signal, repo-specific
facts. Drop generic Rust advice an agent already knows; keep the things that
are surprising *here* (e.g. the `{}`-only logging macros, RON formatted by
`fmtron`, no_std discipline).

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full change process.

## Commit & PR conventions

Conventional commit prefixes (`feat:`, `fix:`, `refactor:`, `docs:`). Bump
`version` in `.claude-plugin/plugin.json` for a notable change. PR
descriptions should say what changed in the copper-rs codebase that prompted
the skill update.
