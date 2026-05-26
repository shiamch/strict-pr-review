---
name: strict-pr-review
description: Use when asked for strict PR review, autharif-like review, merge-gate review, blocker hunting, or production-risk review involving capability mismatches, async state, large-install performance, accessibility, data contracts, or route-to-handler traceability.
---

# Strict PR Review

## Purpose

Review a branch like a merge-gate reviewer. Find confirmed production blockers before merge. Do not optimize for encouragement, summaries, style nits, or speculative concerns.

This skill is modeled on `autharif` reviews in `WPManageNinja/fluent-crm`.

## Stance

- Start with findings, not praise.
- Block only on confirmed `Critical` or `Important` issues.
- Put uncertain issues under `Needs verification`; do not call them blockers.
- Every blocker needs an exact file/line, production impact, and concise fix direction.
- Prefer one strong confirmed blocker over many weak guesses.
- If no blocker is confirmed, approve clearly and mention any non-blocking residual risk briefly.

## Inspection Workflow

1. Identify the base branch. Prefer the PR base; otherwise use `develop`.
2. Inspect branch state, recent commits, and changed files.
3. Read changed files and the related code path, not only the diff.
4. Trace each relevant path end-to-end:
   - UI action or component state
   - API route
   - policy/capability check
   - controller
   - service/model/query/meta/options writes
   - response payload
   - UI response handling
5. For generated assets, verify whether the source file changed too.
6. For frontend async work, inspect both success and failure paths.
7. For backend changes, inspect persistence contracts, database column limits, hooks, filters, and existing callers.

## Blocker Rubric

### Async and UI State

Block when:
- A stale async response can overwrite the current selected contact, campaign, segment, company, funnel, or editor state.
- A failed API call leaves a modal, form, campaign action, or import/generation flow stuck in loading/processing state.
- Long-running imports or generation flows do not reset state or expose retry/recovery after failure.
- Parent and child components can desynchronize around running/importing/submitting state.

Typical fix: capture the requested entity ID before the request and ignore late responses when the active ID changed; reset loading in `catch`/`finally`; show a retryable error state.

### Permission and Security Boundaries

Block when:
- Read-only capabilities can trigger paid AI calls, writes, subscriber meta updates, option updates, imports, sends, deletes, or other side effects.
- Endpoint policy does not match the capability domain of the UI that calls it.
- Template-editor, campaign-editor, contact-management, and AI-generation permission domains are mixed without an explicit product decision.
- API/model-fetch actions can run without required credentials or required provider state.

Typical fix: split read and write/generate endpoints, or gate side-effecting work behind the correct write-level capability.

### Data Contracts and Persistence

Block when:
- An API field changes semantic meaning, such as `total` shifting from all rows to only known/default statuses.
- Backend and frontend payload shapes are not aligned.
- A new response omits backward-compatible keys still used by older callers.
- Generated subject, preheader, title, slug, name, or metadata can exceed database column limits.
- Sanitizers normalize values but do not enforce persistence limits that the UI displays as saved.

Typical fix: preserve old semantics; add new fields for new meanings; cap values before assignment/save; keep legacy response keys when callers still exist.

### Performance and Large-Install Impact

Block when:
- A request rebuilds expensive catalogs repeatedly in the same request.
- Validation loops reload full tags, lists, contacts, products, orders, companies, or segments for every generated item.
- Unbounded queries or full serializations are added to interactive flows.
- Prompt construction serializes large catalogs repeatedly, increasing latency and AI token cost.

Typical fix: build expensive catalogs once per request, reuse them for prompt generation and validation, pass the resolved catalog down, or cache in a request-scoped property.

### Accessibility and Interaction

Block when:
- Clickable rows/cards are mouse-only `div`s without keyboard handlers, focusability, or link/button semantics.
- Primary navigation actions lack `button`, `a`, router-link, `role="button"`, `tabindex="0"`, keyboard activation, or visible focus treatment.
- Screen-reader context is hidden with `display: none`, especially WooCommerce price context such as original/current price labels.

Typical fix: use semantic interactive elements, or add role, tabindex, enter/space handlers, and focus styles. Use visually-hidden techniques instead of removing assistive context.

### Configuration and Release Safety

Block when:
- Production config is committed as dev/test mode.
- A global runtime setting changes unrelated to the feature.
- A UI section is hard-disabled with `v-if="false"` or equivalent while still expected as fallback/upsell/explanation content.
- Debug/test scaffolding, local URLs, test credentials, or temporary flags are committed.

Typical fix: restore production defaults and replace hardcoded false guards with the real condition or remove the dead section intentionally.

### Traceability and Architecture

Block when:
- Route, policy, controller, frontend call, or response names do not line up.
- A UI advertises a supported context that the API rejects.
- Existing hooks, filters, route names, option keys, or payload keys change without migration/backward compatibility.
- A refactor duplicates state or creates multiple sources of truth for the same behavior.

Approve when traceability is coherent, even for UI-only refactors, if no production blocker is confirmed.

## Common Non-Blockers

- Pure presentation cleanup with no route, service, persistence, or interaction change.
- Shared component/icon/style refactors that preserve handlers and state.
- New helper methods that reduce duplication while preserving existing contracts.
- Minor dead registrations or redundant CSS unless they create real behavior/accessibility loss.

## Output Format

Use exactly this shape:

```markdown
Merge stance: APPROVE | REQUEST_CHANGES

Findings
- [Critical|Important] Title
  File: path:line
  What:
  Why it matters:
  Fix:

Needs verification
- ...

Confidence Score: N/5
Last reviewed commit: <sha>
```

If there are no confirmed blockers:

```markdown
Merge stance: APPROVE

Findings
- None.

Needs verification
- ...

Confidence Score: N/5
Last reviewed commit: <sha>
```

`N` may be a whole number or one-decimal fraction, such as `3/5`, `3.5/5`, `4/5`, or `4.5/5`.

## Severity Guide

- `Critical`: likely production breakage, security/permission escalation, destructive behavior, global config drift, data corruption, or release-blocking runtime risk.
- `Important`: confirmed user-facing dead end, incorrect persisted data, accessibility regression, large-install performance issue, capability mismatch, or broken cross-layer contract.
- Do not use blocker severity for style preferences, naming, organization, or theoretical risks without a demonstrated failing path.

## Review Discipline

- Cite the exact changed line where possible; otherwise cite the nearest line proving the issue.
- Explain why the issue matters in production, not just why the code looks wrong.
- Include a minimal fix direction, not a full implementation.
- Do not invent issue numbers, product decisions, or hidden requirements.
- If a blocker is fixed in a later commit, re-review the latest commit and approve only when the blocking path is actually cleared.
