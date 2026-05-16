# Approval rubric

After aggregating findings from all subagents (or from inline review), the orchestrator decides the verdict and the corresponding GitHub action.

This decision is deterministic — apply the rules below in order. Bias is **toward approval**: a single warning in an otherwise clean PR should not block merge.

## Inputs

Counts of findings AFTER confidence filtering (low-confidence non-critical findings are already dropped per `output-schema.md`).

- `critical_count` = number of findings with `severity: critical`
- `warning_count` = number of findings with `severity: warning`
- `suggestion_count` = number of findings with `severity: suggestion`

## Decision rules (apply in order, first match wins)

### significant_concerns

Trigger:
- `critical_count >= 1`

GitHub action: post inline comments + summary. The submit `event` is mode-dependent (see "GitHub submit action by verdict" below): `REQUEST_CHANGES` in review-only mode, `COMMENT` in auto mode. The summary clearly explains why the PR cannot be merged as-is.

### minor_issues

Trigger:
- `critical_count == 0` AND
- `warning_count >= 3`

GitHub action: post inline comments + summary, no review approval or rejection. The summary recommends addressing warnings before merge but does not block.

Rationale: multiple warnings in one PR suggest a pattern that warrants attention without being a hard block.

### approved_with_comments

Trigger:
- `critical_count == 0` AND
- (`warning_count >= 1` OR `suggestion_count >= 1`) AND
- `warning_count < 3`

GitHub action: post inline comments + summary. The submit `event` is mode-dependent (see "GitHub submit action by verdict" below): `APPROVE` in review-only mode, `COMMENT` in auto mode (you cannot approve your own PR).

Rationale: the PR is acceptable but the author should consider the flagged items.

### approved

Trigger:
- `critical_count == 0` AND `warning_count == 0` AND `suggestion_count == 0`

GitHub action: post a brief top-level summary saying the review is clean. The submit `event` is mode-dependent (see "GitHub submit action by verdict" below): `APPROVE` in review-only mode, `COMMENT` in auto mode.

## GitHub submit action by verdict

The verdict is mode-independent. The **GitHub review `event`** used to submit it is **mode-dependent**, because GitHub rejects a formal `APPROVE` / `REQUEST_CHANGES` review on a PR authored by the same user (HTTP 422). In auto mode the plugin authored the PR, so it is always the author.

Inline comments and the verdict are submitted together as one batched review via `POST .../pulls/{N}/reviews` (see SKILL.md Phase 8.3) with this `event`:

| Verdict | Auto mode `event` | Review-only mode `event` |
|---------|-------------------|--------------------------|
| `approved` | `APPROVE` | `APPROVE` |
| `approved_with_comments` | `COMMENT` | `APPROVE` |
| `minor_issues` | `COMMENT` | `COMMENT` |
| `significant_concerns` | `COMMENT` | `REQUEST_CHANGES` |

For `minor_issues`, the review is always `COMMENT` — requesting changes for "three warnings" would feel heavy-handed; the comments communicate the concerns. For `significant_concerns` in auto mode, `COMMENT` is the correct default (not a fallback) since the author cannot request changes on their own PR — the blocking verdict is stated explicitly in the review body and top-level summary.

## Verdict reasoning

In the `verdict_reason` field of the output schema, write one sentence explaining the verdict:

| Verdict | Example reason |
|---------|---------------|
| `approved` | "No issues found in the diff." |
| `approved_with_comments` | "One warning and one suggestion; the warning does not block merge." |
| `minor_issues` | "Three warnings across security and correctness — pattern worth addressing before merge." |
| `significant_concerns` | "Critical SQL injection in `src/api/users.ts` line 42 must be fixed before merge." |

## Override: break-glass commands

If a user posts a comment on the PR containing the exact text `pr-audit: skip-review` before re-running `/pr-audit {N}`, the plugin skips review and posts a top-level comment: *"Review skipped by user request."* This is the equivalent of a manual override and exists for emergency-shipping situations.

This is detected by reading existing PR comments via `gh pr view {N} --json comments` before starting the review.

## Re-review behavior

When a user pushes a fix commit and re-runs `/pr-audit {N}`, the orchestrator:

1. Reads its own previous comments on the PR (identified by an HTML marker like `<!-- pr-audit:finding:{id} -->`).
2. For each previous finding, checks if the issue is still present in the current diff.
3. If resolved → posts a reply to the thread: `✅ Resolved in this revision.` and calls `gh api graphql` to resolve the thread.
4. If still present → re-emits the finding in the new review (it stays open).
5. New findings from the new diff are added.

The verdict is recomputed based on the current state of findings.

## Edge case: only documentation changes

If the orchestrator's noise-filter step concluded that the PR is documentation-only (see `noise-filters.md` and `tier-decision.md`), the verdict is automatically `approved` with summary: *"Documentation-only changes, no code review performed."* No inline comments.

## Edge case: PR too large

If the PR exceeds `lines_changed > 2000` or `files_changed > 50`, the orchestrator includes this warning in the summary regardless of verdict: *"This PR is large (X lines, Y files). The review may be incomplete due to context limits. Consider splitting into smaller PRs."*

The verdict itself is computed normally from the findings the reviewers did surface.
