# cubic-api reference

Recipes for the three cubic-loop backends, organized per operation. Kept
out of `SKILL.md` so the main flow stays scannable. See
[Backend selection](../SKILL.md#backend-selection) for when each
backend is used.

Conventions:

- `$OWNER`, `$REPO`, `$PR`, `$HEAD_SHA` resolved as in `SKILL.md`
  preflight / step A.
- `$BACKEND` ∈ `{cubic, gh, api}`, set by the Backend-selection block
  in `SKILL.md`.
- The `gh` backend prefers `gh pr <subcmd>` when one exists and
  cascades to `gh api` for GraphQL-only operations (review threads,
  `resolveReviewThread`). That cascade is internal to the `gh` backend
  — it is **not** a runtime switch.

## cubic CLI: JSON schema (v1.3.1)

The `cubic` backend uses `cubic review --json [-b <base>] [-c
<commit>] [-p <prompt>]`. The CLI delegates to the configured AI
provider (see `cubic auth`) and emits a single JSON object on stdout
when complete:

```json
{
  "issues": [
    {
      "path": "src/foo.ts",
      "line": 42,
      "severity": "warning",
      "body": "Consider …"
    }
  ],
  "error": "Internal error"
}
```

Fields used by cubic-loop:

- `.issues` — array of findings. `cubic-loop` maps:
  - `UNRESOLVED = .issues | length`
  - `SCORE = max(0, 10 - UNRESOLVED)` — deterministic, because the CLI
    does not emit a `PR score: N/10` line (that line is produced by
    the cubic.dev GitHub bot driven by
    `reviews.custom_instructions` in `cubic.yaml`).
  - `STAMP = false` (CLI never auto-approves).
- `.error` — set when the underlying provider failed. Surface and stop
  the loop; do not continue against partial results.

Detect schema drift cheaply by validating the top-level keys:

```bash
echo "$CUBIC_JSON" | jq -e 'has("issues")' >/dev/null \
  || die "unexpected cubic review JSON: $CUBIC_JSON"
```

If `cubic` ships a future version with a real `score` field, prefer it
over the derived value.

## Identifying the cubic bot (gh / api backends)

cubic posts under a GitHub App bot account whose login matches
`^cubic.*\[bot\]$` (commonly `cubic-dev-ai[bot]`). Detect dynamically
so the skill keeps working if cubic renames its bot:

```bash
CUBIC_BOT=$(
  gh api "repos/$OWNER/$REPO/pulls/$PR/reviews" \
    --jq '[.[] | select(.user.type == "Bot") | .user.login | select(test("^cubic"; "i"))] | last // empty'
)
```

If `$CUBIC_BOT` is empty, cubic hasn't reviewed this PR yet — wait
another poll cycle before retrying.

## Trigger review

- **`cubic`**:

  ```bash
  CUBIC_JSON=$(timeout "$TIMEOUT" cubic review --json \
    ${BASE:+-b "$BASE"} ${HEAD_SHA:+-c "$HEAD_SHA"})
  CUBIC_RC=$?
  if [ "$CUBIC_RC" -eq 124 ]; then
    # 124 = GNU coreutils timeout signal — treat as iteration timeout.
    ITERATION_TIMED_OUT=1
  elif [ "$CUBIC_RC" -ne 0 ]; then
    die "cubic review exited $CUBIC_RC: $CUBIC_JSON"
  fi
  ```

  Synchronous — there is no "already running" concern. Check
  `CUBIC_RC` before parsing `CUBIC_JSON` — a `timeout`-killed CLI can
  leave the JSON truncated, which would crash later `jq` calls in
  step D.

- **`gh`**:

  ```bash
  CUBIC_STATE=$(gh pr checks "$PR" --json name,status \
    --jq '[.[] | select(.name | test("^cubic"; "i"))] | last | .status // empty')
  if [ "$CUBIC_STATE" != "queued" ] && [ "$CUBIC_STATE" != "in_progress" ]; then
    gh pr comment "$PR" --body "@cubic-dev review"
  fi
  ```

- **`api`**:

  ```bash
  CUBIC_STATE=$(gh api "repos/$OWNER/$REPO/commits/$HEAD_SHA/check-runs" \
    --jq '[.check_runs[] | select(.name | test("^cubic"; "i"))] | last | .status // empty')
  if [ "$CUBIC_STATE" != "queued" ] && [ "$CUBIC_STATE" != "in_progress" ]; then
    gh api -X POST "repos/$OWNER/$REPO/issues/$PR/comments" \
      -f body="@cubic-dev review"
  fi
  ```

## Poll check run

- **`cubic`**: not applicable. The CLI call from "Trigger review"
  blocks until completion; treat its exit as `completed`. The
  `--timeout` flag still applies, wrapped around the CLI call.

- **`gh`**:

  ```bash
  STATUS=$(gh pr checks "$PR" --json name,status \
    --jq '[.[] | select(.name | test("^cubic"; "i"))] | last | .status // "missing"')
  ```

- **`api`**:

  ```bash
  STATUS=$(gh api "repos/$OWNER/$REPO/commits/$HEAD_SHA/check-runs" \
    --jq '[.check_runs[] | select(.name | test("^cubic"; "i"))] | last | .status // "missing"')
  ```

Useful fields when `STATUS == "completed"`: `conclusion`
(`success`/`failure`/`neutral`/`cancelled`), `details_url`. Both gh
and api recipes return `"missing"` when no cubic check exists yet —
the outer poll loop treats that as "keep waiting".

## Fetch verdict

Goal: produce normalized `SCORE`, `STAMP`, `UNRESOLVED` for the rest
of the loop.

- **`cubic`**: parse the JSON captured in "Trigger review". Validate
  shape and surface errors before deriving any other value:

  ```bash
  echo "$CUBIC_JSON" | jq -e 'has("issues")' >/dev/null \
    || die "unexpected cubic review JSON: $CUBIC_JSON"
  ERR=$(echo "$CUBIC_JSON" | jq -r '.error // ""')
  [ -n "$ERR" ] && die "cubic review error: $ERR"
  UNRESOLVED=$(echo "$CUBIC_JSON" | jq '.issues | length')
  SCORE=$(( 10 - UNRESOLVED ))
  [ "$SCORE" -lt 0 ] && SCORE=0
  STAMP=""
  ```

  Note: `--score N` against the `cubic` backend means "exit cleanly
  when ≤ `10 - N` findings remain", not "minimum quality grade". The
  CLI does not weight severity; each finding subtracts 1 from a 10
  ceiling. Combined with `--score 0` and `UNRESOLVED > 10`, the
  derived score would clamp to 0 and falsely satisfy the threshold
  — guard by also requiring `UNRESOLVED == 0` for clean cubic-backend
  exits (or use `--score 10`, which the default already enforces).

- **`gh`**:

  ```bash
  PR_JSON=$(gh pr view "$PR" --json body,reviews)
  REVIEW_BODY=$(echo "$PR_JSON" | jq -r \
    '[.reviews[] | select(.body // "" | test("^PR score:"; "m"))]
     | sort_by(.submittedAt) | last | .body // ""')
  PR_BODY=$(echo "$PR_JSON" | jq -r '.body // ""')
  ```

- **`api`**:

  ```bash
  REVIEW_BODY=$(gh api "repos/$OWNER/$REPO/pulls/$PR/reviews" \
    --jq '[.[] | select((.body // "") | test("^PR score:"; "m"))]
          | sort_by(.submitted_at) | last | .body // ""')
  PR_BODY=$(gh pr view "$PR" --json body -q .body)
  ```

For `gh`/`api`, parse the score from whichever source is newer:

```bash
SCORE=$(printf '%s' "$REVIEW_BODY $PR_BODY" \
  | grep -Eo '^PR score: [0-9]+/10' | head -1 \
  | grep -Eo '[0-9]+' | head -1)
```

If `SCORE` is empty, the bot responded with no `PR score:` line —
halt with "cubic.yaml missing the score custom instruction".

## List unresolved threads

- **`cubic`**: already in hand — the issues array from "Trigger
  review" *is* the thread list. Each entry's `path`+`line`+`body`
  matches the structure of a remote thread; there are no thread IDs
  because there is no remote state.

- **`gh`** / **`api`**: GraphQL via `gh api graphql` (gh CLI has no
  native review-thread subcommand, so both backends share this
  cascade):

  ```bash
  gh api graphql -f query='
  query($owner: String!, $repo: String!, $pr: Int!, $cursor: String) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $pr) {
        reviewThreads(first: 100, after: $cursor) {
          pageInfo { hasNextPage endCursor }
          nodes {
            id
            isResolved
            isOutdated
            path
            line
            comments(first: 1) {
              nodes { body author { login } }
            }
          }
        }
      }
    }
  }' -F owner="$OWNER" -F repo="$REPO" -F pr="$PR"
  ```

  Filter to threads where `isResolved == false`, `isOutdated == false`,
  and `comments.nodes[0].author.login` matches `^cubic.*\[bot\]?$`.
  Paginate via `pageInfo.endCursor` while `hasNextPage` is true.

## Resolve threads

- **`cubic`**: no-op. Local findings have no remote resolution state.
  A follow-up spec adds `--no-resolve` semantics and a local store of
  handled finding IDs; until then, every cubic-backend iteration sees
  the full set of issues again.

- **`gh`** / **`api`**: GraphQL `resolveReviewThread` mutation,
  batched (~20 IDs per call) to stay under GitHub's rate budget:

  ```bash
  gh api graphql -f query='
  mutation {
    t1: resolveReviewThread(input: {threadId: "MDIyOk..."}) { thread { isResolved } }
    t2: resolveReviewThread(input: {threadId: "MDIyOk..."}) { thread { isResolved } }
  }'
  ```

  Build the mutation body programmatically by joining `tN:
  resolveReviewThread(...)` clauses. If a single resolve fails (e.g.
  someone already resolved out-of-band), GraphQL returns a partial
  result with per-alias errors — continue with the rest.

## Detect stamp (auto-approval)

- **`cubic`**: N/A. The CLI does not approve PRs. Always `STAMP=""`.

- **`gh`**:

  ```bash
  STAMP=$(gh pr view "$PR" --json reviews \
    | jq -r --arg bot "$CUBIC_BOT" \
        '[.reviews[]
          | select(.author.login == $bot)
          | select(.state == "APPROVED")
         ] | last | .id // ""')
  ```

  Limitation: `gh pr view --json reviews` does not expose `user.type`,
  so this recipe filters on the resolved bot login alone. A regular
  user whose login happens to match `$CUBIC_BOT` exactly would still
  spoof a stamp — defense is the dynamic `$CUBIC_BOT` resolution
  (which already requires `user.type == "Bot"`). If absolute certainty
  is required, use the `api` recipe below.

- **`api`**:

  `gh api --jq` does not accept jq CLI flags like `--arg`. Pipe
  through the real `jq` binary so the bot login is passed safely:

  ```bash
  STAMP=$(gh api "repos/$OWNER/$REPO/pulls/$PR/reviews" \
    | jq -r --arg bot "$CUBIC_BOT" \
        '[.[]
          | select(.user.type == "Bot")
          | select(.user.login == $bot)
          | select(.state == "APPROVED")
         ] | last | .id // ""')
  ```

If `$CUBIC_BOT` hasn't been resolved yet (cubic hasn't reviewed the PR
at all), stamp detection is by definition negative — skip the query.
If non-empty, cubic has stamped this PR.

