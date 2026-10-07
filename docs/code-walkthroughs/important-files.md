# Code Walkthroughs for Important Files

These walkthroughs cover the most architecturally important files in the first codebase. Read them in this order when learning the system.

## `apps/api/prisma/schema.prisma`

### What the File Does

Defines the product database model: organizations, users, memberships, courses, versions, modules, lessons, enrollments, progress, quizzes, assignments, certificates, reporting support, audit logs, events, and operational records.

### Why It Exists

In a business application, the schema is a large part of the architecture. The API and frontend are easier to understand after you understand the nouns and relationships.

### Connections

- Seed data in `apps/api/prisma/seed.ts` creates realistic rows for this schema.
- API routes query and mutate these models.
- Domain helpers assume the same conceptual boundaries: content is separate from progress, definitions are separate from attempts/submissions.

### Important Sections

- `Course` and `CourseVersion`: stable course identity versus published/draft content.
- `Enrollment` and `LessonProgress`: learner state.
- `Quiz`, `QuizQuestion`, `QuizAttempt`: assessment definition versus attempt history.
- `Certificate`: generated artifact and verification record.
- `DomainEvent` and `OutboxMessage`: async integration boundary.

### Senior-Level Concepts

The schema protects historical truth. Learners enroll in a course version, so later authoring changes do not rewrite what they completed.

### How to Read It

Start with `Organization`, `User`, and `Membership`, then follow one course through `CourseVersion`, `Module`, `Lesson`, `Enrollment`, `LessonProgress`, `QuizAttempt`, `AssignmentSubmission`, and `Certificate`.

## `packages/domain/src/authorization.ts`

### What the File Does

Provides shared authorization helpers for role checks and sensitive actions.

### Why It Exists

Permissions are business rules. They should not be scattered across controllers and UI components.

### Connections

- API middleware uses `hasAnyRole` (`apps/api/src/middleware/auth.ts:62-73`). The other policy helpers in this file (`canManageCourse`, `canViewLearnerRecord`, `canImpersonate`, `hasRoleAtLeast`) are only called from tests today.
- Tests in `domain.test.ts` protect role behavior.
- Frontend role checks should mirror this only for UX, not security.

### Important Sections

- `roleRank`: defines role hierarchy.
- `canManageCourse`: course authoring policy.
- `canViewLearnerRecord`: transcript/reporting policy.
- `canImpersonate`: support tooling guardrail.

### Senior-Level Concepts

The backend is the source of truth. Frontend navigation can hide buttons, but API middleware must enforce access.

### How to Read It

Read from general helpers to specific policies. Ask, “which product workflow needs this rule?”

## `packages/domain/src/completion.ts`

### What the File Does

Calculates lesson completion and course/certificate eligibility.

### Why It Exists

Completion logic affects certificates, reporting, learner dashboards, and compliance. That makes it too important to bury in one route.

### Connections

- Learner dashboard progress uses the percentage helper.
- Enrollment completion uses certificate eligibility (`canIssueCertificate`, called at `apps/api/src/routes/learner.routes.ts:237-244`). `isCourseComplete` is not called by the API, and the course version's `completionRuleJson` is never read.
- Tests describe the expected behavior.

### Important Sections

- `calculateLessonCompletionPercent`.
- `isCourseComplete`.
- `canIssueCertificate`.

### Senior-Level Concepts

Small pure functions make high-risk business logic easy to test without databases, servers, or browsers.

### How to Read It

Read each function as a policy sentence. Then find the API route that calls it.

## `packages/domain/src/quiz.ts`

### What the File Does

Scores quiz answers, checks retake limits, and deterministically randomizes question selection.

### Why It Exists

Assessment logic needs repeatability and auditability.

### Connections

- `apps/api/src/routes/learner.routes.ts` uses `selectRandomizedQuestionIds` and `scoreQuiz` when creating quiz attempts (`:144-157`). The retake limit is checked inline (`:137-142`) instead of through `canStartQuizAttempt`.
- `domain.test.ts` verifies scoring and deterministic randomization.

### Important Sections

