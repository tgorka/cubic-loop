---
title: 'cubic-loop pluggable backends (cubic CLI / gh CLI / gh api) with fallback chain'
type: 'feature'
created: '2026-05-18'
status: 'done'
baseline_commit: '00e64c5d1b0e326e022c869ef91185b1f22115ef'
context:
  - '{project-root}/skills/cubic-loop/SKILL.md'
  - '{project-root}/skills/cubic-loop/references/cubic-api.md'
  - '{project-root}/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** cubic-loop hard-fails on missing `cubic.yaml` and talks to GitHub solely via `gh api`. When the cubic.dev GitHub app is not installed on a repo, the loop is unusable — even though the `cubic` CLI ships a `cubic review --json` mode that performs a local headless review without any server-side config.

**Approach:** Introduce a `--backend auto|cubic|gh|api` flag that selects the transport for each cubic-related operation (preflight, trigger, fetch verdict, poll check, list threads, resolve threads). `auto` probes `cubic` (authed) → `gh` (authed) → halt. The `cubic.yaml` preflight is downgraded to a notice when the active backend is `cubic`. Backends are a transport layer only; loop semantics (score thresholds, max-iters, timeout, exit conditions) are unchanged.

## Boundaries & Constraints

**Always:**
- `--backend` defaults to `auto`; legal values: `auto`, `cubic`, `gh`, `api`.
- `auto` probe order: `cubic` (requires `command -v cubic` AND `cubic auth list` showing creds) → `gh` (requires `gh auth status` success) → halt with a clear error.
- Each loop operation has an explicit per-backend recipe in `references/cubic-api.md`. Missing-recipe ≠ silent skip.
- `gh` backend prefers `gh pr <subcmd>` where one exists; cascades to `gh api` for operations gh CLI does not expose (review-thread GraphQL, `resolveReviewThread` mutation). This cascade is internal to the `gh` backend — not a backend switch.
- `cubic.yaml` preflight: hard-fail when active backend is `gh` or `api`; notice when active backend is `cubic`.
- Report block prints `Backend: cubic|gh|api` so the user can see which transport ran.

**Ask First:**
- If `--backend=cubic` is explicit but `cubic` is missing/unauthed → HALT with install hint, do not silently fall through.
- If `auto` picks `gh`/`api` AND `cubic.yaml` is also missing → HALT (no path to a review exists).

**Never:**
- Auto-install or upgrade `cubic`, `gh`, or any other binary.
- Change loop exit conditions, score parsing rules, or iteration counters per backend.
- Add backends beyond cubic CLI and gh CLI/api in this spec.
- Implement `--no-resolve` / save/restore thread IDs — that is a follow-up spec.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| `auto`, cubic authed | `--backend auto`, `cubic` in PATH, `cubic auth list` non-empty | Select `cubic`; cubic.yaml preflight = notice | N/A |
| `auto`, no cubic, gh authed | no `cubic`, `gh auth status` OK | Select `gh`; cubic.yaml preflight = hard-fail | If cubic.yaml missing → halt |
| `auto`, neither | no `cubic`, no `gh` auth | Halt during preflight | Exit non-zero with both install hints |
| explicit cubic missing | `--backend cubic`, `cubic` not in PATH | Halt with `curl ... \| sh` install hint | Exit non-zero |
| explicit gh, no cubic.yaml | `--backend gh`, repo without cubic.yaml | Halt (current behavior preserved) | Exit non-zero |
| `cubic` JSON has no score | `cubic review --json` output lacks score field | Surface "cubic emitted no score" + raw output, stop | Exit non-zero |
| `gh` lists threads | active backend `gh`, listing unresolved threads | `gh api graphql ...` (gh CLI has no native subcmd) | N/A — same auth |

</frozen-after-approval>

## Code Map

