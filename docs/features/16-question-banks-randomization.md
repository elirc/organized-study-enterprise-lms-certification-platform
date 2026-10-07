# Feature: Quiz Question Banks and Randomized Attempts

## Goal

Support reusable question pools and deterministic randomized attempts.

## Impacted Areas of the Codebase

- Prisma models: `QuestionBank`, `QuizQuestion`, `QuizOption`.
- Domain helper: `selectRandomizedQuestionIds`.
- Quiz attempt route.

## Implementation Approach

Questions can belong to a quiz and question bank. Attempts store selected question order.

## Design Considerations

The attempt must store question order so audits and reviews can reproduce what the learner saw.

## Build Journal

Implemented question banks, option scoring, and randomized attempt order storage.

A junior developer should learn that randomness in assessment systems must become durable historical data.

## As built (checked against this snapshot)

- `QuestionBank`, `QuizQuestion` (optionally in a bank) and `QuizOption` with `isCorrect` (`apps/api/prisma/schema.prisma:245-275`); the seeded quiz has `maxAttempts: 3`, `passingScore: 80`, `questionCount: 2`, `randomizeQuestionOrder: true` (`apps/api/prisma/seed.ts:113-117`).
- `selectRandomizedQuestionIds` (`packages/domain/src/quiz.ts:24-36`): Fisher-Yates driven by a seeded linear congruential generator; the API seeds it with `<enrollmentId>-<attemptNumber>` (`apps/api/src/routes/learner.routes.ts:144-147`) and stores the order in `QuizAttempt.questionOrder` (`:164`).

**Gaps.** Questions are drawn from the quiz's own questions, not from the bank. The order is chosen at submission time, so the learner never sees it before answering; there is no "start attempt" step. Because the seed is predictable, anyone who knows the enrollment ID can compute the selection.

**Exercise.** *Goal:* split attempts into `POST .../attempts` (start: choose and store the order, return questions without `isCorrect`) and `POST .../attempts/:attemptId/submit` (score only the stored questions). *Check:* the questions returned at start are exactly the ones scored at submit, in the same order.
