# Feature: Scheduled Cohorts

## Goal

Group learners into scheduled course runs with start and end dates.

## Impacted Areas of the Codebase

- Prisma models: `Cohort`, `Enrollment`.
- Seed data: demo cohort.
- Admin and reporting dashboards.

## Implementation Approach

`Cohort` belongs to an organization and course. Enrollments optionally reference a cohort so reporting can group learners by run.

## Design Considerations

Cohorts are not the course itself. They are scheduling and reporting containers around a course.

## Build Journal

Implemented cohort storage and seeded a May 2026 security cohort. This connects to enrollment workflows through `Enrollment.cohortId`.

A junior developer should learn to avoid mixing schedule state into course content tables. Future improvements include cohort-specific announcements and waitlists.

## As built (checked against this snapshot)

- `Cohort` (`apps/api/prisma/schema.prisma:361-371`) has `organizationId`, `courseId`, `name`, `startsAt`, `endsAt`. `courseId` is a plain string with no relation to `Course`, so the database does not check it.
- `Enrollment.cohortId` is optional (`:198`, relation at `:206`).
- The seed creates one cohort and attaches the seeded enrollment to it (`apps/api/prisma/seed.ts:165-177`).

**Gaps.** No API route creates, lists or reports cohorts; the enrollment route never sets `cohortId`. The admin and reporting dashboards mentioned above do not group by cohort.

**Exercise.** *Goal:* add `courseId` as a real relation and a `GET /admin/cohorts` route that returns each cohort with its enrollment count. *Check:* the seeded cohort is listed with one enrollment, and creating a cohort with a nonexistent `courseId` now fails with a foreign-key error.