- `skills/cubic-loop/SKILL.md` -- main skill flow; needs `--backend` arg, backend-selection section, per-step transport annotations, updated Preflight and Report.
- `skills/cubic-loop/references/cubic-api.md` -- recipe library; needs reorganization into per-operation sections each holding three sub-recipes (cubic / gh / api).
- `AGENTS.md` -- repo's contributor doc; verify nothing contradicts the new flag.

## Tasks & Acceptance

**Execution:**
- [x] `skills/cubic-loop/SKILL.md` -- add `--backend auto|cubic|gh|api` row to Inputs table -- exposes the flag.
- [x] `skills/cubic-loop/SKILL.md` -- insert "Backend selection" section after Inputs -- documents probe order, per-operation transport, and the report line.
- [x] `skills/cubic-loop/SKILL.md` -- update Preflight item 4 -- branch on active backend: cubic → notice; gh/api → existing hard-fail.
- [x] `skills/cubic-loop/SKILL.md` -- annotate Loop steps B (trigger), C (poll), D (fetch verdict), F (list threads), G (resolve threads) -- each step references the corresponding recipe block in `references/cubic-api.md` and shows the dispatch line.
- [x] `skills/cubic-loop/SKILL.md` -- update Report -- add `Backend: <name>` line.
- [x] `skills/cubic-loop/references/cubic-api.md` -- reorganize into per-operation sections (Trigger review, Poll check, Fetch verdict, List unresolved threads, Resolve threads, Detect stamp); each section has cubic / gh / api recipes.
- [x] `skills/cubic-loop/references/cubic-api.md` -- document `cubic review --json` output shape -- score field path, findings array structure, mapping to score/threads/stamp.
- [x] `.changeset/cubic-loop-pluggable-backends.md` -- add changeset (per AGENTS.md "every prose change is potentially user-visible").

**Acceptance Criteria:**
- Given `cubic` is installed and `cubic auth list` shows creds, when user runs `/cubic-loop --backend auto` in a repo without `cubic.yaml`, then the skill picks the cubic backend, emits a notice instead of halting, and proceeds.
- Given no `cubic` but `gh` is authed, when user runs `/cubic-loop` (default `auto`), then the skill picks the gh backend and the current cubic.yaml hard-fail is preserved.
- Given `--backend cubic` is explicit but `cubic` is not in PATH, when preflight runs, then the skill halts with a single-line install hint and a non-zero exit; it does NOT fall through to gh.
- Given the gh backend is active, when listing unresolved threads, then the skill cascades to `gh api graphql` while using `gh pr comment` / `gh pr view` / `gh pr checks` for operations that have native subcommands.
- Given any backend is active, when the loop reports, then the final report block contains a `Backend: <name>` line matching the backend that actually ran.

## Design Notes

Backends abstract **transport**, not loop policy. The score parser, exit rules, and iteration counter live in `SKILL.md` once and consume backend-normalized values (`SCORE`, `STAMP`, `UNRESOLVED`). Each backend is responsible for producing those values from its native API.

Per-operation mapping (high-level — full recipes in `references/cubic-api.md`):

| Operation       | cubic                                  | gh                            | api                              |
|-----------------|----------------------------------------|-------------------------------|----------------------------------|
| Trigger review  | `cubic review --json` (local)          | `gh pr comment "$PR" -b "@cubic-dev review"` | `gh api ... -X POST` issue comment |
| Poll check      | synthetic (returns `completed` after cubic exits) | `gh pr checks "$PR" --json ...` | `gh api .../check-runs` |
| Fetch verdict   | parse CLI JSON                         | `gh pr view --json body,reviews` | `gh api .../pulls/.../reviews`  |
| List threads    | parse CLI findings array               | cascade → `gh api graphql`    | `gh api graphql`                 |
| Resolve threads | (deferred — spec 2)                    | cascade → `gh api graphql`    | `gh api graphql` mutation        |
| Detect stamp    | N/A (CLI never stamps)                 | `gh pr view --json reviews`   | `gh api .../pulls/.../reviews`   |

For the cubic backend, `resolveReviewThread` has no remote analogue this iteration — leave the resolve step as a no-op and note that spec 2 (`--no-resolve` + save/restore) will define local persistence semantics.