## Triggering a re-review (`gh` / `api`)

cubic responds to `@cubic-dev review` as a PR comment. Posting the
same comment again on an unchanged commit is a no-op — cubic dedupes.
So only post when:

- the latest check run is `completed` (we want a fresh one), **and**
- we've pushed something since the last completed review (HEAD SHA
  differs from the SHA cubic last reviewed).

If a given installation uses a different handle (e.g. `@cubic
review`), the trigger comment in `SKILL.md` step B is the only thing
that needs to change.

## When cubic seems stuck

If the check run sits at `in_progress` past the per-iteration timeout
(`gh` / `api` backends):

1. Check the PR for a recent cubic comment that says "skipping review"
   or "ignoring" — `cubic.yaml`'s `reviews.ignore` rules may be hiding
   the diff (PR title regex, label, ignored files).
2. Check the PR's draft state — `check_drafts: false` in `cubic.yaml`
   means draft PRs aren't reviewed.
3. Check external-contributor status —
   `external_contributors_require_manual_review` means a maintainer
   has to click "Approve cubic to review" once before the bot will
   engage.

For the `cubic` backend, "stuck" typically means `cubic review` hit
its `--timeout`. Re-run with `--print-logs --log-level DEBUG` to see
where the underlying AI provider call hung (usually auth or rate
limits — see `cubic auth list`).

`cubic-loop` surfaces these via the report's "Remaining issues" block;
don't try to auto-resolve them.
