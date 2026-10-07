# Feature: Grading Workflows

## Goal

Let instructors review submitted assignments, assign scores, and provide feedback.

## Impacted Areas of the Codebase

- Prisma model: `AssignmentSubmission`.
- Admin route: `POST /admin/submissions/:submissionId/grade`.
- Instructor UI submissions table.

## Implementation Approach

Submissions move through `DRAFT`, `SUBMITTED`, `RETURNED`, and `GRADED`. Grading stores scorer, score, feedback, and timestamp.

## Design Considerations

Submission state must be separate from assignment definition because every learner has their own lifecycle.

## Build Journal

Implemented grading fields and route. The instructor dashboard highlights pending submissions.

A junior developer should learn to model workflow state explicitly instead of inferring it from nullable timestamps alone.

## As built (checked against this snapshot)

- `AssignmentSubmissionStatus` is `DRAFT | SUBMITTED | RETURNED | GRADED` (`apps/api/prisma/schema.prisma:37-42`).
- `POST /admin/submissions/:submissionId/grade` (`apps/api/src/routes/admin.routes.ts:86-104`), for instructors and admins, sets score (0-100), optional feedback, `gradedById`, `gradedAt` and `GRADED`.
- The instructor page lists submissions from demo data (`apps/web/app/instructor/page.tsx:56`).

**Gaps.** Nothing ever sets `RETURNED` or `DRAFT`. Grading does not check that the submission belongs to the caller's organization, does not require it to be `SUBMITTED` (you can re-grade, or grade a draft), and writes no audit log. A learner can resubmit after grading: the learner upsert sets the status back to `SUBMITTED` but keeps the old score (`apps/api/src/routes/learner.routes.ts:189-193`).

**Exercise.** *Goal:* require `status === "SUBMITTED"` and a matching organization before grading, and write a `submission.graded` audit row. *Check:* grading the same submission twice returns 409 the second time, and `GET /admin/audit-logs` shows one `submission.graded` entry.
