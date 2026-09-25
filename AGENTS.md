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
- Suwayomi-Server `v2.3.2243` still uses `Int` for manga, chapter, and category IDs and `LongString` for source IDs. Its schema does not define `ID`.
- The remote `v3-GraphQL-API` branch is older work with a different directory layout and should not be treated as a ready-made compatibility fix.

## Critical Compatibility Constraint

GraphQL validates declared variable types before executing an operation. Match the server schema exactly:

- Manga, chapter, and category identifiers use `Int!` where required.
- Source identifiers use `LongString!`; the scalar accepts string variable values so JavaScript does not lose 64-bit precision.
- Do not change these declarations to `ID!`. Suwayomi-Server `v2.3.2243` rejects that document with `Unknown type ID`.

## Identifier Rules

- Suwayomi returns manga, chapter, and category IDs as 32-bit numbers. Paperback exposes manga and chapter IDs to this extension as strings.
- Convert Paperback-facing values with `parseGraphQLInt`, which rejects malformed and out-of-range values before JSON serialization. Do not use permissive `parseInt` for GraphQL IDs because values such as `123abc` would be silently truncated and `NaN` would serialize as `null`.
- Keep source IDs as strings when passing them to `LongString!`; JavaScript numbers cannot safely represent every Kotlin `Long`.
- Keep category IDs as strings in persisted settings, then validate them with `parseGraphQLInt` when constructing a GraphQL request.
- Page numbers, offsets, result limits, chapter numbers, page counts, timestamps, and chapter `sourceOrder` remain numeric.

Paperback currently exposes a chapter's `sourceOrder` as its chapter ID. Code that fetches pages or updates read status must first resolve `(mangaId, sourceOrder)` to the backend chapter database ID, then pass that database ID to the mutation.

## GraphQL Change Checklist

When modifying operations in `src/TachiDesk/`:

1. Inspect every inline query and mutation, including manga details, chapter lookup, page fetching, categories, source browsing, search, and tracker updates.
2. Confirm variable types against each supported Suwayomi schema version.
3. Keep both `GQL_GET_SOURCE_MANGAS` and `GQL_SEARCH_SOURCE` as mutations using `fetchSourceManga`.
4. Keep TypeScript response models aligned with the server schema and convert IDs to strings only at Paperback or persisted-settings boundaries.
5. Validate every Paperback-facing manga, chapter, or category ID with `parseGraphQLInt` before using it in an `Int` variable.
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

There are no schema-contract tests in the repository. A successful bundle proves compilation, not compatibility with a live Suwayomi server. GraphQL changes should be tested against the oldest supported server and the target v2.3 server before release.

## Scope Discipline

- Preserve the existing Paperback 0.8 APIs and project style unless an upgrade is intentional.
- Avoid committing generated bundle or homepage output unless the release workflow requires it.
- Do not combine schema compatibility work with unrelated dependency upgrades or broad refactors.
- State the supported Suwayomi version range in a PR and release notes whenever GraphQL variable types change.