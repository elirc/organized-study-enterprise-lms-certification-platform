# Feature: Course Versioning

## Goal

Allow course authors to create new versions without rewriting the version learners are already enrolled in.

## Impacted Areas of the Codebase

- Prisma models: `Course`, `CourseVersion`, `Module`, `Lesson`, `Quiz`, `Assignment`.
- API routes: authoring version creation and publishing.
- Learner enrollment: enrollments point to a specific course version.

## Implementation Approach

The schema stores stable course identity in `Course` and versioned content in `CourseVersion`. Modules, lessons, quizzes, assignments, content packages, discussions, and certificates connect to a version. Publishing updates `Course.currentVersionId`.

## Design Considerations

This protects historical learner records. A learner enrolled in version 1 should not see their assignments change when an instructor drafts version 2.

## Build Journal

Implemented course versions in the Prisma schema and authoring routes. This connects to enrollment by storing both `courseId` and `courseVersionId`.

A junior developer should learn that versioning is usually a relationship design problem before it is a UI problem. The tradeoff is more joins and more careful publishing logic.

## As built (checked against this snapshot)

- Schema: `Course.currentVersionId` (`apps/api/prisma/schema.prisma:122`), `CourseVersion` unique on `(courseId, versionNumber)` (`:161`), `Enrollment` stores both `courseId` and `courseVersionId` and is unique on `(userId, courseVersionId)` (`:213`).
- Authoring: `POST /authoring/courses` creates version 1 as `DRAFT` (`apps/api/src/routes/authoring.routes.ts:18-60`); `POST /authoring/courses/:courseId/versions` creates the next number as `DRAFT`, copying the completion rule but **not** the modules, lessons, quizzes or assignments (`:62-90`); `POST /authoring/versions/:versionId/publish` sets the version `PUBLISHED` and points `Course.currentVersionId` at it (`:133-154`).

**Gaps.** Enrollment ignores `currentVersionId` and picks the highest version number whatever its status (`apps/api/src/routes/learner.routes.ts:56-59`), so learners can land in an unpublished draft. Publishing is two separate writes (not a transaction), does not check the version was a draft, and does not archive the previous published version. None of the version routes checks that the course belongs to the caller's organization.

**Exercise.** *Goal:* change enrollment to use `course.currentVersionId` and return 404 when it is null. *Check:* after creating version 2 (draft) of the seeded course, a new enrollment still points at version 1; after publishing version 2, the next enrollment points at version 2 while the earlier one keeps version 1.
