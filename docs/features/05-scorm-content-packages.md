# Feature: SCORM-Like Content Packages

## Goal

Represent imported structured training packages without fully implementing SCORM runtime behavior.

## Impacted Areas of the Codebase

- Prisma model: `ContentPackage`.
- Course version detail include.

## Implementation Approach

The model stores a package name and manifest JSON under a course version.

## Design Considerations

The realistic boundary is storing the manifest and mapping it into lessons later. Full SCORM sequencing is intentionally out of scope.

## Build Journal

Implemented content package storage in the schema. It connects to course versions so imported package content can vary by published version.

A junior developer should learn to create a durable boundary for a complex external standard before implementing every edge case.

## As built (checked against this snapshot)

- `ContentPackage` (`apps/api/prisma/schema.prisma:421-428`): `courseVersionId`, `name`, `manifestJson`, `importedAt`.
- It is included in `GET /catalog/courses/:courseId` (`apps/api/src/routes/catalog.routes.ts:81`).

**Gaps.** No route imports a package and the seed creates none, so the include always returns an empty list. There is no mapping from manifest to lessons. The boundary described above exists only as a table.

**Exercise.** *Goal:* add `POST /authoring/versions/:versionId/content-packages` that accepts `{ name, manifestJson }`, validated with Zod (require at least an `items` array). *Check:* after posting a package, the catalog detail for that course shows it under the right version, and a body without `items` returns 400 (after you also map `ZodError` to 400 in `apps/api/src/http.ts`).
