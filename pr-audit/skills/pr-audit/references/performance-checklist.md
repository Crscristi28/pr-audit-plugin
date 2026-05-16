# Performance checklist

This checklist defines what the performance-reviewer subagent looks for. It targets meaningful performance impact — not micro-optimizations or theoretical concerns.

The bar for flagging is high. A performance issue must have a **concrete trigger scenario** (a specific input, scale, or call frequency) that produces a measurable problem.

## What to Flag

### Database query patterns

- **N+1 queries**: a loop over results where each iteration triggers another database query. The diff shows the loop and the per-iteration query side by side.
- Missing or unused database indexes for queries on large tables, when the diff shows the query and no index hint or schema change.
- `SELECT *` on a table when the diff shows the consumer only uses two or three columns.
- Queries inside React render functions, useEffect with no dependencies, or any hot path that triggers re-renders.
- ORM methods that produce surprising query patterns (`include` deep nesting in Prisma, `joinedload` chains in SQLAlchemy that generate one query per association).

### Blocking operations in async paths

- Synchronous file IO (`readFileSync`, `writeFileSync`) inside request handlers or React render.
- CPU-bound work on the main thread (large JSON parse, sync hashing, sync image processing) in a request handler.
- Awaiting a serial chain of independent network calls that could run in parallel.

### Unbounded operations

- Loops with no upper bound that depend on user-provided counts (`for (let i = 0; i < req.body.count; i++)`).
- Recursion without depth check on user-controlled structure.
- `Array.fill()`, `new Array(n)`, or other operations with a size derived from user input.
- Polling intervals less than 1 second without a clear reason.

### Memory issues

- Building strings or arrays inside a loop with no upper bound.
- Streaming responses loaded entirely into memory before processing.
- Closures that retain references to large objects beyond their useful lifetime (subtle, flag only with high confidence).
- React component arrays without keys (forces full re-render of all children).

### Caching and re-computation

- Expensive computation re-run on every render in React without `useMemo` or `useCallback` when the dependencies clearly don't change.
- Repeated synchronous filesystem reads of the same file in a request lifecycle.
- API calls inside `useEffect` without proper dependency array, causing re-fetch on every render.

### Network and I/O

- Sequential `await` calls that have no data dependency on each other (`const a = await fetchA(); const b = await fetchB();` where neither uses the other) — should be `Promise.all`.
- Missing connection pooling for database calls in a server.
- Re-establishing a new connection per request when a long-lived client should be reused.

### Frontend-specific

- Images served at full resolution where a thumbnail would suffice.
- Bundled imports of large libraries (`import * as moment from 'moment'`) when a tree-shakeable alternative exists in the codebase.
- Inline `style={{...}}` objects in render functions creating new object references each render and causing memo busts.
- Missing `loading="lazy"` on below-the-fold images, **only if** the diff context shows the image is below the fold and the project uses lazy loading elsewhere.

## What NOT to Flag

- Micro-optimizations: `for` vs `forEach`, `+` vs template literals, `for...of` vs `for...in`.
- Style choices: `const` vs `let`, named functions vs anonymous.
- Theoretical scaling concerns at sizes the diff does not reach ("if you have 10 million users this won't scale" — not actionable for a 50-line change).
- Performance suggestions for code that is clearly cold-path (one-time setup, CLI scripts, test code).
- Generic "consider using a CDN", "consider Redis caching", or other infrastructure suggestions that are out of scope for a code review.
- Suggestions that contradict the project's clear pattern (e.g., suggesting React Query when the project uses SWR consistently).
- Speculative concerns about garbage collection pressure without a clear hot path.

## Common false positives to actively avoid

- `Array.map` followed by `.filter` followed by `.reduce` — these are idiomatic chains, not performance issues unless the array is very large in a hot path.
- `JSON.stringify` for logging — not a perf issue at typical sizes.
- React functional components re-rendering "too often" — most components re-render fine, only flag when there's a clear unnecessary work pattern.
- Database queries in `getServerSideProps` or `loader` functions — these are designed to query per-request; flag only if the query itself is wrong, not the location.

## Severity calibration examples

- **critical**: N+1 query in a route called on every page load against a table with millions of rows; loop calling a paid API (OpenAI, Twilio) with no upper bound; synchronous CPU-bound work blocking the event loop in a Node.js server.
- **warning**: missing `Promise.all` causing serial 200ms requests that could parallelize; missing `useMemo` on an expensive computation in a frequently rendered component; SELECT * returning 50 columns when only 2 are used.
- **suggestion**: opportunity to use index `LIKE 'prefix%'` instead of `LIKE '%prefix%'`; consider streaming this CSV instead of loading it fully.

## Special note on AI / LLM calls

LLM API calls (OpenAI, Anthropic, Google) deserve specific attention because they are expensive (per-token cost) and slow (multi-second latency).

Always flag as **critical** or **warning**:

- LLM call inside a loop without an iteration cap.
- LLM call without `max_tokens` set.
- LLM call without a timeout (the network default is too long for user-facing requests).
- LLM call in a request handler that does not stream the response when the user is waiting on the result.
- LLM call on every keystroke without debouncing.
- Embedding generation in a loop without batching.

These are not just performance issues — they directly translate to user-visible latency and cloud bills, both of which matter to the kind of developers using this plugin.
