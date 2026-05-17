---
"cubic-loop": minor
---

Land the real `cubic-loop` skill body. Replaces the placeholder under
`skills/cubic-loop/SKILL.md` with an iterative loop that drives a GitHub
PR through cubic.dev review until it's green (default: `PR score: 10/10`
and zero unresolved cubic threads) or until cubic auto-approves the PR
(default exit; disable with `--no-stamp`), bounded by `--max-iters`
(default 5) and a per-iteration `--timeout` (default 30 min). Cubic-specific
API recipes (bot detection, GraphQL thread resolution, stamp detection)
live in `skills/cubic-loop/references/cubic-api.md`. README's Status
section now reflects the landed skill and documents invocation.
