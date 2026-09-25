# Maintainer Guide

## Project Purpose

This repository contains a Paperback extension that lets the iOS app browse manga through a self-hosted Suwayomi-Server (formerly Tachidesk). The extension translates Paperback source APIs into Suwayomi GraphQL requests.

The active implementation is under `src/TachiDesk/`:

- `TachiDesk.ts` contains Paperback entry points and inline GraphQL operations.
- `Common.ts` contains request helpers, persisted settings, response models, source/category loading, and server connection logic.
- `Settings.ts` contains the extension settings UI.

## Release And Schema Baseline

- The latest tagged extension release is `0.8-v2` at commit `b882c62`.
- Commit `5209e3e` changed `fetchSourceManga` from a query to a mutation. Keep that behavior for browsing and search.
- The `master` implementation after that fix was verified against Suwayomi-Server `v2.2.2100`.
- Suwayomi-Server `v2.3.2243` changed database-backed GraphQL identifiers from numeric scalars to string-based `ID` scalars.
- The remote `v3-GraphQL-API` branch is older work with a different directory layout. It does not implement the Suwayomi `v2.3.x` ID migration and should not be treated as a ready-made fix.

## Critical Compatibility Constraint

Changing an operation variable from `Int!` or `LongString!` to `ID!` is not backward compatible at the GraphQL document level. GraphQL validates the declared variable type against the server argument type before executing the operation:

- A client variable declared as `ID!` cannot be used where a v2.2 schema expects `Int!`.
- A client variable declared as `ID!` cannot be used where a v2.2 schema expects `LongString!`.
- The fact that the GraphQL `ID` scalar can coerce integer input does not make differently declared variable types interchangeable.

Do not submit a direct `Int!`/`LongString!` to `ID!` replacement as a generally compatible fix. It requires an explicit minimum supported Suwayomi version or a dual-schema implementation.

For dual support, keep separate legacy and v2.3 operation variants and select the correct set from a reliable server capability or version check. Prefer `aboutServer.version` or schema introspection over matching error text. Normalize returned entity IDs with `String(value)` at the response boundary.

## Identifier Rules

Treat these as opaque strings in Paperback and TypeScript models:

- Manga IDs
- Chapter database IDs
- Category IDs when using a v2.3 schema
- Source IDs

Do not call `parseInt` on an opaque entity ID. Numeric-looking IDs may exceed JavaScript's safe integer range, and future IDs may not be numeric.

These values remain numeric and must not be converted to `ID` merely because their names are related to chapters or lists:

- Chapter `sourceOrder`
- Page number, offset, and result limit variables
- Chapter number and page count
- Timestamps used for date or staleness calculations

Paperback currently exposes a chapter's `sourceOrder` as its chapter ID. Code that fetches pages or updates read status must first resolve `(mangaId, sourceOrder)` to the backend chapter database ID, then pass that database ID to the mutation.

## GraphQL Change Checklist

When modifying operations in `src/TachiDesk/`:

1. Inspect every inline query and mutation, including manga details, chapter lookup, page fetching, categories, source browsing, search, and tracker updates.
2. Confirm variable types against each supported Suwayomi schema version.
3. Keep both `GQL_GET_SOURCE_MANGAS` and `GQL_SEARCH_SOURCE` as mutations using `fetchSourceManga`.
4. Update TypeScript response models and persisted mappings when a scalar changes.
5. Convert IDs to strings at API boundaries; do not convert them back to numbers at call sites.
6. Test manga browsing, source search, chapter loading, page loading, category sections, and read-progress updates.

## Build And Validation

Install exactly the locked dependencies and build from the repository root:

```bash
npm ci
npm run build
```

Run a standalone type check with:

```bash
npx tsc --noEmit
```

The pinned `@paperback/types` package publishes raw TypeScript containing an unused import. Keep strict type checking enabled, but project-wide `noUnusedLocals` may need to remain disabled until that dependency is upgraded.

There are no schema-contract tests in the repository. A successful bundle proves compilation, not compatibility with a live Suwayomi server. Schema migrations should be tested against both the oldest supported server and the target v2.3 server before release.

## Scope Discipline

- Preserve the existing Paperback 0.8 APIs and project style unless an upgrade is intentional.
- Avoid committing generated bundle or homepage output unless the release workflow requires it.
- Do not combine schema compatibility work with unrelated dependency upgrades or broad refactors.
- State the supported Suwayomi version range in a PR and release notes whenever GraphQL variable types change.