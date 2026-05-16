# Correctness checklist

This checklist defines what the correctness-reviewer subagent looks for. It targets logic bugs, error handling gaps, edge cases, concurrency issues, and behavioral regressions.

## What to Flag

### Null, undefined, and missing values

- Accessing properties or methods on a value that the diff shows can be `null`, `undefined`, or missing — without an optional chain, guard, or default.
- Destructuring a value that may be `undefined` without a default (`const { x } = maybeUndefined`).
- Array access by index without checking length (`arr[5]` where the diff doesn't establish length ≥ 6).
- `JSON.parse` of network responses, localStorage, cookies, or file content without try/catch.

### Error handling

- Promise chains without `.catch` in code paths that reach the user. Unhandled rejections become silent bugs or app crashes.
- `await` calls in critical paths without try/catch where the error needs to be handled differently from a generic 500.
- Swallowed errors: `catch (e) {}` with no logging, re-throw, or user-facing handling.
- HTTP fetches that do not check `response.ok` before parsing the body.
- Database transactions that do not roll back on error.

### Off-by-one and range errors

- Loop conditions using `<` where `<=` is needed (or vice versa) and the diff shows the intent.
- Slicing or substring operations with indices that may exceed length.
- Pagination logic with `LIMIT` and `OFFSET` that double-counts boundary rows.

### Race conditions and concurrency

- Read-modify-write patterns on shared state without locking or atomic operations: `count = await get(); await set(count + 1);`.
- `useState` updates that depend on the previous state but use the value form instead of the functional form: `setCount(count + 1)` inside an event handler that may fire multiple times.
- File operations without locking when concurrent writers are possible.
- Optimistic UI updates without a reconciliation step if the server rejects.

### State mutations

- Direct mutation of state in frameworks that require immutability (React, Redux, Zustand without immer): `state.items.push(x)` instead of `setState({ ...state, items: [...state.items, x] })`.
- `Array.sort()`, `Array.reverse()`, `Array.splice()` called on props or state without copying first.
- `Object.assign(target, ...)` where `target` is a reference shared elsewhere.

### Date, time, and locale

- `new Date(stringFromUser)` without checking `!isNaN(date.getTime())`.
- Date comparisons that mix local time and UTC: `date.getDate()` (local) compared to a UTC timestamp.
- Hardcoded timezones (`'America/New_York'`) in user-facing logic that should respect the user's locale.
- Currency formatting without explicit locale or currency code.
- `Date.now()` used as a "unique enough" ID (collisions occur within the same millisecond under load).

### API and data contracts

- Function signature changes that drop or rename required parameters where call sites in the diff still pass the old shape.
- Return type changes that callers in the diff have not been updated to handle.
- Removing fields from API response bodies that callers depend on.
- Renaming database columns referenced by code that has not been updated.
- Migration that changes column type in a way that may fail on existing data (e.g., `text` → `int` without verifying all values are numeric).

### Logic errors specific to the language / framework

- React hooks called conditionally or inside loops.
- React `useEffect` missing dependencies that the body uses.
- React `useEffect` with no dependency array that runs on every render and produces side effects.
- Python `default=[]` or `default={}` in function arguments (shared mutable default).
- Go: returning a pointer to a stack-local that escapes the function.
- TypeScript: `as` casts that bypass type errors the type system was correctly catching.

### Missing edge cases

- Empty input not handled: empty array, empty string, empty object where the code assumes at least one element.
- Single-element input where the code assumes pairs.
- Zero or negative input where the code assumes positive.
- Unicode in inputs where the code uses `.length` or byte-level operations and the user is asking for character count.

### Boolean logic

- `if (x = y)` (assignment instead of comparison).
- `==` vs `===` in JavaScript where loose equality changes behavior (`0 == ""`, `null == undefined`).
- Negation errors: `!x.length` when `x` may be undefined (throws); should be `!x?.length`.
- De Morgan's law applied incorrectly during refactor.

## What NOT to Flag

- Style preferences: arrow functions vs function expressions, `let` vs `const`, naming.
- Generic "add more tests" — only flag if the diff visibly introduces a behavior change that is testable and untested in the same diff (and even then, the test-coverage-reviewer handles this in future versions).
- Performance concerns — that's the performance-reviewer's job. Flag a perf issue only if it is also a correctness bug (e.g., infinite loop).
- Documentation suggestions — out of scope.
- Refactoring opportunities that do not fix a bug.
- Generic "consider using async/await" when the existing promise chain is correct.
- Type narrowing suggestions when the code already works correctly.

## Common false positives to actively avoid

- `let x: SomeType = ...` where the reviewer infers the type "should" be `const` — `let` vs `const` is style.
- `const x = await foo()` in an async function that returns the value — the `await` is redundant but harmless.
- `Array.isArray(x)` before iterating — defensive but not a bug.
- `if (!user) return;` early-return patterns — not a finding.
- Calling `String(x)` or `Number(x)` for explicit conversion — not a bug.

## Severity calibration examples

- **critical**: production crash on a common code path (NPE on user load); race condition that causes lost writes; infinite loop that exhausts resources; `SELECT *` query returned to the client that includes another user's data.
- **warning**: missing `.catch` on a network call that will surface as an unhandled promise rejection; off-by-one in pagination causing duplicate rows; missing null check on an optional config field that may be unset in production.
- **suggestion**: prefer functional `setState` form for safer concurrency; consider adding explicit error handling here even though the framework's default handler works.
