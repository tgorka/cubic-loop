# AGENTS.md

The contract for AI coding agents (Claude Code, Codex, etc.) and human contributors working on this repo.

## What this project is

`cubic-loop` is a Claude Code skill that iteratively drives a pull request through [cubic.dev](https://cubic.dev) review until the PR is green (configurable score threshold and zero unresolved comments) or a max-iteration ceiling is reached. The skill source lives under `skills/cubic-loop/`.

## Repo layout

- `skills/cubic-loop/` — the distributable Claude Code skill (SKILL.md + any reference files). Installed under `~/.claude/skills/` by consumers.
- `src/` — TypeScript helpers (currently empty). Reserved for parsing/orchestration utilities the skill may shell out to.
- `.changeset/` — pending Changeset entries; aggregated into release notes.
- `.github/` — CI, release, dependabot, PR + issue templates.
- `cubic.yaml` — this repo's own cubic.dev review config. The repo dogfoods cubic-loop on itself.
- `biome.json`, `tsconfig.json`, `bunfig.toml`, `package.json` — runtime + tooling config.

## Quality gates

Every change must satisfy:

- `bun run check` exits 0 (Biome lint + `bun test`).
- `bunx tsc --noEmit` exits 0.
- Tests added or updated for any behavior change in `src/`.
- Changeset entry added via `bun run changeset` for user-visible changes (skip for docs-only or chore PRs).

## Code conventions

- Files: `kebab-case.ts`. Tests colocated, named `<source>.test.ts`.
- TypeScript: `camelCase` for functions/variables, `PascalCase` for types (no `I` prefix), `SCREAMING_SNAKE_CASE` for constants.
- No `console.log` in runtime code (Biome enforces `noConsole` as an error). Use a logger module or stderr for one-off CLI status.
- No `any`. Strict mode is on; respect `noUncheckedIndexedAccess`.
- Async = `async/await`. Prefer Bun-native APIs (`Bun.file`, `Bun.write`, `Bun.YAML.parse`, `Bun.spawn`).
- Biome 2.4 only — no ESLint/Prettier.

## Skill conventions

- The skill body in `skills/cubic-loop/SKILL.md` follows the [Claude Code skill format](https://docs.claude.com/claude-code/skills): YAML frontmatter (`name`, `description`, optional `allowed-tools`, etc.) followed by markdown instructions.
- Reference docs the skill loads on demand live under `skills/cubic-loop/references/`.
- Treat the skill as user-facing: every prose change is potentially a user-visible change and warrants a Changeset entry.

## Runtime scope discipline

These rules cover what *running code and dispatched agents may write to on disk during execution* — they do not restrict normal source edits to files in this repo. Editing `src/`, `skills/`, configs, and docs is expected; follow `CONTRIBUTING.md`.

- Runtime writes stay inside the project root (this repo's working tree). Don't touch paths outside it unless the user explicitly asks.
- Inside the repo, runtime/generated state belongs under standard build dirs (`dist/`, `coverage/`, `node_modules/`). Avoid scattering generated files elsewhere.
- Atomic writes (tmp + rename) when modifying anything under shared state directories.

## Test patterns

- Colocate tests next to source (`<source>.test.ts`). No `tests/` dir inside `src/`.
- Filesystem-touching tests use `mkdtemp(path.join(os.tmpdir(), "cubic-loop-<concern>-"))` and clean up in `afterEach`.
- Unique test ID prefix per concern (e.g., `LOOP_*`, `PARSE_*`).

## When in doubt

Read the current `skills/cubic-loop/SKILL.md` and the latest entries under `.changeset/` to find the in-flight work. If a request conflicts with an existing artifact, surface the conflict to the user rather than silently re-deciding.
