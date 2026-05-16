# Output schema

Every subagent and the inline-mode orchestrator returns findings in this exact YAML schema. The aggregation step parses this and produces the final PR comments and handoff brief.

## Schema (YAML)

```yaml
findings:
  - severity: critical | warning | suggestion
    confidence: high | medium | low
    domain: security | correctness | performance | architecture | api-contract | a11y | test-coverage | business-logic
    file: path/to/file.ts
    start_line: 42
    end_line: 47
    issue_header: "Short two-to-four-word title"
    issue_content: |
      One paragraph. What is wrong, why it matters, and the specific
      scenario or input that triggers it. Use backticks for identifiers
      and paths. Do not mention line numbers in this field — they are
      in start_line and end_line.
    reasoning: |
      Step-by-step internal reasoning for why this is an issue. Cite
      specific lines from the diff. This field is for the aggregator
      and the user — it is not posted as a PR comment.
    suggested_fix: |
      Concrete code suggestion or specific guidance. If a code block
      makes sense, include it. If the fix is configuration or
      platform-level, give the exact steps.

review_summary: |
  One-paragraph plain-text overview of what changed in this PR and
  the overall risk profile. Written for the PR summary comment.
  Maximum 150 words.

verdict: approved | approved_with_comments | minor_issues | significant_concerns
verdict_reason: |
  One sentence explaining why this verdict was chosen, referencing
  the approval rubric.
```

If no findings, return `findings: []` and an appropriate `review_summary` / `verdict`.

## Severity definitions

These are shared across all subagents and the inline orchestrator. Apply them consistently.

### critical

The issue, if shipped, will cause one or more of:

- A security exploit (data breach, account takeover, remote code execution)
- Data loss or corruption (destructive DB operation, lost user data)
- Production outage (crash, infinite loop, exhaustion of cloud resources causing billing or availability impact)

Critical findings block merge in the approval rubric. Examples:

- SQL injection where user input is concatenated into a query
- Hardcoded API key or secret in source code
- `DELETE` or `DROP` without `WHERE`
- Webhook handler missing signature verification
- Race condition with confirmed concurrent access

### warning

Real risk under specific conditions. Should be fixed before merge but not strictly blocking. Examples:

- Missing error handling on a network call that would crash the request
- N+1 query in a hot path
- Missing input validation on a user-facing field
- Race condition possible but unlikely
- Breaking API change without version bump

### suggestion

A hardening or improvement opportunity that is not blocking. The author may reasonably choose not to act on it. Examples:

- Edge case the author has clearly considered but not explicitly handled
- A more idiomatic way to write the same logic
- Missing comment explaining a non-obvious choice

Suggestions are posted but do not affect the verdict beyond `approved_with_comments`.

## Confidence definitions

Confidence reflects how certain the reviewer is that the issue is real.

- **high** — The reviewer can point to the exact code, the exact trigger scenario, and explain the impact without speculation. The issue is reproducible from the diff alone.
- **medium** — The reviewer can identify the pattern but the trigger scenario depends on context outside the diff. The issue is likely real but not 100% verified.
- **low** — The reviewer suspects an issue but cannot fully confirm without more context. Speculative.

## Confidence filtering rules (applied by aggregator)

- `confidence: high` → always include in PR comments.
- `confidence: medium` → include in PR comments.
- `confidence: low` AND `severity: critical` → include in PR comments with explicit uncertainty note in `issue_content`.
- `confidence: low` AND `severity != critical` → omit from PR comments. Include in handoff brief under "Possibly worth checking" section.

## Tone rules

These apply to `issue_header`, `issue_content`, and `suggested_fix` fields. The `reasoning` field is internal and exempt.

- Direct, matter-of-fact, helpful.
- No filler phrases: "Great job", "Thanks for", "Looks good overall", "Just a small thing".
- No accusatory language: "You forgot", "You should have", "This is wrong because you".
- Do not overstate impact. If an issue only arises under specific inputs or environments, say so upfront in `issue_content`.
- Use backticks for identifiers, paths, variables, types, and code symbols.
- Keep `issue_content` concise: the reader should grasp the point without rereading.

## Example finding

```yaml
findings:
  - severity: critical
    confidence: high
    domain: security
    file: src/api/admin/users.ts
    start_line: 14
    end_line: 18
    issue_header: "SQL injection in user search"
    issue_content: |
      User-controlled `req.query.name` is concatenated directly into the
      SQL string at line 16. An attacker passing
      `'; DROP TABLE users; --` would execute arbitrary SQL with the
      database role this connection runs as.
    reasoning: |
      The diff adds `query = "SELECT * FROM users WHERE name = '" + req.query.name + "'"`
      with no parameterization, no escaping, and no apparent sanitization
      in the middleware chain. The route lacks any input validation
      decorator visible in the diff or in the context file
      src/middleware/validate.ts.
    suggested_fix: |
      Use a parameterized query:

          const query = "SELECT * FROM users WHERE name = $1";
          const result = await db.query(query, [req.query.name]);

      Or use the project's existing query builder if available.
```
