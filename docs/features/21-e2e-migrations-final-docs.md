# Feature: E2E Tests, Migration Discipline, and Final Documentation

## Goal

Create the verification and documentation foundation for a living codebase.

## Impacted Areas of the Codebase

- Root verification scripts.
- Prisma migration commands.
- Documentation folders.

## Implementation Approach

The repository includes `npm run verify`, Prisma migration scripts, and final design documentation. Browser tests are represented in the future workflow and can be expanded with Playwright against the seeded app.

## Design Considerations

Migration discipline is partly technical and partly team habit: schemas should change through migrations and seed data should teach workflows.

## Build Journal

Implemented verification scripts and migration entry points. Added final system design documentation.

A junior developer should learn that a codebase is not finished when the first screen renders; maintainability comes from repeatable verification and readable design records.

## As built (checked against this snapshot)

- Root `verify` = `typecheck && test && build` (`package.json`); `db:migrate` runs `prisma migrate dev` in `@lms/api`.
- Tests: ten domain tests (`packages/domain/src/domain.test.ts`) and one API test that only checks `createApp` returns a function (`apps/api/src/app.test.ts`). The worker and web packages pass with no tests or have none.
- There is no `apps/api/prisma/migrations/` folder, no Playwright configuration and no CI workflow in this snapshot.
- `docs/architecture/final-system-design.md` exists and now includes how to run the system, verified findings and exercises.

**Exercise.** *Goal:* add one API test with `supertest` that logs in as the seeded learner against a migrated and seeded database. *Check:* `npm test` shows the test passing, and it fails if you change the seeded password.