- `scoreQuiz`: exact answer matching.
- `canStartQuizAttempt`: policy helper.
- `selectRandomizedQuestionIds`: seeded randomization.

### Senior-Level Concepts

Randomized assessment systems must store what the learner saw. The route stores `questionOrder` in `QuizAttempt`.

### How to Read It

Trace an attempt from selected answers to score, then read the API route to see persistence.

## `apps/api/src/middleware/auth.ts`

### What the File Does

Authenticates JWTs and enforces organization-scoped roles.

### Why It Exists

Most enterprise SaaS bugs are tenant or permission bugs. Central middleware reduces duplicated checks.

### Connections

- Auth route creates JWTs.
- Catalog, learner, authoring, and admin routes require this middleware.
- Domain authorization helpers provide role logic.

### Important Sections

- `requireAuth`: verifies token and loads memberships.
- `requireOrganizationRole`: checks role against `x-organization-id`.

### Senior-Level Concepts

Tenant context must be explicit. This API uses an organization header so routes do not guess which tenant a user is acting in.

### How to Read It

Follow request headers into `request.user`, then into route-level authorization.

## `apps/api/src/routes/learner.routes.ts`

### What the File Does

Owns learner workflows: dashboard, enrollment, lesson progress, quiz attempts, assignment submissions, and completion.

### Why It Exists

This is the core vertical slice of the product.

### Connections

- Calls domain helpers for progress, scoring, randomization, and certificate eligibility.
- Writes domain events and audit logs.
- Reads and writes Prisma models across the learning workflow.

### Important Sections

- `/dashboard`: read model for learner homepage.
- `/courses/:courseId/enroll`: creates enrollment and lesson progress rows.
- `/quizzes/:quizId/attempts`: scores and stores attempts.
- `/enrollments/:enrollmentId/complete`: issues certificates when eligible.

### Senior-Level Concepts

A route should coordinate workflow, not hide all business logic. Notice how pure policy logic lives in `packages/domain`.

### How to Read It

Pick one workflow and trace database writes in order. Enrollment is the best first example.

## `apps/api/src/routes/authoring.routes.ts`

### What the File Does

Provides instructor/admin course authoring operations.

### Why It Exists

Authoring is intentionally separated from learner consumption.

### Connections

- Uses role middleware.
- Writes `Course`, `CourseVersion`, `Module`, and `Lesson`.
- Publishing updates both version and course status.

### Important Sections

- `POST /courses`: creates draft course and version.
- `POST /courses/:courseId/versions`: starts a new draft version.
- `POST /versions/:versionId/publish`: promotes version to published.

### Senior-Level Concepts

Publishing is a state transition. Treat it differently from ordinary editing.

### How to Read It

Read the publish route after the create route to understand lifecycle.

## `apps/api/src/routes/admin.routes.ts`

### What the File Does

Owns enterprise admin workflows: reporting, grading, imports, exports, feature flags, impersonation, transcripts, audit logs, and API keys.

### Why It Exists

Enterprise LMS products are purchased for administration and compliance as much as learning.

### Connections

- Uses organization role middleware.
- Calls domain transcript helper.
- Writes audit logs and domain events.

### Important Sections

- `/dashboard`: high-level training operations summary.
- `/organization-report`: reporting by course.
- `/bulk-import`: learner import.
- `/exports`: async export request.
- `/impersonation`: support session tracking.

### Senior-Level Concepts

Admin routes often need stricter audit and authorization than learner routes because they can affect many users.

### How to Read It

Read the route list as product capabilities, then inspect which roles are allowed.

## `apps/worker/src/worker.ts`

### What the File Does

Processes background jobs for certificates, notifications, video processing, and outbox dispatch.

### Why It Exists

Slow or retryable work should not block API requests.

### Connections

- Queues are defined in `apps/worker/src/queues.ts`. Nothing in the repository adds jobs to them (no `.add(...)` call, and the API does not import the worker), so in this snapshot the processors never run.
- Prisma models store job effects.
- Domain events and outbox messages provide async work records.

### Important Sections

- `certificate-generation`: updates certificate artifact URL.
- `notification-delivery`: marks notifications sent.
- `video-processing`: mock processing state transition.
- `outbox-dispatch`: marks pending outbox messages sent.

