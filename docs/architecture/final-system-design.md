# Final System Design

## Repository Structure

- `apps/api`: Express modular monolith API with Prisma.
- `apps/web`: Next.js SaaS frontend.
- `apps/worker`: BullMQ worker process.
- `packages/domain`: shared business rules and tests.
- `docs/architecture`: architecture planning, journal, and final design.
- `docs/features`: feature plans and build journals.
- `docs/code-walkthroughs`: guided reading for important files.
- `docs/backlog`: phased product backlog.

## Major Modules

### Identity and Authorization

Users belong to organizations through memberships. Membership roles are `LEARNER`, `INSTRUCTOR`, `ORG_ADMIN`, and `ADMIN`. API middleware loads the current user and checks organization-scoped roles.

### Content and Authoring

Courses are stable product records. Course versions hold publishable learning content. Modules and lessons belong to versions. This separation lets instructors draft future versions without changing what learners already enrolled in.

### Learner Activity

Enrollments connect users to course versions. Lesson progress, quiz attempts, assignment submissions, and certificates are separate records so the system can report and audit learning history.

### Assessment

Quizzes have definitions, questions, options, attempt policies, randomized question order, and attempts. Assignments have definitions, submissions, rubrics, grading state, score, and feedback.

### Certification

Certificates are generated artifacts with verification codes and revocation status. They are issued after completion rules are met.

### Reporting and Administration

Admin routes expose dashboards, organization reports, transcripts, audit logs, bulk import, CSV export requests, feature flags, impersonation, and API key management.

### Async Platform

Domain events and outbox messages record asynchronous work. The worker processes certificates, notifications, video assets, and outbox dispatch with retry policies.

## Key Workflows

1. Instructor creates a course and draft course version.
2. Instructor adds modules, lessons, quizzes, assignments, and rubrics.
3. Instructor publishes the course version.
4. Learner enrolls in the published version.
5. System creates lesson progress rows.
6. Learner completes lessons, submits assignments, and attempts quizzes.
7. Instructor grades submissions and provides feedback.
8. Learner completes the enrollment.
9. System issues a certificate if rules are met.
10. Admins report on completion and audit important actions.

## System Design Decisions

- Modular monolith instead of microservices to teach boundaries without distributed complexity.
- Prisma schema as the central domain map.
- Pure domain helpers for high-risk business rules.
- Backend authorization as source of truth.
- Versioned content to preserve learner history.
- Outbox pattern to prepare for reliable async integration.
- Documentation as part of the codebase, not a separate afterthought.

## Tradeoffs

- The frontend uses demo data fallbacks so it can teach the UI without requiring infrastructure first.
- API routes are modular but still intentionally simple; a larger production app might add service classes per module.
- SCORM, video processing, exports, and public API are production-shaped scaffolds, not full external integrations.
- Reporting uses direct Prisma queries rather than materialized analytics tables.

## Improvements

- Add full Playwright workflows against a seeded local database.
- Add generated OpenAPI contracts from route schemas.
- Add real CSV export artifact storage.
- Add full SCORM runtime tracking.
- Add cycle detection for prerequisites.
- Add richer tenant isolation tests.
- Add CI with PostgreSQL and Redis service containers.
- Add migration review checklists and schema-change ADRs.

## Running it (from the code; this snapshot has no root README)

1. `docker compose up -d` starts PostgreSQL 16 (user, password and database all `lms`) and Redis 7 (`docker-compose.yml`).
2. Create `apps/api/.env` with `DATABASE_URL=postgresql://lms:lms@localhost:5432/lms` and a real `JWT_SECRET`. `apps/api/src/config.ts:5` falls back to `"development-only-secret"` when `JWT_SECRET` is missing, so forgetting it still "works" with a secret that is in the repository.
3. `npm install`, then `npm run db:generate`, `npm run db:migrate` (runs `prisma migrate dev`; there is no `migrations/` folder yet, so the first run creates one) and `npm run db:seed`.
4. `npm run dev` starts the API (port 4000) and the web app together; `npm run dev --workspace @lms/worker` starts the worker (`tsx watch src/worker.ts`).
5. Seeded logins (all with password `Password123!`, `apps/api/prisma/seed.ts:19-22`): `learner@`, `instructor@`, `admin@` and `orgadmin@northwind.test`, in one organization, slug `northwind-learning`.

