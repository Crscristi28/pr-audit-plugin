# Tier decision logic

The orchestrator classifies every PR diff into one of three tiers. The tier determines whether to spawn subagents or review inline, and which subagents to spawn.

This decision is **deterministic** — apply the rules below mechanically. Do not let the LLM "feel" the tier; compute it from facts.

## Inputs to compute

After applying noise filters (see `noise-filters.md`), compute:

- `lines_changed` = total lines added + lines removed across all non-filtered files
- `files_changed` = count of distinct files touched (after noise filtering)
- `security_sensitive` = boolean. True if any changed file matches a security-sensitive pattern (see `path-patterns.md`)
- `db_schema_changed` = boolean. True if any changed file matches db-schema patterns
- `api_surface_changed` = boolean. True if any changed file matches api-surface patterns

## Tier rules (apply in order, first match wins)

### Inline

Trigger:
- `lines_changed <= 50` AND
- `files_changed <= 3` AND
- `security_sensitive == false` AND
- `db_schema_changed == false` AND
- `api_surface_changed == false`

Action: Orchestrator reviews the diff directly using the three domain checklists (security, correctness, performance) in a single LLM pass. **No subagent spawn.** Returns the same YAML output schema as a subagent.

Rationale: For small diffs in non-sensitive paths, spawn overhead is not worth it. The orchestrator can hold all three domain perspectives in one context.

### Deep

Trigger (any of):
- `lines_changed > 200` OR
- `db_schema_changed == true` OR
- `security_sensitive == true` AND `lines_changed > 50` OR
- `api_surface_changed == true` AND `lines_changed > 50`

Action: Spawn the **3 core subagents** in parallel via the Task tool:
- `security-reviewer`
- `correctness-reviewer`
- `performance-reviewer`

PLUS the relevant **Tier 2 conditional subagents** based on which path flags are set:

- `api_surface_changed == true` → also spawn `api-contract-reviewer`
- `db_schema_changed == true` → also spawn `architecture-reviewer`
- `frontend_changed == true` → also spawn `a11y-reviewer`
- One or more source files in the diff have no matching test sibling (e.g., `src/foo/bar.ts` exists but `src/foo/bar.test.ts`, `src/foo/__tests__/bar.test.ts`, or equivalent does not appear in the diff or context) → also spawn `test-coverage-reviewer`
- `.pr-audit/rules.md` exists in the repo root AND contains a section with the heading `## Business Logic` → also spawn `business-logic-reviewer` (passing the rule text in its prompt)

A single Deep tier review may spawn between 3 and 8 subagents depending on which Tier 2 triggers fire. All spawn in parallel via Task tool calls in a single batch.

### Standard

Trigger: everything else. Falls through when neither Inline nor Deep matches.

Action: Spawn the **3 core subagents** in parallel via the Task tool. **No Tier 2 subagents in Standard tier** — they are reserved for Deep tier where the diff is large enough or sensitive enough to warrant the extra review depth.

Rationale: Standard tier handles the bulk of typical PRs (50–200 lines, no special path triggers). The 3 core subagents cover the most common bug classes without overhead.

## Edge case: small diff, security-sensitive path

Example: a 15-line change in `src/auth/login.ts`.

Inline rule fails (`security_sensitive == true`). Deep rule fails (`lines_changed <= 50`). Falls through to Standard → spawn all 3 core subagents.

This is the correct behavior. Even small diffs in security-sensitive paths deserve the focused security-reviewer subagent.

## Edge case: doc-only changes

If every non-noise-filtered file matches `*.md`, `*.txt`, `LICENSE`, `.gitignore`, or `docs/**`, then `lines_changed` may be > 50 but the diff is documentation only.

Special handling: orchestrator may post a minimal top-level comment ("Documentation-only changes, no code review performed") and skip both subagent spawn and inline review. This is the equivalent of approving cleanly.

## Edge case: PR is too large

If `lines_changed > 2000` or `files_changed > 50`, the orchestrator emits a warning in the PR summary comment recommending the PR be split. Review continues at Deep tier but findings may be incomplete due to context limits. The handoff brief includes a "PR too large" note for the user's main session.

## Output of the tier decision step

The orchestrator records the decision in its internal state:

```yaml
tier: inline | standard | deep
tier_reason: "Lines changed (37) under 50, files (2) under 3, no special paths"
subagents_to_spawn:
  - security-reviewer
  - correctness-reviewer
  - performance-reviewer
path_flags:
  security_sensitive: false
  db_schema_changed: false
  api_surface_changed: false
```

This state is consumed by the spawn step and recorded in the handoff brief for transparency.
