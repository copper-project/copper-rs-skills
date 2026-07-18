# Contributing to copper-rs skills

These skills capture repository-specific knowledge about
[copper-rs](https://github.com/copper-project/copper-rs). They are only useful if they
stay accurate, so the bar for edits is: **does this match the current code?**

## Ground rules

- **Authority order when sources disagree: code > rustdoc/API > wiki/book narrative.**
  Verify a claim against the source before writing it down; cite concrete paths
  (`core/cu29_runtime/src/cutask.rs`) and `just` targets rather than vague guidance.
- **One concern per skill.** Keep the split clean:
  - `copper-arch` — runtime architecture & execution model
  - `copper-core-dev` — core runtime/SDK development
  - `copper-coding-style` — code style & conventions
  - `copper-workflow` — build, test & debugging commands
  - `copper-api-flavor` — taste rules for new user-facing traits
  - `copper-macro-debug` — `#[copper_runtime]` compile-time issues
  - `copper-ron-config` — RON schema reference
  - `copper-component-design` — source/task/sink/bridge implementation recipes
  - `copper-debug-replay` — runtime debug/replay/remote-debug deep dive

  If something fits two, put it in the most specific one and cross-reference by skill name.
- **Each skill is a directory with a `SKILL.md`.** The YAML frontmatter needs a `name`
  (must equal the directory name) and a `description` written so an agent can decide
  relevance from it alone — lead with what the skill is and the concrete triggers ("Use
  when …").
- **Keep it tight.** Prefer the smallest set of high-signal, repo-specific facts. Drop
  generic Rust advice an agent already knows; keep the things that are surprising *here*
  (e.g. the `{}`-only logging macros, RON formatted by `fmtron`, no_std discipline).

## Making a change

1. Edit the relevant `SKILL.md`. If you move content between skills, fix the
   cross-references in both.
2. Re-read the affected skill end to end — it should make sense standalone.
3. Bump `version` in `.claude-plugin/plugin.json` for a notable change.
4. Open a PR describing what changed in the codebase that prompted the skill update.

## Adding a new skill

Create `copper-<topic>/SKILL.md`, set matching `name`/`description`, link it from the
README's overview table, and from sibling skills where relevant. No change to the plugin
manifests is needed — skills are discovered from the repo root (`"skills": "./"`).
