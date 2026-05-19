# Changelog

## 0.2.0

### Minor Changes

- e3b4774: Land the real `cubic-loop` skill body. Replaces the placeholder under
  `skills/cubic-loop/SKILL.md` with an iterative loop that drives a GitHub
  PR through cubic.dev review until it's green (default: `PR score: 10/10`
  and zero unresolved cubic threads) or until cubic auto-approves the PR
  (default exit; disable with `--no-stamp`), bounded by `--max-iters`
  (default 5) and a per-iteration `--timeout` (default 30 min). Cubic-specific
  API recipes (bot detection, GraphQL thread resolution, stamp detection)
  live in `skills/cubic-loop/references/cubic-api.md`. README's Status
  section now reflects the landed skill and documents invocation.
- 1bf6127: Ship cubic-loop as a Claude Code plugin. Adds `.claude-plugin/plugin.json`
  (plugin name `cubic`, version `0.1.0`, skills path `./skills/`) and
  `.claude-plugin/marketplace.json` (single-plugin marketplace named
  `cubic-loop`) modelled on `tgorka/bmad-stepper`. Users can now install
  via:

  ```
  /plugin marketplace add tgorka/cubic-loop
  /plugin install cubic@cubic-loop
  ```

  The skill becomes available as `cubic:cubic-loop`. Hand-copying
  `skills/cubic-loop/` to `~/.claude/skills/` still works as a fallback.
  `cubic.yaml` excludes `.claude-plugin/**` from auto-stamp so plugin
  metadata changes always get human review. README documents the install
  flow and updates the invocation examples to the namespaced form.

This file is auto-managed by [Changesets](https://github.com/changesets/changesets). Add a Changeset entry via `bun run changeset` for any user-visible change; entries are aggregated into release notes when a "Version Packages" PR merges.

## Unreleased