### Senior-Level Concepts

Retries and idempotency matter. This mock worker is simple, but the structure teaches where production concerns would live.

### How to Read It

Read each worker as a state transition from queued work to durable database update.

## `apps/web/components/app-shell.tsx`

### What the File Does

Defines the shared application chrome and navigation.

### Why It Exists

Real SaaS apps need consistent navigation across role-focused workflows.

### Connections

- Used by all Next.js pages.
- Imports Lucide icons for recognizable controls.
- Links to learner, catalog, course, instructor, admin, reporting, verification, login, and settings pages.

### Important Sections

- `navItems`: the product map.
- `AppShell`: sidebar, topbar, and main content layout.

### Senior-Level Concepts

Navigation reveals product architecture. The app shell makes role boundaries visible to users.

### How to Read It

Map each nav item to an API module or product workflow.

## `apps/web/app/page.tsx`

### What the File Does

Renders the learner dashboard.

### Why It Exists

The learner dashboard is the first everyday workflow: continue courses, see progress, and track due items.

### Connections

- Uses demo data now; can later call `apps/web/lib/api.ts`. No page imports `lib/api.ts` yet: every page, including login and certificate verification, renders `lib/demo-data.ts`.
- Mirrors learner API dashboard structure.
- Links conceptually to progress, assignments, quizzes, and certificates.

### Important Sections

- Metrics grid.
- Enrolled courses list.
- Due items table.

### Senior-Level Concepts

A dashboard is a read model. It should be shaped around user decisions, not raw database tables.

### How to Read It

Compare the page sections to the learner API response and identify which fields would become live data.

## `docs/features/*.md`

### What the Files Do

Each feature document records the goal, impacted areas, implementation approach, design considerations, and build journal.

### Why They Exist

They teach the habit of thinking before coding and summarizing after coding.

### Connections

- The backlog links to feature docs.
- The final design document summarizes their combined result.

### Senior-Level Concepts

Maintainability is partly written memory. Future developers need to know why the system is shaped this way.

### How to Read Them

Pick a feature, read the plan, inspect the files it names, then read the build journal.

## Line references and review notes (2026-10 docs pass)

The walkthroughs above describe intent. This section pins each file to line numbers in this snapshot and records what a reviewer finds there. Nothing was run.

### `apps/api/prisma/schema.prisma`

- Roles and statuses are enums at the top (`:10-67`). Tenancy: `Organization` (`:69`), `User` (`:85`), `Membership` unique per `(userId, organizationId)` (`:114`).
- Content: `Course` with `currentVersionId` (`:118-141`), `CourseVersion` unique per `(courseId, versionNumber)` (`:142-163`), `Module` and `Lesson` unique per position (`:172`, `:189`).
- Activity: `Enrollment` unique per `(userId, courseVersionId)` (`:192-216`), `LessonProgress` unique per `(enrollmentId, lessonId)` (`:227`), `QuizAttempt` unique per `(quizId, enrollmentId, attemptNumber)` (`:291`), `AssignmentSubmission` unique per `(assignmentId, enrollmentId)` (`:341`), `Certificate` unique per enrollment and verification code (`:344-359`).
- Review notes: `Cohort.courseId` and `VideoAsset` have no relation to `Course`/`Organization`; `QuizOption.isCorrect` (`:271`) sits next to the option text, so any query that includes options must be careful about who receives it.

### `apps/api/src/middleware/auth.ts`

- `requireAuth` (`:13-46`): Bearer token, `jwt.verify` with `config.jwtSecret`, user and memberships loaded per request; any verification error becomes 401 "Invalid authentication token".
- `requireOrganizationRole(roles)` (`:48-82`): organization from the `x-organization-id` header (400 if missing), role checked with `hasAnyRole`.
- Review note: the header proves the caller's role *in that organization*; routes still have to check that the record they touch is in the same organization. Most do not (see `docs/architecture/final-system-design.md`, finding 4).

### `apps/api/src/routes/learner.routes.ts`

