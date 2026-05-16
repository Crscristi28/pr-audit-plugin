# Test coverage checklist

This checklist defines what the test-coverage-reviewer subagent looks for. It targets meaningful test coverage gaps in the PR diff — not raw line coverage metrics.

The bar for flagging is medium. Flag missing tests only when the diff introduces **non-trivial logic** that can fail in identifiable ways, and the diff does not include a corresponding test.

## What to Flag

### Missing tests for new logic

- New function with conditional branches (`if`, `switch`, ternary chains) and no matching test file or test case in the diff.
- New API endpoint, route handler, or controller method without an integration test exercising at least the happy path.
- New error handling branch (`catch`, `try`, error-returning condition) without a test for the error case.
- New form validation logic without a test for the failure case.
- New reducer or state machine transition without a test asserting the resulting state.
- New algorithm or transformation with non-trivial output (parsing, formatting, calculation) without an assertion test.

### Test sibling expectations

- New file `src/foo/bar.ts` containing exported logic without a matching `src/foo/bar.test.ts`, `src/foo/__tests__/bar.test.ts`, or equivalent per project convention.
- Modified function where the diff adds new conditional paths that the existing tests (visible in the diff or referenced context files) don't cover.

### Test quality regressions

- Tests added that only assert `expect(true).toBe(true)` or equivalent placeholder.
- Tests that mock the function under test (instead of testing real behavior).
- Tests that skip critical assertions with `// TODO: assert later` comments.
- Tests that are disabled via `xit`, `it.skip`, `describe.skip` without an explanatory comment.

### Coverage of edge cases for changed logic

- New input validation that handles only the happy path — no test for malformed input.
- New pagination/limit logic without test for empty result set, single-item, or max-limit cases.
- New date/timezone handling without test crossing a daylight saving boundary or timezone offset.
- New currency/decimal math without test for rounding edge cases.

### Integration boundaries

- New external API client (HTTP, database, queue) without a test that mocks the boundary and asserts the request shape and response handling.
- New error retry logic without a test that simulates failure and verifies retry behavior.

## What NOT to Flag

- Files matching `*.test.*`, `*.spec.*`, `__tests__/`, `tests/`, `cypress/`, `playwright/`, `e2e/` — these are tests themselves.
- Trivial getter functions, type definitions, constants, configuration files.
- Pure rename or refactor where logic is unchanged and existing tests still pass.
- Storybook stories, mocks, fixtures, seed scripts.
- Generated code (already excluded by noise filters but as defense).
- Documentation files, README updates, CHANGELOG updates.
- One-line CSS or styling changes.
- "100% coverage" — never flag for raw coverage percentage. Only flag specific identifiable gaps.

## Common false positives to actively avoid

- Test file exists but in a non-conventional location (some projects put tests next to source, some in `__tests__/`, some in a top-level `tests/`). Look for any test file referencing the new code before flagging.
- New utility added that is exported but the diff also shows its first caller — that caller's test may cover the utility transitively.
- Component changes where the only logic is JSX rendering — Storybook stories or screenshot tests may exist in context files not shown.
- Type-only changes (TypeScript interfaces, enums, type aliases) — types are validated by the compiler, not tests.
- Files that are purely re-exports from other tested modules.

## Detection heuristics

When the orchestrator decides whether to spawn this reviewer, it checks:

1. Does the diff contain source files (non-test, non-doc, non-config)?
2. For each source file: is there a sibling test file in the diff (e.g., `foo.ts` + `foo.test.ts`)?
3. If non-trivial source files exist without test siblings → spawn this reviewer.

When this reviewer evaluates the diff, it also examines context files the orchestrator passed in (typically the most likely test file locations) before flagging.

## Severity calibration examples

- **critical**: new payment processing logic in `src/api/charge.ts` with no test file in the diff or context — payment bugs cause direct financial impact.
- **warning**: new authentication helper with branching logic, no test in diff; new reducer action handler with no test asserting state change.
- **suggestion**: consider adding a test for the error path here; this validation rule could benefit from a malformed-input test case.