To call the API, `POST /auth/login` with email and password returns a JWT (8-hour expiry, `auth.routes.ts:36`). Send it as `Authorization: Bearer <token>` and send the organization ID as `x-organization-id` on every route that needs tenant context.

## What the code shows (2026-10 docs pass)

Checked against this snapshot by reading the code; nothing was run. Line numbers refer to this snapshot.

### Identity and authorization

- `requireAuth` (`apps/api/src/middleware/auth.ts:13-46`) verifies the JWT and loads the user's memberships on every request.
- `requireOrganizationRole` (`:48-82`) checks that the user has one of the listed roles **in the organization named by the `x-organization-id` header**. It does not check that the record being changed belongs to that organization.
- Routers that use it: authoring (all routes, `authoring.routes.ts:10`), and most admin routes. The catalog and learner routers use only `requireAuth`.
- Of the domain authorization helpers (`packages/domain/src/authorization.ts`), only `hasAnyRole` is used by the API. `canManageCourse`, `canViewLearnerRecord`, `canImpersonate` and `hasRoleAtLeast` appear only in tests.

### The learner workflow, as implemented

| Step | Route (in `apps/api/src/routes/learner.routes.ts`) | What it does |
|---|---|---|
| Enroll | `POST /learner/courses/:courseId/enroll` (`:51-95`) | takes the course's **highest-numbered version, whatever its status**, creates an enrollment due in 21 days, one `LessonProgress` per lesson, and an `enrollment.created` event + outbox message |
| Progress | `POST /learner/lessons/:lessonId/progress` (`:97-117`) | updates the `(enrollmentId, lessonId)` progress row from the body |
| Quiz | `POST /learner/quizzes/:quizId/attempts` (`:119-173`) | checks `maxAttempts`, picks a seeded random question order, scores with `scoreQuiz`, stores answers, order, score and `passed` |
| Assignment | `POST /learner/assignments/:assignmentId/submissions` (`:175-204`) | upserts the submission as `SUBMITTED` |
| Complete | `POST /learner/enrollments/:enrollmentId/complete` (`:206-268`) | **always** marks the enrollment `COMPLETED`, then issues a certificate only if `canIssueCertificate` passes, then audits |

### Review findings