- `/dashboard` (`:14-49`), `/courses/:courseId/enroll` (`:51-95`), `/lessons/:lessonId/progress` (`:97-117`), `/quizzes/:quizId/attempts` (`:119-173`), `/assignments/:assignmentId/submissions` (`:175-204`), `/enrollments/:enrollmentId/complete` (`:206-268`).
- Review notes: no route checks that the enrollment in the body or path belongs to the caller; enrollment picks the highest version number even if it is a draft (`:56-59`); completion sets `COMPLETED` before deciding anything (`:228-234`) and treats "no submissions" as "assignments complete" (`:226`); enrollment, lesson-progress rows and the domain event are separate writes with no transaction (`:66-91`).

### `apps/api/src/routes/authoring.routes.ts`

- Router guard: `requireAuth` + `requireOrganizationRole(["INSTRUCTOR", "ORG_ADMIN", "ADMIN"])` (`:10`).
- `POST /courses` (`:18-60`) creates the course and a `DRAFT` version 1 with a default completion rule (`:34-38`) and audits; `POST /courses/:courseId/versions` (`:62-90`); `POST /versions/:versionId/modules` (`:92-105`); `POST /modules/:moduleId/lessons` (`:107-131`); `POST /versions/:versionId/publish` (`:133-154`).
- Review notes: only course creation writes an audit log; the version, module, lesson and publish routes look records up by ID without checking the organization; publishing is two writes without a transaction.

### `apps/api/src/routes/admin.routes.ts`

- Dashboard (`:16-52`), organization report (`:54-84`), grading (`:86-104`), bulk import (`:106-143`), exports (`:145-168`), feature flags (`:170-191`), impersonation (`:193-207`), transcript (`:209-235`), audit logs (`:237-249`), API keys (`:251-269`), and a 404 fallback for unknown admin paths (`:271-273`).
- Review notes: grading and transcript ignore the organization; bulk import gives every new learner `Password123!` and silently skips adding existing users; impersonation, flags, grading and API keys are not audited.

### `apps/api/src/http.ts`

- `HttpError` (`:18-25`), `asyncHandler` (`:27-33`), `errorHandler` (`:35-47`). Review note: only `HttpError` keeps its status; validation and database errors become 500 with their raw message.

### `apps/worker/src/worker.ts` and `queues.ts`

- Four queues with retry policies (`queues.ts:9-41`); four workers (`worker.ts:8-73`); a shared `failed` logger (`:75-79`).
- Review notes: nothing produces jobs; the outbox worker marks messages `SENT` without delivering them; none of the processors is idempotent beyond "update to the same value".

### `apps/web`

- `components/app-shell.tsx:6-16` is the product map (nine nav items, including a fixed demo certificate code `CERT-DEMO2026`).
- Every page renders `lib/demo-data.ts`. The verification page always shows "Verified" (`app/certificates/verify/[code]/page.tsx:7`, `:17-19`); the login button is `type="button"` with no handler (`app/login/page.tsx:24`).

## Exercises

Each has a **Goal** and a **Check** you can observe.

1. **Trace one enrollment.**
   *Goal:* log in as `orgadmin@northwind.test` (the seed only enrolls the learner, and a second enrollment in the same version would hit the unique index), enroll in the seeded course, then list the rows created.
   *Check:* one `Enrollment`, one `LessonProgress` per lesson in the enrolled version, one `DomainEvent` of type `enrollment.created` and one `OutboxMessage` in `PENDING` — which stays `PENDING`, because nothing dispatches it.
2. **Find the cross-tenant write.**
   *Goal:* create a second organization in Prisma Studio with its own instructor, then, as that instructor with the second organization's ID in the header, call `POST /authoring/versions/<a Northwind version id>/publish`.
   *Check:* it succeeds. Add an organization check (load the version's course and compare `organizationId`) and confirm it now returns 404.
3. **Connect one page.**
   *Goal:* make `apps/web/app/catalog/page.tsx` call `apiGet("/catalog/courses", session)` with a token from a real login.
   *Check:* the catalog shows the seeded published course from the database, and renaming the course in Prisma Studio changes what the page shows.
