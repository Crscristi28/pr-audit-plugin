# Handoff brief template

After the review is posted on GitHub, the orchestrator prints a **short handoff brief directly in the chat / terminal**. It does NOT write a file.

Rationale: every finding is already an inline comment on the GitHub PR — that is the durable, precise record. A long brief that re-types each finding just duplicates the PR and burns tokens. The brief's only job is a quick orientation: verdict, scale, where to look. For full detail the reader opens the PR.

## Template

The brief is short — roughly 10–15 lines. Print it as plain markdown in the chat:

```markdown
## PR Audit — PR #{N}

**Verdict:** {verdict} — {one-line verdict consequence, e.g. "do not merge" / "ok to merge"}
**Tier:** {tier} · {subagent_count} subagents · {lines_changed} lines, {file_count} files
**PR:** {pr_url}

**Findings:** 🔴 {critical_count} critical · 🟠 {warning_count} warning · 🟡 {suggestion_count} suggestion
Every finding is an inline comment on the PR — open the PR for full detail and suggested fixes.

**Top concerns:**
- {critical finding 1 — issue_header only}
- {critical finding 2 — issue_header only}
- {critical finding 3 — issue_header only}

{if any low-confidence findings were withheld from GitHub:}
**Not posted (low confidence, verify yourself):** {comma-separated issue_headers}

→ Fix from the PR comments. After pushing fixes, re-run `/pr-audit {N}` to verify.
```

Rules:
- **Top concerns**: list at most 3, critical first; if fewer than 3 criticals, fill with the highest warnings; if no findings at all, omit the section.
- Never re-type `issue_content` or `suggested_fix` — those live in the PR comments.
- No "Instructions" section, no "What was verified" section, no per-finding blocks. Keep it tight.
- Print it in the chat. Do not create `outputs/`, do not write a `.md` file.

## Field substitution

| Placeholder | Source |
|-------------|--------|
| `{N}` | PR number from `gh pr view` or argument |
| `{pr_url}` | `gh pr view --json url` |
| `{tier}` | From tier-decision step |
| `{subagent_count}` | Number of subagents spawned (0 for Inline tier — say "inline review, no subagents") |
| `{lines_changed}` | Total added + removed across review scope |
| `{file_count}` | Number of files in scope after noise filtering |
| `{verdict}` | From approval-rubric step |
| `{critical_count}` / `{warning_count}` / `{suggestion_count}` | Counts from the master findings table (Phase 7.4) |

## Edge case: clean review (no findings)

If the review produced zero findings, the brief is even shorter:

```markdown
## PR Audit — PR #{N}

**Verdict:** approved — no issues found in {lines_changed} lines across {file_count} files.
**PR:** {pr_url}

The PR was auto-approved on GitHub. Merge when ready: `gh pr merge {N} --squash`
```