Auto-probe inline snippet for SKILL.md:

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
  cubic|gh|api) ;;
  *) die "unknown --backend: $BACKEND" ;;
esac
```

## Verification

**Commands:**
- `grep -cE '^## Backend selection' skills/cubic-loop/SKILL.md` -- expected: `1`.
- `grep -nE '^## (Trigger review|Poll check|Fetch verdict|List unresolved threads|Resolve threads|Detect stamp)' skills/cubic-loop/references/cubic-api.md` -- expected: six matches, one per operation.
- `grep -c 'Backend:' skills/cubic-loop/SKILL.md` -- expected: at least 2 (the selection section + the Report block).

**Manual checks:**
- Read SKILL.md end-to-end. For each cubic API touchpoint in Loop steps B/C/D/F/G, confirm a backend dispatch is visible (either inline `case "$BACKEND" in ...` or an explicit reference to a recipe section).
- Confirm the report block in SKILL.md prints `Backend: <name>` in both the success and the incomplete templates.

## Suggested Review Order

**Design entry point — Backend selection contract**

- Start here: the Inputs row + Backend selection section define the flag, the auto-probe, and the per-operation transport matrix.
  [`SKILL.md:45`](../../skills/cubic-loop/SKILL.md#L45)
  [`SKILL.md:50`](../../skills/cubic-loop/SKILL.md#L50)

**cubic CLI integration — local review semantics**

- The CLI JSON schema and the SCORE/STAMP/UNRESOLVED derivation that turns local issues into loop-normalized values.
  [`cubic-api.md:19`](../../skills/cubic-loop/references/cubic-api.md#L19)

- Cubic-backend Trigger recipe with `timeout` rc handling (so a killed CLI never leaves truncated JSON downstream).
  [`cubic-api.md:80`](../../skills/cubic-loop/references/cubic-api.md#L80)

- Cubic-backend Fetch-verdict recipe: `has("issues")` shape check, `.error` short-circuit, then SCORE derivation.
  [`cubic-api.md:147`](../../skills/cubic-loop/references/cubic-api.md#L147)

**Preflight + loop dispatch**

- Preflight item 4: cubic.yaml is a notice under the cubic backend (the actual screenshot blocker), hard-fail elsewhere.
  [`SKILL.md:120`](../../skills/cubic-loop/SKILL.md#L120)

- Loop step B (trigger) — cubic vs gh vs api dispatch with rationale for each path.
  [`SKILL.md:190`](../../skills/cubic-loop/SKILL.md#L190)

- Loop step D (fetch verdict) — backend-normalized SCORE/STAMP/UNRESOLVED; the contract the exit conditions consume.
  [`SKILL.md:259`](../../skills/cubic-loop/SKILL.md#L259)

- Loop step G (resolve threads) — gh/api use the GraphQL mutation; cubic is a documented no-op pending spec 2.
  [`SKILL.md:356`](../../skills/cubic-loop/SKILL.md#L356)

**Recipes per operation (gh / api)**

- gh stamp recipe limitation (no `user.type` filter from `gh pr view`) — spelled out so future contributors don't reintroduce the safety check incorrectly.
  [`cubic-api.md:280`](../../skills/cubic-loop/references/cubic-api.md#L280)

- Explicit-backend availability checks (cubic / gh / api arms validate before falling through).
  [`SKILL.md:68`](../../skills/cubic-loop/SKILL.md#L68)

**Peripherals**

- Report block adds `Backend:` line in both success and incomplete templates.
  [`SKILL.md:391`](../../skills/cubic-loop/SKILL.md#L391)

- Changeset entry.
  [`cubic-loop-pluggable-backends.md`](../../.changeset/cubic-loop-pluggable-backends.md)

- Deferred-work log: out-of-scope findings from review (gh hostname mismatch, cubic binary collision, provider outage retry, etc.).
  [`deferred-work.md`](deferred-work.md)