1. **Learner routes do not check ownership.** None of progress, quiz attempt, submission or completion verifies that `enrollmentId` belongs to the caller. Any signed-in user who knows an enrollment ID can complete another learner's lessons, submit their quiz or assignment, or complete their enrollment (`learner.routes.ts:100-113`, `:122-169`, `:179-200`, `:209-234`).
2. **Completion ignores the course's completion rule.** `CourseVersion.completionRuleJson` (set at `authoring.routes.ts:34-38`) is never read. The completion route always sets `COMPLETED` (`:228-234`) and decides the certificate with its own inputs: any passed quiz, and `submissions.every(GRADED)` — which is `true` when there are no submissions, so assignments are never actually required (`:225-226`). `isCourseComplete` in `packages/domain/src/completion.ts:18-36`, which does read a rule, is unused.
3. **Enrollment can target a draft version.** Enroll uses `versions orderBy versionNumber desc take 1` (`:56-59`), not `Course.currentVersionId` or a `PUBLISHED` status, so once an instructor starts version 2, new learners are enrolled into the unpublished draft. This contradicts the versioning feature's own goal.
4. **Cross-organization writes in authoring and admin.** The role check uses the header's organization, but `POST /authoring/courses/:courseId/versions`, `/versions/:versionId/modules`, `/modules/:moduleId/lessons`, `/versions/:versionId/publish` (`authoring.routes.ts:62-154`) and `POST /admin/submissions/:submissionId/grade` (`admin.routes.ts:86-104`) look records up by ID only. An instructor in one organization can publish or grade in another. `GET /admin/learners/:learnerId/transcript` (`:209-235`) has the same gap for reads.
5. **The catalog leaks answers and drafts.** `GET /catalog/courses/:courseId` (`catalog.routes.ts:72-102`) returns every version, including drafts, with quiz options and their `isCorrect` flags, to any signed-in user, without an organization check. `GET /catalog/courses` lets any user pass `status=DRAFT` (`:15`, `:29`) for any organization ID they put in the header.
6. **Bulk import can rename users in other organizations and gives everyone the same password.** It upserts by email; for an existing user it only updates `name` and does **not** add a membership (`admin.routes.ts:120-129`), yet reports them as imported. New users get `Password123!` (`:116`).
7. **Impersonation, feature flags, grading and API keys write no audit log** (`admin.routes.ts:86-104`, `:170-191`, `:193-207`, `:251-269`), although the feature docs call impersonation "auditable". Impersonation does not check that the target belongs to the organization.
8. **Errors from validation become 500s.** `errorHandler` (`apps/api/src/http.ts:35-47`) maps only `HttpError`; a Zod error or a Prisma unique violation (completing an enrollment twice hits `Certificate.enrollmentId @unique`) is returned as 500 with the raw message.
9. **Nothing enqueues worker jobs.** The queues in `apps/worker/src/queues.ts` are never imported by the API, and no code calls `.add(...)`. The outbox worker only runs when an `outbox-dispatch` job arrives, so outbox messages stay `PENDING`, and certificate artifacts, notifications and video processing never run.
10. **The web app shows only demo data.** Every page imports `apps/web/lib/demo-data.ts`; `apps/web/lib/api.ts` is never imported. The certificate verification page shows "Verified" and the first demo certificate for **any** code (`apps/web/app/certificates/verify/[code]/page.tsx:7`, `:17-19`). The login button has no handler (`apps/web/app/login/page.tsx:24`).
11. **Tests:** ten domain tests (`packages/domain/src/domain.test.ts`) and one API test that only checks `createApp` returns a function (`apps/api/src/app.test.ts`).

## Exercises

Each has a **Goal** and a **Check** you can observe.

1. **Log in and read the dashboard.**
   *Goal:* `POST /auth/login` as `learner@northwind.test`, then `GET /learner/dashboard` with the token and the organization ID from the login response.
   *Check:* you see the seeded enrollment(s) with a `progressPercent` computed from lesson progress; without the `x-organization-id` header the list is empty (the filter becomes `organizationId: ""`).
2. **Prove finding 1, then fix it.**
   *Goal:* log in as the instructor and call `POST /learner/enrollments/<the learner's enrollment id>/complete`.
   *Check:* it succeeds and the learner's enrollment is `COMPLETED`. Add a check that the enrollment's `userId` equals the caller, and confirm the same request now returns 404 or 403.
3. **Use the completion rule.**
   *Goal:* make the completion route read `courseVersion.completionRuleJson` and call `isCourseComplete`; only mark `COMPLETED` when it returns true.
   *Check:* completing an enrollment with 0% lesson progress now returns 409 and leaves the status unchanged.
4. **Stop the answer leak.**
   *Goal:* give the catalog detail route an explicit `select` for quiz options that omits `isCorrect`, and only return the published version to non-instructors.
   *Check:* as the learner, the response contains no `isCorrect` key anywhere; as the instructor, the authoring view (a new route) still can see it.
5. **Wire one queue.**
   *Goal:* after the certificate is created, enqueue `certificate-generation` with `{ certificateId }`.
   *Check:* with the worker running, the certificate's `artifactUrl` changes from `/certificates/<enrollmentId>.pdf` to `/generated/certificates/<certificateId>.pdf`.
