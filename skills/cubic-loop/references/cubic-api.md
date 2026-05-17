# cubic-api reference

Cubic-specific GitHub API recipes used by `cubic-loop`. Kept out of
`SKILL.md` so the main flow stays scannable.

All commands below assume `$OWNER`, `$REPO`, `$PR`, and `$HEAD_SHA` are
already set, as in `SKILL.md` step C/D.

## Identifying the cubic bot

cubic posts under a GitHub App bot account. The exact login varies by
installation but matches `^cubic.*\[bot\]$` (commonly `cubic-dev-ai[bot]`).
**Detect dynamically** rather than hard-coding — the skill should keep
working if cubic renames its bot:

```bash
CUBIC_BOT=$(
  gh api "repos/$OWNER/$REPO/pulls/$PR/reviews" \
    --jq '[.[] | select(.user.type == "Bot") | .user.login | select(test("^cubic"; "i"))] | last // empty'
)
```

If `$CUBIC_BOT` is empty, cubic hasn't reviewed this PR yet — wait
another poll cycle in `SKILL.md` step C before re-trying.

## Cubic check run

cubic posts a GitHub check run whose name matches `^cubic`
(case-insensitive). Pull the latest one for the head commit:

```bash
gh api "repos/$OWNER/$REPO/commits/$HEAD_SHA/check-runs" \
  --jq '[.check_runs[] | select(.name | test("^cubic"; "i"))] | last'
```

Useful fields: `status` (`queued`/`in_progress`/`completed`),
`conclusion` (when completed: `success`/`failure`/`neutral`/`cancelled`),
`details_url`.

## Latest cubic review summary

Look up the most recent review whose body opens with the contracted
score line:

```bash
gh api "repos/$OWNER/$REPO/pulls/$PR/reviews" \
  --jq '[.[] | select(.body | test("^PR score:"; "m"))] | sort_by(.submitted_at) | last'
```

Returned object includes `body`, `state` (`APPROVED` / `CHANGES_REQUESTED`
/ `COMMENTED`), `submitted_at`, `user.login`.

Parse the score from the body:

```bash
SCORE=$(printf '%s' "$REVIEW_BODY" | grep -Eo '^PR score: [0-9]+/10' | head -1 | grep -Eo '[0-9]+' | head -1)
```

## Unresolved cubic review threads (GraphQL)

REST's `pulls/<n>/comments` doesn't expose thread-resolution state.
Use the GraphQL `reviewThreads` connection:

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
            nodes {
              body
              author { login }
            }
          }
        }
      }
    }
  }
}' -F owner="$OWNER" -F repo="$REPO" -F pr="$PR"
```

Filter to threads that are:

- `isResolved == false`
- `isOutdated == false` (cubic comments on stale lines are usually safe
  to ignore — but flag them in the report)
- `comments.nodes[0].author.login` matches `^cubic.*\[bot\]?$`

Paginate via `pageInfo.endCursor` while `hasNextPage` is true.

## Resolving threads

Batch resolves in a single GraphQL mutation per ~20 thread IDs to keep
under GitHub's mutation rate budget:

```bash
gh api graphql -f query='
mutation {
  t1: resolveReviewThread(input: {threadId: "MDIyOk..."}) { thread { isResolved } }
  t2: resolveReviewThread(input: {threadId: "MDIyOk..."}) { thread { isResolved } }
}'
```

Build the mutation body in your loop language (bash heredoc, jq) by
joining `tN: resolveReviewThread(...)` clauses for each thread ID.

If a single resolve mutation fails (e.g. someone already resolved the
thread out-of-band), GraphQL returns a partial result with errors per
alias — continue with the rest.

## Detecting a cubic stamp (auto-approval)

cubic's stamp = a `state == "APPROVED"` review authored by the cubic
bot:

```bash
STAMP=$(
  gh api "repos/$OWNER/$REPO/pulls/$PR/reviews" \
    --jq '[.[] | select(.user.login | test("^cubic"; "i")) | select(.state == "APPROVED")] | last | .id // empty'
)
```

If non-empty, cubic has stamped this PR.

## Triggering a re-review

cubic responds to `@cubic-dev review` as a PR comment. Posting the same
comment again on an unchanged commit is a no-op — cubic dedupes. So
only post when:

- the latest check run is `completed` (we want a fresh one), **and**
- we've pushed something since the last completed review (HEAD SHA
  differs from the SHA cubic last reviewed).

If a given installation uses a different handle (e.g. `@cubic review`),
the trigger comment in `SKILL.md` step B is the only thing that needs
to change.

## When cubic seems stuck

If the check run sits at `in_progress` past the per-iteration timeout:

1. Check the PR for a recent cubic comment that says "skipping review"
   or "ignoring" — `cubic.yaml`'s `reviews.ignore` rules may be hiding
   the diff (PR title regex, label, ignored files).
2. Check the PR's draft state — `check_drafts: false` in `cubic.yaml`
   means draft PRs aren't reviewed.
3. Check external-contributor status — `external_contributors_require_manual_review`
   means a maintainer has to click "Approve cubic to review" once before
   the bot will engage.

`cubic-loop` surfaces these via the report's "Remaining issues" block;
don't try to auto-resolve them.
