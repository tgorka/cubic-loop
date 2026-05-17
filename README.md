# cubic-loop

A Claude Code skill that iteratively drives a PR through [cubic.dev](https://cubic.dev) review — trigger review, apply fixes for actionable comments, re-trigger, repeat — until the PR is green (configurable score threshold, zero unresolved comments) or a maximum-iteration ceiling is reached.

Inspired by [`greploop`](https://github.com/greptileai/greptile) for Greptile and modelled on the [`tgorka/bmad-pr`](https://github.com/tgorka/bmad-pr) project scaffold.

## Status

Early scaffold. Project structure (Bun + TypeScript + Biome + Changesets + GitHub workflows + cubic.yaml) is in place; the skill body under `skills/cubic-loop/SKILL.md` is intentionally a placeholder pending design.

## Requirements

- [Bun](https://bun.sh) >= 1.3
- [Claude Code](https://docs.claude.com/claude-code) (the skill is consumed there)
- `gh` (GitHub CLI), authenticated
- A repository with cubic.dev installed and a `cubic.yaml` checked in

macOS or Linux (Windows via WSL).

## Quick Start

```bash
bun install --frozen-lockfile
bun run check   # Biome lint + tests (release-blocker gate)
```

## Scripts

| Script              | What it does                                              |
| ------------------- | --------------------------------------------------------- |
| `bun run check`     | Biome lint (CI mode) + `bun test`. The CI gate.           |
| `bun run lint`      | Biome lint without tests.                                 |
| `bun run format`    | Biome auto-format (writes changes).                       |
| `bun run test`      | Run the test suite (passes when no tests are present).    |
| `bun run test:watch`| Run the suite in watch mode.                              |
| `bun run changeset` | Create a Changeset entry describing a user-visible change.|

## Repository Layout

- `skills/cubic-loop/` — the Claude Code skill source (SKILL.md + any references). Distributed as part of this repo and installed under `~/.claude/skills/` (or via a plugin).
- `src/` — TypeScript helpers (empty for now; reserved for parsing/orchestration utilities the skill may shell out to).
- `cubic.yaml` — this repo's own cubic.dev review config. cubic-loop dogfoods itself.
- `.changeset/` — pending Changeset entries for the next release.
- `.github/` — CI, release, dependabot, PR + issue templates.

## Style Checker

Biome 2.4 enforces formatting (2-space indent, double quotes, semicolons) and lint rules (`noConsole`, `noUnusedVariables`, `noImplicitAnyLet`, `useExhaustiveDependencies`). Build outputs are excluded — see `biome.json`.

## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for the development setup, PR flow, and code-style conventions. The [`AGENTS.md`](./AGENTS.md) file is the contract for AI coding agents working in this repo.

## License

MIT — see [`LICENSE`](./LICENSE).
