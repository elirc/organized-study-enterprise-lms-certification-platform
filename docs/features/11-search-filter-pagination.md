# Feature: Course Search, Filtering, and Pagination

## Goal

Make the catalog usable for growing course libraries.

## Impacted Areas of the Codebase

- Catalog route: `GET /catalog/courses`.
- Catalog page controls.

## Implementation Approach

The API accepts `q`, `status`, `page`, and `pageSize`, returning pagination metadata.

## Design Considerations

Search is intentionally simple at first. PostgreSQL full-text search or an external index could be added later.

## Build Journal

Implemented query parameters and catalog UI controls.

A junior developer should learn to include pagination before tables become large.

## As built (checked against this snapshot)

- `GET /catalog/courses` (`apps/api/src/routes/catalog.routes.ts:18-70`) validates `page` (default 1), `pageSize` (default 10, max 50), `q` and `status` with Zod, searches title and summary case-insensitively, and returns `{ courses, pagination: { page, pageSize, total } }`.

**Gaps.** Any signed-in user may pass any organization ID in the header and any status, including `DRAFT` (`:15`, `:22-29`); there is no membership check on this router. Each course in the list includes its latest version with all modules and lessons — heavy for a list. The catalog page itself renders demo data (`apps/web/app/catalog/page.tsx:29`). Invalid query values throw a `ZodError`, which the error handler returns as 500.

**Exercise.** *Goal:* require membership in the header's organization (use `requireOrganizationRole` with all four roles) and restrict non-instructors to `PUBLISHED`. *Check:* the learner gets 403 for `?status=DRAFT` and 200 without it; a user from another organization gets 403.
