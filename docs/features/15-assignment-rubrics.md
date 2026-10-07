# Feature: Assignment Rubrics

## Goal

Define scoring criteria for assignments.

## Impacted Areas of the Codebase

- Prisma models: `Rubric`, `RubricCriterion`, `Assignment`.
- Seed data.

## Implementation Approach

Assignments optionally reference rubrics. Rubrics belong to organizations and contain criteria with max points.

## Design Considerations

Rubrics can be reused across assignments and courses.

## Build Journal

Implemented rubric schema and seeded a security scenario rubric.

A junior developer should learn when to normalize reusable grading structure.

## As built (checked against this snapshot)

- `Rubric` (organization-owned) and `RubricCriterion` (with max points) exist (`apps/api/prisma/schema.prisma:306-323`); `Assignment.rubricId` is optional (`:297`). The seed creates one rubric and attaches it (`apps/api/prisma/seed.ts:134-160`).
- The catalog detail includes each assignment's rubric with criteria (`apps/api/src/routes/catalog.routes.ts:80`).

**Gaps.** Grading ignores the rubric: the score is a single integer 0-100 (`apps/api/src/routes/admin.routes.ts:90`), with no per-criterion scores and no check against the criteria's maximums.

**Exercise.** *Goal:* accept `criterionScores: [{ criterionId, points }]` when grading, validate each against the criterion's max points, and store the sum as the score. *Check:* a score above a criterion's maximum returns 400; a valid set stores a total equal to the sum.
