# Feature: Learner Transcript

## Goal

Show a learner's course history, completion status, and certificates.

## Impacted Areas of the Codebase

- Domain helper: `buildTranscriptSummary`.
- Admin route: `GET /admin/learners/:learnerId/transcript`.
- Prisma model: `LearnerTranscriptRow`.

## Implementation Approach

The route builds transcript summaries from enrollments and certificate records.

## Design Considerations

Transcript rows can later become immutable snapshots for compliance.

## Build Journal

Implemented transcript aggregation helper and route.

A junior developer should learn the difference between live reporting and historical records.

## As built (checked against this snapshot)

- `buildTranscriptSummary` (`packages/domain/src/transcript.ts:3-26`) counts active, completed and certified courses and averages completion.
- `GET /admin/learners/:learnerId/transcript` (`apps/api/src/routes/admin.routes.ts:209-235`), for instructors and admins, computes per-enrollment completion from lesson progress and returns rows plus the summary.

**Gaps.** `LearnerTranscriptRow` (`apps/api/prisma/schema.prisma:440-448`) is never written by the API, so the "historical record" exists only as a table. The route filters by `userId` only, not by organization: an instructor in any organization can read any learner's transcript. Learners have no route to read their own transcript.

**Exercise.** *Goal:* add `organizationId` from the header to the enrollment filter, and add `GET /learner/transcript` for the caller. *Check:* the instructor of another organization gets an empty list for a Northwind learner; the learner sees their own rows.
