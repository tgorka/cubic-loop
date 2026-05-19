# Deferred work

Findings surfaced during spec-cubic-loop-backends review that are out of
scope for that spec but should be revisited.

## From spec-cubic-loop-backends review (2026-05-18)

- **gh-auth-status hostname mismatch.** `gh auth status` returns 0 even
  if authed to a different host (e.g., GHE while repo is on
  github.com). Auto-probe currently accepts that and the loop fails
  later in `gh pr view`. Fix: probe with
  `gh auth status --hostname <host-from-remote>`.
- **`cubic` binary name collision.** Another tool named `cubic` could
  shadow the cubic.dev CLI. Currently auto-probe trusts `command -v`.
  Mitigation: also check `cubic --version` matches expected pattern.
- **Provider outage retry semantics.** When `cubic review` sets
  `.error` due to a transient provider hiccup, the loop dies. Could
  add bounded retry before halting.
- **Step F.4 "resolve message" for gh/api backends.** GraphQL
  `resolveReviewThread` mutation has no message field — the
  long-standing "note the reason in the resolve message" instruction
  is unreachable. Pre-existing in original SKILL.md.
- **Regex `^cubic.*\[bot\]?$` typo.** `references/cubic-api.md` thread
  filter has the trailing `?` making `[bot]` optional; the bot-id
  resolver uses the strict `\[bot\]$`. Unify on the strict form.
  Pre-existing in original references file.
- **`cubic` backend convergence with no resolve persistence.** Without
  the spec-2 save/restore mechanism, every cubic-backend iteration
  re-emits the same findings even if step F addressed them. Spec 2
  (`--no-resolve` + save/restore thread IDs) addresses this.
- **`--backend cubic` semantics when a PR exists.** The spec assumes
  cubic backend still operates against the PR (git push, commit, etc.).
  An offline "no PR" mode would require gating preflight #5, steps A
  and H, and the Report block on PR presence. Future enhancement.
- **`--timeout` does not wrap `gh pr comment` POST.** Network calls in
  step B can hang past the deadline. Wrap with `timeout` for
  uniformity. Pre-existing.
