# Path patterns

The orchestrator classifies file paths using these patterns to drive tier decisions and subagent selection.

Apply each pattern set as a regular expression against the relative file path (forward slashes). A file may match multiple categories.

## Security-sensitive paths

Files that handle authentication, authorization, secrets, middleware, environment configuration, or database schema. These trigger the security-sensitive flag in tier decisions, which forces at least Standard tier even for tiny diffs.

```
^src/auth/.*
^src/middleware/.*
^src/api/.*
^app/api/.*
^pages/api/.*
^routes/.*
^lib/auth/.*
^lib/security/.*
^server/.*
\.env(\.|$).*
^prisma/.*
^migrations/.*
^drizzle/.*
.*\.sql$
^supabase/.*
^firebase\.json$
^firestore\.rules$
^.*[Ss]ecret.*\.(ts|tsx|js|jsx|py|go|rs|rb)$
```

## API-surface paths

Files that define HTTP routes, GraphQL schemas, protobuf definitions, or OpenAPI specs. Changes here may break consumers.

```
^routes/.*
^src/api/.*
^app/api/.*
^pages/api/.*
^src/routes/.*
.*\.graphql$
.*\.gql$
.*\.proto$
openapi\.(yaml|yml|json)$
swagger\.(yaml|yml|json)$
^api/.*\.(yaml|yml|json|ts|tsx|js|jsx|py)$
.*[Ss]chema.*\.(ts|tsx|js|jsx|py)$
```

## Frontend paths

UI components and templates. Relevant for accessibility and UI breakage review (Tier 2, not yet implemented in v0.1).

```
^src/components/.*
^src/pages/.*
^pages/.*
^app/.*\.(tsx|jsx)$
^components/.*
.*\.vue$
.*\.svelte$
.*\.astro$
.*\.html$
.*\.tsx$
.*\.jsx$
```

## DB-schema paths

Files that define database schema or run migrations. Always Deep tier — schema changes are high impact.

```
^prisma/schema\.prisma$
^drizzle/.*
^migrations/.*
^db/migrations/.*
^supabase/migrations/.*
^server/db/schema\..*
.*\.sql$
```

## Test paths (informational)

Files matching these patterns are excluded from many checks because they are test infrastructure, not production code.

```
^tests?/.*
^__tests__/.*
.*\.test\.(ts|tsx|js|jsx|py|go|rs|rb)$
.*\.spec\.(ts|tsx|js|jsx|py|go|rs|rb)$
^spec/.*
^cypress/.*
^playwright/.*
^e2e/.*
```

Test paths are still reviewed for correctness issues that would produce false positives in production review (e.g., hardcoded test credentials are fine in tests, would be flagged in production).

## Doc paths (informational)

Documentation-only changes can skip code review entirely (see `tier-decision.md` edge case).

```
\.md$
\.mdx$
\.txt$
\.rst$
^LICENSE.*$
^docs/.*
^doc/.*
\.gitignore$
\.editorconfig$
\.prettierrc.*$
\.eslintrc.*$
```

## Build / generated paths (always filtered by noise-filters)

Listed for reference but handled by `noise-filters.md`:

```
^dist/.*
^build/.*
^out/.*
^\.next/.*
^node_modules/.*
.*\.min\.js$
.*\.min\.css$
.*\.map$
```

## Stack hints (informational, not used for tier decision)

These hints can be used by future Tier 2 subagents to load stack-specific checklists. Detection is based on the presence of these files anywhere in the diff or repo root:

| Pattern | Stack hint |
|---------|------------|
| `^next\.config\..*` | Next.js |
| `^vite\.config\..*` | Vite |
| `^remix\.config\..*` | Remix |
| `^astro\.config\..*` | Astro |
| `^svelte\.config\..*` | SvelteKit / Svelte |
| `^vercel\.json$` | Vercel deployment |
| `^netlify\.toml$` | Netlify deployment |
| `^wrangler\.toml$` | Cloudflare Workers |
| `^firebase\.json$` | Firebase |
| `^supabase/config\.toml$` | Supabase |
| `^Dockerfile$` | Container deployment |

## Quick reference table

| Category | Forces Deep tier | Flag passed to subagents |
|----------|------------------|--------------------------|
| security-sensitive | When lines > 50 | `path_flags.security_sensitive = true` |
| api-surface | When lines > 50 | `path_flags.api_surface_changed = true` |
| db-schema | Always | `path_flags.db_schema_changed = true` |
| frontend | No (Tier 2 trigger) | `path_flags.frontend_changed = true` |
| test | No | `path_flags.test_changed = true` |
| doc | No (may skip review) | `path_flags.doc_only_changes = true` |
