# Feature: Organization-Level Reporting

## Goal

Give organization admins visibility into enrollments, completion, and certificates by course.

## Impacted Areas of the Codebase

- Admin route: `GET /admin/organization-report`.
- Frontend reports page.
- Prisma enrollment and certificate relations.

## Implementation Approach

Reporting queries aggregate course enrollments, completions, and certificate counts.

## Design Considerations

Reporting is a first-class module because enterprise training buyers care about compliance and completion evidence.

## Build Journal

Implemented organization report route and dashboard UI.

A junior developer should learn that reporting should be designed early because it affects schema and indexing decisions.

## As built (checked against this snapshot)

- `GET /admin/organization-report` (`apps/api/src/routes/admin.routes.ts:54-84`), for `ORG_ADMIN` and `ADMIN`: per course in the organization, enrollment count, completion rate (rounded percent of `COMPLETED`), and certificate count.
- `GET /admin/dashboard` (`:16-52`) adds organization-wide counts, the 8 newest `SUBMITTED` submissions and 10 newest audit logs.
- The reports page renders demo data (`apps/web/app/reports/page.tsx`).

**Gaps.** The report loads every enrollment with all its lesson progress into memory to count them (`:59-69`); `lessonProgress` is included but never used. Because the completion route marks enrollments `COMPLETED` unconditionally, "completion rate" counts completions that did not meet any rule.

**Exercise.** *Goal:* rewrite the report with `groupBy` on `Enrollment` (`courseId`, `status`) and a count of certificates. *Check:* the JSON is identical to the current route's output for the seed data, and the Prisma query log shows no `LessonProgress` query.
