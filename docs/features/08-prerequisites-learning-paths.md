# Feature: Prerequisites and Learning Paths

## Goal

Support ordered learning journeys and course dependencies.

## Impacted Areas of the Codebase

- Prisma models: `LearningPath`, `LearningPathItem`, `CoursePrerequisite`.
- Catalog course detail includes prerequisites.

## Implementation Approach

Learning paths are ordered collections of courses. Prerequisites connect a course to required prior courses.

## Design Considerations

Prerequisites are graph-like data; future work should detect cycles and enforce completion before enrollment.

## Build Journal

Implemented schema support and API include for prerequisites.

A junior developer should learn to separate roadmap structure from course content itself.

## As built (checked against this snapshot)

- `LearningPath`, `LearningPathItem` (unique `(learningPathId, position)`) and `CoursePrerequisite` (unique `(courseId, prerequisiteId)`) exist (`apps/api/prisma/schema.prisma:393-419`).
- `GET /catalog/courses/:courseId` includes prerequisites (`apps/api/src/routes/catalog.routes.ts:77`).

**Gaps.** The seed creates no paths or prerequisites (it only deletes them), no route manages them, and enrollment does not check prerequisites. The cycle detection mentioned in the design notes does not exist.

**Exercise.** *Goal:* before creating an enrollment, check that every prerequisite course has a `COMPLETED` enrollment for the user, and return 409 otherwise. *Check:* after adding a prerequisite to the seeded course in Prisma Studio, enrolling a fresh user fails with 409 and names the missing course.
