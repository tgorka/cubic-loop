---
name: cubic-loop
description: >
  Iteratively drives a GitHub PR through cubic.dev review until the PR is
  green — `PR score: N/10` at or above a configurable threshold AND zero
  unresolved cubic comments, OR cubic auto-approves the PR — or until a
  max-iteration ceiling is reached. Triggers cubic review, waits, applies
  fixes for actionable comments, resolves threads, pushes, re-triggers,
  repeats. Use when the user wants to fully optimize a PR against
  cubic.dev's code review standards.
license: MIT
compatibility: >
  Requires git and gh (GitHub CLI) authenticated. Target repository must
  have cubic.dev installed and a `cubic.yaml` whose `reviews.custom_instructions`
  asks cubic to begin every review summary with `PR score: N/10`. cubic-loop
  is GitHub-only — cubic.dev does not support other VCS providers.
metadata:
  author: tgorka
  version: "0.2"
allowed-tools: Bash(gh:*) Bash(git:*)
---

# cubic-loop

Iteratively fix a PR until cubic.dev gives it a clean review: `PR score`
at or above the configured threshold (default `10/10`) AND zero unresolved
cubic threads, **or** cubic posts an approval (its `auto_approve`
stamp). Capped by a max-iteration ceiling so the loop can never run away.

Modelled on `greploop`. cubic.dev specifics — bot login, summary format,
GraphQL recipes — live in `references/cubic-api.md` and are loaded only
when needed.

## Inputs

All optional. Parse from the invocation, fall back to defaults.

