# Feature: Bulk Learner Import and CSV Exports

## Goal

Support common admin operations for enterprise training teams.

## Impacted Areas of the Codebase

- Admin routes: `POST /admin/bulk-import`, `POST /admin/exports`.
- Prisma model: `CsvExport`.
- Worker/outbox boundary.

## Implementation Approach

Bulk import upserts learners and memberships. CSV exports create queued records and domain events.

## Design Considerations

Large imports and exports should eventually become background jobs with downloadable artifacts.

## Build Journal

Implemented learner import route, export request route, and dashboard controls.

A junior developer should learn that admin workflows often need audit logs and async processing.

## As built (checked against this snapshot)

- `POST /admin/bulk-import` (`apps/api/src/routes/admin.routes.ts:106-143`): for each `{ email, name }`, upsert the user; new users get a `LEARNER` membership and the password `Password123!`; one audit row with the count.
- `POST /admin/exports` (`:145-168`): creates a `CsvExport` row with status `QUEUED` and records a `csv_export.requested` domain event plus outbox message; returns 202.

**Gaps.** Existing users are **not** added to the organization (the `update` branch only changes `name`), yet are returned as imported — and an org admin can rename a user who belongs only to another organization. Every imported learner shares one known password with no forced reset. Nothing processes `CsvExport` rows; the outbox message is never dispatched, so exports stay `QUEUED` forever.

**Exercise.** *Goal:* in the update branch, upsert a membership for this organization, and stop changing `name` for users who are not yet members. *Check:* importing `instructor@northwind.test` into a second organization creates a second membership and leaves the user's name unchanged.
