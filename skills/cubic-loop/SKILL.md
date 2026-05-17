---
name: cubic-loop
description: >
  Iteratively drives a GitHub PR through cubic.dev review until the PR is green
  (configurable score threshold and zero unresolved comments) or a maximum
  iteration ceiling is reached. Triggers cubic review, applies fixes for
  actionable comments, pushes, re-triggers, repeats. Use when the user wants
  to fully optimize a PR against cubic.dev's code review standards.
license: MIT
compatibility: Requires git and gh (GitHub CLI) authenticated. Target repository must have cubic.dev installed and a cubic.yaml that asks reviews to begin with `PR score: N/10`.
metadata:
  author: tgorka
  version: "0.1"
allowed-tools: Bash(gh:*) Bash(git:*)
---

# cubic-loop

> **Status:** scaffold. Skill body is intentionally a placeholder —
> design and instructions land in a follow-up change.

Iteratively fix a PR until cubic.dev gives a clean review: PR score at or
above the configured threshold (default 10/10) and zero unresolved cubic
comments, capped by a max-iteration ceiling.

Refer to the project README for context. The detailed loop instructions
(trigger / fetch / exit / fix / resolve / push) will be specified here
once the design discussion is closed out.