| Arg              | Default               | Meaning                                                                                  |
| ---------------- | --------------------- | ---------------------------------------------------------------------------------------- |
| `--pr N`         | current branch's PR   | PR number to drive. Detect from the current branch if omitted.                           |
| `--max-iters N`  | `5`                   | Hard ceiling on review/fix cycles. Never loop forever.                                   |
| `--timeout S`    | `1800` (30 min)       | Per-iteration seconds to wait for cubic's check to complete before giving up the round.  |
| `--score N`      | `10`                  | Minimum `PR score: N/10` required for a clean exit.                                      |
| `--no-stamp`     | off                   | By default a cubic auto-approval (stamp) ends the loop as success. Pass this to disable. |
| `--backend MODE` | `auto`                | Transport for cubic ops. `auto` probes `cubic` (authed) → `gh` (authed). Other values: `cubic`, `gh`, `api`. See [Backend selection](#backend-selection). |

If the user passes `--pr` as the first positional arg (`/cubic-loop 47`),
accept that too.

## Backend selection

cubic-loop talks to cubic via one of three backends:

- **`cubic`** — local headless review using the `cubic` CLI (`cubic review --json`).
  Needs no `cubic.yaml`, no GitHub App install, no PR. Score is derived from
  the issues array (see [`references/cubic-api.md`](references/cubic-api.md)).
- **`gh`** — GitHub CLI subcommands (`gh pr view`, `gh pr comment`,
  `gh pr checks`). Cascades to `gh api` for operations gh CLI does not
  expose natively (review-thread GraphQL, `resolveReviewThread`). Talks to
  the cubic.dev GitHub bot on the repo.
- **`api`** — raw `gh api` REST + GraphQL. Same target as `gh`, lower-level
  transport. Useful when gh's higher-level commands behave unexpectedly.

`--backend auto` (the default) probes in order and picks the first that
works:

```bash
case "${BACKEND:-auto}" in
  auto)
    if command -v cubic >/dev/null && cubic auth list >/dev/null 2>&1; then
      BACKEND=cubic
    elif command -v gh >/dev/null && gh auth status >/dev/null 2>&1; then
      BACKEND=gh
    else
      die "no usable backend: install cubic (https://cubic.dev) or authenticate gh"
    fi
    ;;
  cubic)
    command -v cubic >/dev/null \
      || die "--backend cubic: cubic CLI not found in PATH (install: https://cubic.dev)"
    cubic auth list >/dev/null 2>&1 \
      || die "--backend cubic: cubic CLI unauthenticated (run 'cubic auth login')"
    ;;
  gh|api)
    command -v gh >/dev/null \
      || die "--backend $BACKEND: gh CLI not found in PATH (install: https://cli.github.com)"
    gh auth status >/dev/null 2>&1 \
      || die "--backend $BACKEND: gh CLI unauthenticated (run 'gh auth login')"
    ;;
  *) die "unknown --backend: $BACKEND (valid: auto, cubic, gh, api)" ;;
esac
```

`auto` never selects `api` — `api` and `gh` share the same `gh auth
status` precondition, so auto prefers the higher-level `gh` and you
must pass `--backend api` explicitly to use the raw transport.

If the user passes an explicit backend that is unavailable (`--backend
cubic` but `cubic` is not in PATH or unauthed; `--backend gh`/`api`
but `gh auth status` fails), halt with an install/auth hint. Don't
silently fall through — the user asked for that transport on purpose.

Per-operation transport (recipes in
[`references/cubic-api.md`](references/cubic-api.md)):

| Operation                       | `cubic`                          | `gh`                                                    | `api`                          |
|---------------------------------|----------------------------------|---------------------------------------------------------|--------------------------------|
| Preflight `cubic.yaml` check    | notice if missing                | hard-fail if missing                                    | hard-fail if missing           |
| Trigger review                  | `cubic review --json` (local)    | `gh pr comment "$PR" --body "@cubic-dev review"`        | `gh api ... -X POST` issue comment |
| Poll check run                  | synthetic (`completed` on exit)  | `gh pr checks "$PR" --json name,status,conclusion`      | `gh api .../check-runs`        |
| Fetch verdict                   | parse `issues` from CLI JSON     | `gh pr view --json body,reviews`                        | `gh api .../pulls/.../reviews` |
| List unresolved threads         | issues array (local)             | cascade → `gh api graphql` (no native subcmd)           | `gh api graphql`               |
| Resolve threads                 | no-op (deferred — see Notes)     | cascade → `gh api graphql` mutation                     | `gh api graphql` mutation      |
| Detect stamp (auto-approval)    | N/A (CLI never stamps)           | `gh pr view --json reviews`                             | `gh api .../pulls/.../reviews` |

Loop semantics (`--score`, `--max-iters`, `--timeout`, exit conditions)
are identical across backends — the backend only abstracts how
`SCORE`/`STAMP`/`UNRESOLVED` are obtained.

## Preflight

Halt and report if any of these fail. Don't try to recover automatically —
the user should see what's wrong:

1. `gh auth status` succeeds.
2. We're inside a git working tree (`git rev-parse --is-inside-work-tree`).
3. Working tree is **clean** (`git status --porcelain` empty). Cubic-loop
   commits and pushes during the loop; uncommitted local changes would get
   bundled into the first fix commit unintentionally. Ask the user to
   commit, stash, or revert before proceeding.
4. The repo has a `cubic.yaml`. Behavior depends on the active backend
   (see [Backend selection](#backend-selection)):
   - `cubic` backend: absent `cubic.yaml` is a **notice**, not a halt.
     `cubic review --json` is local and does not need the GitHub App or
     a repo-level config. The notice still mentions that the cubic.dev
     bot won't see the PR.
   - `gh` or `api` backend: hard-fail if absent. The loop talks to the
     cubic.dev GitHub bot via PR comments; without `cubic.yaml` the bot
     will never review, so the loop would time out on iteration 1.
5. A PR exists for the current branch (or `--pr` was provided):
   ```bash
   gh pr view --json number,headRefName,headRefOid,baseRefName
   ```
   If no PR and the user didn't pass `--pr`, halt and tell them to open
   one first.
6. **Baseline-gate check.** Stash any in-progress work, check out the
   merge-base with the PR's base branch (first `git fetch origin <base>`,
   then `git merge-base HEAD origin/<base>`),
   and run each project gate (e.g. `bun run check`, `bunx tsc --noEmit`).
   Return to the PR HEAD. Any gate that already fails on the merge-base
   is **environmental**, not caused by this PR — halt with a clear
   message asking the user to fix baseline first. Cubic-loop refuses to
   silently mask such failures, and a strict in-loop gate (step F) would
   otherwise dead-end on them. If all gates pass on the baseline, record
   that they're expected to pass on every iteration and continue.

Resolve `OWNER` and `REPO` once from `gh repo view --json owner,name`.

## Loop

Repeat for at most `--max-iters` rounds. Track the current iteration
number `N`, starting at `1`.

### A. Push any pending local commits

Cubic only sees what GitHub sees. If the local branch is ahead of the
remote, push it before triggering review:

```bash
git push
```

Let real failures surface (auth, non-fast-forward, hook rejection) —
the loop should halt rather than re-trigger cubic against a stale
remote SHA. `git push` returns 0 when there's nothing to push, so
"everything up-to-date" is already not an error.

Brief settle so GitHub registers the push:

```bash
sleep 5
```

Re-read the head SHA — pushing may have updated it:

```bash
HEAD_SHA=$(gh pr view "$PR" --json headRefOid -q .headRefOid)
```

### B. Trigger cubic (only if not already running)

Dispatch by `$BACKEND` — full recipes in
[`references/cubic-api.md` → Trigger review](references/cubic-api.md#trigger-review).

- **`cubic`**: invoke `cubic review --json [-b origin/<base>] [-c
  $HEAD_SHA]` directly. The CLI runs to completion synchronously, so
  there is nothing to "skip if already running" — the next loop
  iteration just runs the CLI again.
- **`gh`**: skip trigger if the latest `cubic` check run is `queued` or
  `in_progress`; otherwise `gh pr comment "$PR" --body "@cubic-dev
  review"`.
- **`api`**: same logic as `gh`, but use `gh api
  "repos/$OWNER/$REPO/commits/$HEAD_SHA/check-runs"` to read state and
  the REST issue-comments endpoint to post.

For `gh`/`api`, the recipe to read current state:

```bash
CUBIC_STATE=$(
  gh api "repos/$OWNER/$REPO/commits/$HEAD_SHA/check-runs" \
    --jq '[.check_runs[] | select(.name | test("cubic"; "i"))] | last | .status // empty'
)
if [ "$CUBIC_STATE" != "queued" ] && [ "$CUBIC_STATE" != "in_progress" ]; then
  gh pr comment "$PR" --body "@cubic-dev review"
fi
```

(cubic responds to `@cubic-dev review` per its app docs. If a given
installation uses a different handle, the user can edit this line.)

### C. Wait for cubic's check to complete

Dispatch by `$BACKEND` — recipes in
[`references/cubic-api.md` → Poll check run](references/cubic-api.md#poll-check-run).

- **`cubic`**: no polling. `cubic review --json` from step B blocks
  until the review finishes; treat its exit as `completed`. The
  `--timeout` flag still applies — wrap the CLI call in `timeout
  $TIMEOUT cubic review --json ...` and treat a timeout exit as the
  iteration timing out.
- **`gh`**: `gh pr checks "$PR" --json name,status,conclusion` and
  filter for the cubic check.
- **`api`**: `gh api "repos/$OWNER/$REPO/commits/$HEAD_SHA/check-runs"`
  filtered for cubic check name.

Poll bounded by `--timeout` (skip for `cubic` backend per above). cubic
can take a few minutes on large diffs:

```bash
DEADLINE=$(( $(date +%s) + TIMEOUT ))
while [ "$(date +%s)" -lt "$DEADLINE" ]; do
  STATUS=$(
    gh api "repos/$OWNER/$REPO/commits/$HEAD_SHA/check-runs" \
      --jq '[.check_runs[] | select(.name | test("cubic"; "i"))] | last | .status // "missing"'
  )
  case "$STATUS" in
    completed) break ;;
    missing)   echo "Waiting for cubic check to appear..." ;;
    *)         echo "cubic: $STATUS" ;;
  esac
  sleep 10
done
```

If the loop exited because of the deadline (no `break`), treat this
iteration as **timed out**. Report it and stop the outer loop — don't
silently keep iterating against a stuck check.

### D. Fetch cubic's verdict

Goal: produce normalized `SCORE`, `STAMP`, and `UNRESOLVED` values that
the rest of the loop consumes uniformly. Dispatch by `$BACKEND` —
recipes in
[`references/cubic-api.md` → Fetch verdict](references/cubic-api.md#fetch-verdict).

- **`cubic`**: parse the JSON emitted by `cubic review` in step B.
  Observed schema (v1.3.1): `{ "issues": [...], "error"?: string }`.
  - `UNRESOLVED = len(.issues)`.
  - `SCORE = max(0, 10 - UNRESOLVED)` (deterministic local mapping —
    cubic CLI does not emit a `PR score: N/10` line; that line is
    bot-side, driven by `reviews.custom_instructions` in `cubic.yaml`).
  - `STAMP = ""` (empty — the CLI never auto-approves a PR; see
    `references/cubic-api.md` for the gh/api STAMP shape: empty when no
    stamp, review ID string when stamped).
  - If `.error` is non-empty, surface it and stop the loop.
- **`gh`**: `gh pr view "$PR" --json body,reviews -q ...` to extract
  the latest cubic review body and PR body; parse `PR score: (\d+)/10`
  from whichever is newer. Detect `STAMP` from reviews where
  `state == "APPROVED"` and the author is the cubic bot.
- **`api`**: same logic, but read reviews and PR body via `gh api`:

  ```bash
  REVIEW_JSON=$(
    gh api "repos/$OWNER/$REPO/pulls/$PR/reviews" \
      --jq '[.[] | select((.body // "") | test("^PR score:"; "m"))] | sort_by(.submitted_at) | last'
  )
  PR_BODY=$(gh pr view "$PR" --json body -q .body)
  ```

For `gh`/`api`, from whichever source is newer, extract:

- **Score**: parse `PR score: (\d+)/10`.
- **Stamp**: `REVIEW_JSON.state == "APPROVED"` (cubic auto-approved).
- **Unresolved comment count**: see step F.

If both sources lack a score line (gh/api only), treat as "cubic
responded but emitted no score" — surface it and stop. The repo's
`cubic.yaml` is misconfigured (missing or stripped the `PR score: N/10`
custom instruction).

Fetch unresolved cubic threads — recipe per backend in
[`references/cubic-api.md` → List unresolved threads](references/cubic-api.md#list-unresolved-threads).
For `gh`/`api` this is GraphQL via `gh api graphql`; for `cubic` it is
the issues array already in hand. Store the count as `UNRESOLVED`.

### E. Check exit conditions

Stop the loop and report **success** if any of these hold:

- `STAMP` is non-empty (a stamping APPROVED review id was found) **and** `--no-stamp` was not passed.
- `SCORE >= --score` **and** `UNRESOLVED == 0`.

Stop the loop and report **incomplete** if:

- `N == --max-iters` (we hit the ceiling — surface remaining issues).
- iteration timed out in step C.

Otherwise continue to F.

### F. Fix actionable comments

For each unresolved cubic finding (a "thread" on `gh`/`api` backends,
an entry of the `issues` array on the `cubic` backend):

1. Read the file at the reported path/line; understand the comment in
   context — cubic comments often span the surrounding logic, not just
   the literal line.
2. Decide actionable vs. informational. Cubic explicitly labels some
   comments "nit"/"consider" — those can be resolved without code change
   if the existing code is defensible.
3. If actionable, make the smallest correct fix. Don't expand scope.
4. If a false positive, leave the code alone. For `gh`/`api` backends,
   still resolve the thread (next step) and note the reason in the
   resolve message. For the `cubic` backend there is no remote thread
   to resolve — the finding will be re-emitted next iteration until
   spec 2 lands local resolve persistence.

Run the project's quality gates locally before pushing, if defined in
`AGENTS.md` / `cubic.yaml`. For this repo:

```bash
bun run check
bunx tsc --noEmit
```

**Gates are strict during the loop — every gate must exit 0 before
push.** If a fix breaks a gate, iterate on the fix; don't push broken
code at cubic and let it re-flag it.

The "what if a gate is failing on `main`?" case is handled by the
baseline preflight (see Preflight § 6), not by relaxing the in-loop
gate. That way the loop never silently masks regressions, but it also
can't dead-end on an environmental failure that no cubic fix could ever
address.

### G. Resolve addressed threads

Dispatch by `$BACKEND` — recipes in
[`references/cubic-api.md` → Resolve threads](references/cubic-api.md#resolve-threads).

- **`cubic`**: no-op for this iteration. Local-review findings have no
  remote resolution state. A follow-up spec (`--no-resolve` +
  save/restore thread IDs) will define local persistence so subsequent
  iterations can skip already-handled findings.
- **`gh`** / **`api`**: GraphQL `resolveReviewThread` mutation per
  thread. Batch ~20 per mutation to stay under GitHub's rate budget. gh
  CLI has no native wrapper, so both backends use `gh api graphql`
  here — the cascade is the same.

If `cubic.yaml` sets `resolve_threads_when_addressed: true` (as this repo
does), cubic will also auto-resolve once it sees the fix on the next
review pass — but resolving explicitly is faster and avoids a re-review
just to clean up bookkeeping.

### H. Commit and push

```bash
git add -A
git commit -m "address cubic review (cubic-loop iteration $N)"
git push
sleep 5
```

If `git diff --cached --quiet` shows nothing to commit (we resolved
threads but made no code changes), skip the commit and push, and go back
to **B** — cubic should re-review when prodded even without a new
commit.

Increment `N` and loop back to **A**.

## Report

After exiting, print a single status block:

```
cubic-loop complete.
  Backend:     <cubic | gh | api>
  PR:          #<N> (<branch>)
  Iterations:  <N>
  Final score: <X>/10
  Stamp:       <APPROVED | no>
  Resolved:    <count> comments
  Remaining:   <count>
```

If the loop exited early (timeout or max iters), list every remaining
unresolved comment with `path:line — first line of the comment` so the
user can scan them at a glance:

```
cubic-loop stopped after 5 iterations.
  Backend:     gh
  PR:          #47 (feat/foo)
  Final score: 8/10
  Stamp:       no
  Resolved:    9 comments
  Remaining:   2

Remaining issues:
  - src/auth.ts:45 — Consider rate limiting this endpoint
  - src/db.ts:112 — Missing index on user_id column
```

## Notes

- **Why a per-iteration timeout instead of a global one?** Cubic's review
  time depends on diff size; budgeting per-round is a better fit than
  one big global budget that can be eaten by a single slow round.
- **Why default `--score 10`?** Mirrors greploop's `5/5` ethos: keep
  pushing until the bot says ship-it. Lower with `--score 9` (or `8`)
  for pragmatism on noisy reviews.
- **Why default exit on stamp?** This repo's `cubic.yaml` sets
  `auto_approve_behavior: live` + `auto_approve: low_risk_only` —
  cubic only stamps PRs it's already confident in. Treating that as a
  green exit matches user intent for the common case. `--no-stamp` is
  there for repos that don't trust the stamp or want to require an
  explicit score.
- **Why no GitLab/Perforce branches?** cubic.dev is GitHub-only at the
  time of writing. If that changes upstream, the loop generalizes
  cleanly — VCS-specific commands are concentrated in steps A–B and G–H.
