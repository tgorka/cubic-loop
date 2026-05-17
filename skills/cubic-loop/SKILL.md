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

If the user passes `--pr` as the first positional arg (`/cubic-loop 47`),
accept that too.

## Preflight

Halt and report if any of these fail. Don't try to recover automatically —
the user should see what's wrong:

1. `gh auth status` succeeds.
2. We're inside a git working tree (`git rev-parse --is-inside-work-tree`).
3. Working tree is **clean** (`git status --porcelain` empty). Cubic-loop
   commits and pushes during the loop; uncommitted local changes would get
   bundled into the first fix commit unintentionally. Ask the user to
   commit, stash, or revert before proceeding.
4. The repo has a `cubic.yaml`. If absent, warn loudly — cubic isn't
   installed and the loop will never see a cubic review.
5. A PR exists for the current branch (or `--pr` was provided):
   ```bash
   gh pr view --json number,headRefName,headRefOid,baseRefName
   ```
   If no PR and the user didn't pass `--pr`, halt and tell them to open
   one first.

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

cubic posts a GitHub check run named `cubic` (case-insensitive match —
some installations use `cubic / review` or similar). Don't post a fresh
trigger comment if cubic is already in flight for this commit:

```bash
CUBIC_STATE=$(
  gh api "repos/$OWNER/$REPO/commits/$HEAD_SHA/check-runs" \
    --jq '[.check_runs[] | select(.name | test("cubic"; "i"))] | last | .status // empty'
)
```

If `$CUBIC_STATE` is empty (no check yet) or `completed`, post a trigger
comment so cubic re-reviews the latest commit:

```bash
if [ "$CUBIC_STATE" != "queued" ] && [ "$CUBIC_STATE" != "in_progress" ]; then
  gh pr comment "$PR" --body "@cubic-dev review"
fi
```

(cubic responds to `@cubic-dev review` per its app docs. If a given
installation uses a different handle, the user can edit this line.)

### C. Wait for cubic's check to complete

Poll the check run, bounded by `--timeout`. cubic can take a few minutes
on large diffs:

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

cubic surfaces its summary in **two places**; check both and prefer the
freshest:

1. The latest PR review whose body starts with `PR score:`:

   ```bash
   REVIEW_JSON=$(
     gh api "repos/$OWNER/$REPO/pulls/$PR/reviews" \
       --jq '[.[] | select((.body // "") | test("^PR score:"; "m"))] | sort_by(.submitted_at) | last'
   )
   ```

2. The PR body (cubic may regenerate the description with `pr_descriptions.generate: true`):

   ```bash
   PR_BODY=$(gh pr view "$PR" --json body -q .body)
   ```

From whichever is newer, extract:

- **Score**: parse `PR score: (\d+)/10`.
- **Stamp**: `REVIEW_JSON.state == "APPROVED"` (cubic auto-approved).
- **Unresolved comment count**: from unresolved review threads (next step).

If both sources lack a score line, treat as "cubic responded but
emitted no score" — surface it and stop. The repo's `cubic.yaml` is
misconfigured (missing or stripped the `PR score: N/10` custom
instruction).

Fetch unresolved cubic threads via GraphQL — see
`references/cubic-api.md` for the full query and the bot-author filter.
Store the count as `UNRESOLVED`.

### E. Check exit conditions

Stop the loop and report **success** if any of these hold:

- `STAMP == APPROVED` **and** `--no-stamp` was not passed.
- `SCORE >= --score` **and** `UNRESOLVED == 0`.

Stop the loop and report **incomplete** if:

- `N == --max-iters` (we hit the ceiling — surface remaining issues).
- iteration timed out in step C.

Otherwise continue to F.

### F. Fix actionable comments

For each unresolved cubic thread:

1. Read the file at the reported path/line; understand the comment in
   context — cubic comments often span the surrounding logic, not just
   the literal line.
2. Decide actionable vs. informational. Cubic explicitly labels some
   comments "nit"/"consider" — those can be resolved without code change
   if the existing code is defensible.
3. If actionable, make the smallest correct fix. Don't expand scope.
4. If a false positive, leave the code alone but still resolve the
   thread (next step) and note the reason in the resolve message.

Run the project's quality gates locally before pushing, if defined in
`AGENTS.md` / `cubic.yaml`. For this repo:

```bash
bun run check
bunx tsc --noEmit
```

**Gate on regressions, not absolute pass.** Before iter 1, capture a
baseline by running each gate against the branch's merge-base with the
target branch. A gate that was already failing on `main` (e.g. a
pre-existing `tsc` error from an empty `src/`) is **not** a blocker for
the loop — only a gate that flips from passing to failing because of a
cubic fix is. Otherwise the loop dead-ends on environmental issues that
no fix to the cubic findings can address.

### G. Resolve addressed threads

Use the GraphQL `resolveReviewThread` mutation per thread. Full recipe in
`references/cubic-api.md`. Batch resolves in a single mutation per ~20
threads to stay under GitHub's rate budget.

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
