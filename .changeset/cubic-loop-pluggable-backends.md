---
"cubic-loop": minor
---

Add `--backend auto|cubic|gh|api` flag to cubic-loop and a per-operation
transport layer. `auto` (default) probes `cubic` (authed) → `gh`
(authed) and picks the first usable backend. The `cubic` backend uses
`cubic review --json` for local headless review and downgrades the
`cubic.yaml` preflight from a hard-fail to a notice, since local review
needs no GitHub App or repo-level config. The `gh` backend uses `gh
pr <subcmd>` where one exists and transparently cascades to `gh api`
for review-thread GraphQL. The `api` backend is the existing raw `gh
api` path. Loop semantics (`--score`, `--max-iters`, `--timeout`, exit
conditions) are unchanged across backends. The final report block now
includes a `Backend:` line. Recipes per backend per operation live in
`skills/cubic-loop/references/cubic-api.md`.
