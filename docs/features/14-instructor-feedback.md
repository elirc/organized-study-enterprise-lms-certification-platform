# Feature: Instructor Feedback Tools

## Goal

Let instructors provide feedback during grading.

## Impacted Areas of the Codebase

- `AssignmentSubmission.feedback`.
- Grading route.
- Instructor submissions UI.

## Implementation Approach

Feedback is stored with graded submissions and returned in assignment workflows.

## Design Considerations

Feedback should be part of the assessment record because it explains the score.

## Build Journal

Implemented feedback fields in grading.

A junior developer should learn that scores without feedback are often poor product experiences.

## As built (checked against this snapshot)

- `AssignmentSubmission.feedback` is set by the grading route together with the score (`apps/api/src/routes/admin.routes.ts:90-99`); feedback is optional.
- The learner dashboard returns submissions, so feedback reaches the learner through `GET /learner/dashboard` (`apps/api/src/routes/learner.routes.ts:26`, `:44`).

**Gaps.** Feedback is one free-text field per submission: no history, no per-criterion comments (rubrics are not used in grading), and no notification to the learner. If the learner resubmits, the old feedback stays attached to the new content.

**Exercise.** *Goal:* clear `score`, `feedback` and `gradedAt` when a learner resubmits. *Check:* after grading and then resubmitting, the dashboard shows the submission as `SUBMITTED` with no score or feedback.
