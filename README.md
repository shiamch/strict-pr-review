# Strict PR Review Skill

This document describes the `strict-pr-review` skill so it can be added to a
GitHub repository as contributor or reviewer documentation.

The skill turns a normal PR review into a merge-gate review. Its purpose is to
find confirmed production blockers before a branch is merged, especially in
areas where regressions are easy to miss: permissions, async state, data
contracts, persistence, accessibility, large-install performance, and release
configuration.

## When To Use

Use this review mode when a PR needs a strict production-risk pass, such as:

- Reviewing a branch before merge or release.
- Checking requested changes on a high-risk PR.
- Looking for blockers in API, persistence, permission, import, generation, or
  UI state changes.
- Re-reviewing a PR after a blocker was fixed.
- Asking for an `autharif`-style review that prioritizes confirmed issues over
  encouragement, summaries, or style feedback.

Do not use this mode for routine style cleanup, formatting preferences, or
speculative architecture discussion unless those concerns create a confirmed
production issue.

## Reviewer Stance

The reviewer should behave like a merge gate:

- Start with findings, not praise.
- Request changes only for confirmed `Critical` or `Important` blockers.
- Put uncertain concerns under `Needs verification`.
- Include an exact file and line for each blocker whenever possible.
- Explain the production impact, not just why the code looks wrong.
- Provide a concise fix direction, not a full patch.
- Approve clearly when no confirmed blocker exists.

The skill favors one strong confirmed blocker over a long list of weak guesses.

## Inspection Workflow

1. Identify the base branch. Prefer the PR base branch; otherwise use `develop`
   if that is the repository convention.
2. Inspect the branch state, recent commits, and changed files.
3. Read the changed files and the surrounding code path, not only the diff.
4. Trace the affected behavior end to end:
   - UI action or component state.
   - API route or action name.
   - Policy or capability check.
   - Controller or handler.
   - Service, model, query, meta, option, or database write.
   - Response payload.
   - UI success, failure, retry, and loading-state handling.
5. For generated assets, verify whether the source file changed too.
6. For frontend async work, inspect both success and failure paths.
7. For backend changes, inspect persistence contracts, database column limits,
   hooks, filters, and existing callers.

## Blocker Rubric

### Async And UI State

Request changes when stale async responses can overwrite the currently selected
record, a failed request leaves a modal or form stuck in a processing state, or
parent and child components can desynchronize around running, importing, or
submitting state.

Typical fix direction: capture the requested entity ID before the request,
ignore late responses when the active ID changed, reset loading in
`catch`/`finally`, and expose a retryable error state.

### Permission And Security Boundaries

Request changes when read-only capabilities can trigger paid AI calls, writes,
subscriber metadata updates, option updates, imports, sends, deletes, or other
side effects. Also block when endpoint policy does not match the capability
domain used by the UI.

Typical fix direction: split read and write endpoints, or gate side-effecting
work behind the correct write-level capability.

### Data Contracts And Persistence

Request changes when a response field changes semantic meaning, backend and
frontend payload shapes drift apart, backward-compatible keys are removed while
older callers still use them, or generated values can exceed persistence limits.

Typical fix direction: preserve old semantics, add new fields for new meanings,
keep legacy keys where needed, and cap values before saving.

### Performance And Large Installs

Request changes when interactive flows add unbounded queries, repeatedly rebuild
expensive catalogs, reload full datasets inside validation loops, or serialize
large prompt context more than once per request.

Typical fix direction: build expensive catalogs once per request, reuse resolved
data for prompt generation and validation, and pass request-scoped data through
