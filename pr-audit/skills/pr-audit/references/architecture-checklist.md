# Architecture checklist

This checklist defines what the architecture-reviewer subagent looks for. It targets cross-file impacts, public API surface changes, schema migrations, and structural integrity issues that span beyond a single file.

The bar for flagging is medium-high. Issues must have a **concrete cross-file consequence** that can be inferred from the diff and supporting context files. Speculative concerns about future scaling are out of scope.

## What to Flag

### Public API surface changes

- Removed or renamed exports that callers in the diff have not been updated to match.
- Removed or renamed exported types/interfaces that consumers depend on.
- Changes to function signatures (parameter count, parameter types, return type) where call sites in the diff still pass the old shape.
- Newly added required parameters to functions that have existing callers outside the diff (likely breakage).
- Default export changes where consumers import by name expecting the old default.

### Cross-file integration risks

- Imports added that point to non-existent modules (typo, missing file, wrong relative path).
- Circular imports introduced by new dependency edges (file A imports B, B imports A).
- Files that import a module the diff is removing or renaming.
- Re-exports through barrel files (`index.ts`) that no longer match the source they re-export.

### Database schema changes

- Migrations that drop tables or columns without a corresponding `down` migration.
- Migrations that change column types in a way that may fail on existing data (e.g., `text` → `int` without explicit cast or validation).
- New columns added as `NOT NULL` without a `DEFAULT` value on tables with existing rows (will fail).
- Foreign key constraints added on existing tables without verifying referential integrity holds.
- Indexes dropped that may have been load-bearing for queries elsewhere.
- Schema renames where old name is still referenced in code that isn't being updated in the same diff.

### Architectural drift

- New file placed in a directory that doesn't match the project's established layout convention (e.g., new utility in `src/components/` when `src/utils/` exists).
- Logic duplicated from an existing utility that the diff doesn't import (parallel implementations).
- New direct dependency on a low-level module from a high-level layer that the codebase clearly separates.
- Cross-module coupling introduced via a back-channel (global mutable state, monkey-patching, hidden side effects in import).

### Configuration and contracts

- Changes to `package.json` `exports` field that drop public entry points.
- Changes to `tsconfig.json` `paths` mappings that break existing relative imports.
- Renamed environment variables in code without corresponding `.env.example` updates.
- Build configuration changes (Webpack, Vite, Rollup) that change the output structure.

## What NOT to Flag

- Style of refactoring choices when no breakage is evident.
- "This could be better organized" without a concrete contract violation.
- Internal-only restructuring (private helpers, file moves within a single module) that doesn't change exports.
- Newly added internal abstractions that aren't yet over-engineered.
- Generic "consider applying SOLID" or pattern advice without a tied concrete defect.
- Project-layout preferences when no clear convention is visible in the diff or context.

## Common false positives to actively avoid

- Internal helpers renamed where all references are updated in the same diff.
- New file added to a new directory that doesn't match existing layout — only flag if the diff also shows the existing layout convention is contradicted.
- Imports from `node_modules` packages that look unusual — these are user choices, not architectural defects.
- Re-export simplifications (collapsing barrel files) — typically intentional and safe.

## Severity calibration examples

- **critical**: migration drops a column with active production data; removed export of a function called by 50 other files in the codebase; circular import that causes runtime initialization failure; schema change that requires a multi-step migration and the diff has it as one step.
- **warning**: function signature change without all callers updated in diff; migration adds NOT NULL without DEFAULT; renamed component without grep verification of all usages.
- **suggestion**: opportunity to consolidate similar helpers; consider moving this utility to a shared module; barrel file could be simplified.
