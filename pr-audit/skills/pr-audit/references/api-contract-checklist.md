# API contract checklist

This checklist defines what the api-contract-reviewer subagent looks for. It targets breaking changes to API surfaces — REST endpoints, GraphQL schemas, protobuf definitions, OpenAPI specs — that may break external or internal consumers.

The bar for flagging is high. Each finding must identify a **specific contract violation** with a concrete consumer impact.

## What to Flag

### REST API breaking changes

- Removed routes that consumers may depend on.
- Renamed routes without a backward-compatible alias.
- Changed HTTP methods on existing routes (e.g., `POST` → `PUT`).
- Removed query parameters that callers may pass.
- Added required query parameters or body fields to routes that previously did not require them.
- Removed response fields from endpoints with active consumers.
- Renamed response fields without a transition period.
- Changed response field types (e.g., `string` → `number`, single object → array).
- Status code changes for the same logical outcome (e.g., success was `200`, now `204`).
- Removed pagination parameters or changed pagination semantics.

### GraphQL schema changes

- Removed types, fields, queries, mutations, or subscriptions that consumers may reference.
- Renamed fields without a deprecation cycle (no `@deprecated` directive).
- Changed field nullability from optional to required (breaks queries that don't fetch that field).
- Changed argument types or made arguments required.
- Removed enum values (queries comparing against removed value will fail).
- Renamed enum values without migration path.
- Changed input type structure where mutations rely on the old shape.

### Protobuf / gRPC changes

- Removed message fields (breaks deserialization for older clients).
- Reassigned field numbers (silent data corruption).
- Changed field types (especially scalar → message or vice versa).
- Removed RPC methods.
- Removed enum values.

### OpenAPI / Swagger spec changes

- Spec changes that contradict the actual implementation in the same diff.
- New required request properties without a default-on transition.
- Removed paths declared in the spec but still served by implementation (or vice versa).
- Auth requirement changes (route was public, now requires auth — flag if consumers may not have credentials yet).

### Versioning hygiene

- Breaking changes to a route under a `/v1/` prefix without bumping to `/v2/`.
- Breaking changes to a `1.x.y` versioned SDK without major version bump.
- API documentation that contradicts the actual route handler.

### Webhook contract changes

- Changes to webhook payload structure that downstream consumers parse.
- New webhook event types that consumers may not handle gracefully.

## What NOT to Flag

- Internal helper functions or private modules — these are not API contracts.
- Implementation refactoring that does not change the external response shape.
- Performance optimizations that preserve the contract.
- New optional response fields (additive, generally safe).
- New optional query parameters (additive).
- New routes added without removing existing ones.
- Documentation typos.
- Generic "consider versioning" advice without a specific breaking change.

## Common false positives to actively avoid

- Removing fields from `__tests__` fixtures or seed data — not part of the API contract.
- Internal type renames where the type isn't exported to consumers.
- Response field reordering — most consumers parse by key, not position.
- Adding optional new variants to an enum (typically additive).

## Severity calibration examples

- **critical**: removed `GET /api/users/:id` endpoint with mobile app consumers; protobuf field number reassigned; renamed `email` to `email_address` in response of public REST API without deprecation; removed required `auth` from previously public endpoint that mobile apps may not handle yet.
- **warning**: response field type changed from `string` to `number` (consumers may parse as string); added new required query param to existing route; route moved from `/api/v1/` to `/api/v2/` without keeping v1 alias.
- **suggestion**: deprecate this field instead of removing in this PR; document this contract change in CHANGELOG; consider adding a transition period for this rename.
