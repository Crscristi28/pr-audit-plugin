# Business logic checklist

This checklist defines what the business-logic-reviewer subagent looks for. It validates the PR diff against custom business rules defined by the user in `.pr-audit/rules.md` under a `## Business Logic` section.

Unlike the other reviewers, this subagent has **no built-in catalog of issues to flag**. It works exclusively from the user-provided rules in `.pr-audit/rules.md`. If no rules are defined, this reviewer is not spawned.

## Spawn condition

The orchestrator spawns this reviewer ONLY when:

1. `.pr-audit/rules.md` exists in the repository root, AND
2. That file contains a section with the heading `## Business Logic` (case-sensitive).

If either condition is false, do not spawn this reviewer. The diff is not reviewed by this domain.

## Input

The reviewer receives:

1. The diff scope (filtered diff from Phase 4).
2. Context files explicitly passed by the orchestrator.
3. The full text of the `## Business Logic` section from `.pr-audit/rules.md`.
4. This checklist (defining the reviewer's behavior).

## What to Flag

Flag any code in the diff that violates a stated rule from the `## Business Logic` section. Examples of typical business rules the user may write:

- "All API endpoints handling user data must include a `requireAuth` middleware reference."
- "Invoices must always include `tax_id` when `customer.region === 'EU'`."
- "Refund amounts cannot exceed the original payment amount; check by querying `payments` table first."
- "Newly created subscription plans must have `trial_days >= 14`."
- "Logging of credit card data is forbidden, even truncated."
- "All admin actions must record an entry in `audit_log` table within the same transaction."

For each rule violation found:

1. Quote the specific rule from the user's `.pr-audit/rules.md`.
2. Identify the line(s) in the diff that violate it.
3. Explain why the diff violates the rule.
4. Provide a concrete fix that satisfies the rule.

## What NOT to Flag

- Anything not stated in the `## Business Logic` section. This reviewer does NOT add its own opinions about business correctness.
- Rules from other sections of `.pr-audit/rules.md` (those are for other reviewers — security, performance, etc.).
- General code quality concerns — those are the correctness reviewer's job.
- Style or naming preferences not tied to a stated rule.
- Stylistic interpretations of vague rules. If the rule is ambiguous, flag with `confidence: low` and explain the ambiguity, but err on the side of not flagging.

## Edge cases

### Rule references code outside the diff

If a rule like "All admin actions must record an entry in `audit_log`" requires examining `audit_log` write call sites, but those calls are not in the diff, you can only flag the new admin action if there is no visible call to the audit log within the new code. Trust that the user's rule implies a specific code pattern.

### Rule is vague or non-mechanical

If a rule says something like "Code should be clean" or "Follow good business practices" — do not flag anything. These are not actionable rules. Note in the output that the rule is too vague to verify mechanically.

### Multiple rules apply to the same code

A single line may violate multiple business rules. Emit one finding per rule violation (separate `findings` entries), so each can be addressed independently.

### Rules contradict the security or correctness reviewers

If a business rule says something like "log everything for audit" but the security reviewer flagged that as data exposure — flag both findings independently. The user resolves the conflict at review time.

## Severity calibration

The user implicitly sets severity through their rule wording:

- Rules using "must", "required", "always", "never" → **critical** or **warning** depending on impact.
- Rules using "should", "prefer" → **suggestion**.
- If rule wording is neutral ("All API endpoints include..."), default to **warning** unless the violation has clear data/compliance impact (then **critical**).

## Severity examples

- **critical**: rule says "Logging credit card numbers is forbidden" and diff adds `logger.info(\`Charging ${card.number}\`)` — direct PCI violation.
- **warning**: rule says "Admin actions must record in audit_log" and new admin action in diff has no `audit_log.create` call.
- **suggestion**: rule says "Prefer named exports over default exports" and diff adds a new default export.

## Output

Always set `domain: business-logic`. In the `reasoning` field, quote the specific rule from `.pr-audit/rules.md` that was violated and explain how the diff violates it.

If no violations are found, return `findings: []`.
