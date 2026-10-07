# Feature: Retake Policies, Due Dates, Late Submissions, and Completion Rules

## Goal

Represent real assessment policies around attempts, dates, and completion.

## Impacted Areas of the Codebase

- `Quiz.maxAttempts`.
- `Enrollment.dueAt`.
- `Assignment.dueDaysAfterEnrollment`.
- Domain completion helpers.

## Implementation Approach

The quiz attempt route checks max attempts. Enrollments and assignments store due-date inputs. Completion rules live in shared domain helpers.

## Design Considerations

Policy logic belongs in shared helpers so it can be tested without the database.

## Build Journal

Implemented retake enforcement, due-date fields, and domain completion logic.

A junior developer should learn to pull high-risk business rules out of controllers.

## As built (checked against this snapshot)

- Retakes: the quiz route counts existing attempts and returns 409 once `maxAttempts` is reached (`apps/api/src/routes/learner.routes.ts:137-142`). `canStartQuizAttempt` in `packages/domain/src/quiz.ts:20-22` implements the same rule but is not used by the route.
- Due dates: enrollments get `dueAt` = now + 21 days (`learner.routes.ts:70`); `Assignment.dueDaysAfterEnrollment` exists (`apps/api/prisma/schema.prisma:300`).
- Completion: `calculateLessonCompletionPercent`, `isCourseComplete` and `canIssueCertificate` in `packages/domain/src/completion.ts`.

**Gaps.** The attempt count is read before the insert, so two simultaneous submissions can both pass the limit; the unique index on `(quizId, enrollmentId, attemptNumber)` then rejects one with a 500. Nothing reads `dueAt` or `dueDaysAfterEnrollment`, so late submissions are neither flagged nor blocked. The completion route does not use `isCourseComplete` (see `docs/architecture/final-system-design.md`, finding 2).

**Exercise.** *Goal:* mark a submission late when `submittedAt` is after enrollment `createdAt` + `dueDaysAfterEnrollment`. *Check:* with `dueDaysAfterEnrollment` set to 0 on the seeded assignment, a new submission is flagged late, and with it unset it is not.
