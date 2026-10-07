# Initial Architecture Plan

This document comes before the first implementation pass. The point is to make the codebase easier to reason about for a junior developer learning how larger systems are shaped.

## Product Shape

The application is an enterprise learning management and certification platform. Companies create courses, publish structured learning content, enroll learners, track progress, assess learners with quizzes and assignments, issue certificates, and report on compliance or training completion.

The first version is a modular monolith. That means one deployable backend owns the product workflows, but the code is organized into clear modules so it could later be split into services if there were a real operational need.

## System Boundaries

- `apps/api`: Express API, Prisma database access, authentication, authorization, audit logging, and product workflows.
- `apps/web`: Next.js frontend for learners, instructors, admins, and organization admins.
- `apps/worker`: background processors for certificate generation, notification reminders, video processing mocks, and outbox dispatch.
- `packages/domain`: shared TypeScript business rules, policy helpers, completion calculations, quiz scoring, transcript helpers, and feature flag utilities.
- `docs`: architecture plans, architecture journal, feature plans, build journals, walkthroughs, and final system design.

## Architectural Principles

- Content and learner state are separate. A lesson definition is not a learner's progress through that lesson.
- Authoring and publishing are separate. Draft content can change without rewriting the version a learner is enrolled in.
- Quiz definitions and quiz attempts are separate. Attempts are historical records.
- Assignment definitions and submissions are separate. Submissions can be graded, returned, and audited.
- Certificates are generated artifacts. They have identifiers, verification codes, issue dates, and revocation state.
- Reporting is a product module, not a collection of random SQL queries.
- Backend authorization is the source of truth. Frontend checks are only for navigation and experience.
- Audit logs capture security-sensitive and compliance-sensitive events.
- Domain events and an outbox give the codebase a realistic path toward asynchronous workflows.

## Data Model Overview

The core model groups are:

- Identity: users, organizations, memberships, roles, API keys, impersonation sessions.
- Learning content: courses, course versions, modules, lessons, quiz definitions, assignment definitions, rubrics, question banks, content packages, video assets.
- Learner activity: enrollments, lesson progress, quiz attempts, assignment submissions, certificates, transcripts.
- Scheduling and collaboration: cohorts, cohort enrollments, discussion threads, discussion posts, feedback.
- Platform operations: feature flags, notifications, audit logs, domain events, outbox messages.

## Backend Module Plan

- `auth`: login, demo auth, JWT creation, current user resolution.
- `authorization`: role and organization checks.
- `organizations`: tenant-level membership and reporting.
- `courses`: catalog, authoring, publishing, versioning, prerequisites, learning paths.
- `enrollments`: enrollment workflow, progress rollups, completion rules.
- `quizzes`: quiz creation, question banks, randomized attempts, retake policies.
- `assignments`: definitions, submissions, rubrics, grading, due dates.
- `certificates`: generation, verification, revocation.
- `reporting`: learner, instructor, admin, organization reporting.
- `admin`: bulk import, impersonation, feature flags, audit review.
- `integrations`: public API keys and CSV exports.
- `platform`: health, observability, domain events, outbox.

## Frontend Plan

The first screen should feel like an internal SaaS product, not a landing page. It will provide:

- Learner dashboard with enrolled courses, progress, due items, certificates, transcript.
- Catalog with search, filtering, pagination, and prerequisites.
- Course workspace with modules, lessons, quizzes, assignments, and discussions.
- Instructor dashboard for authoring, cohorts, grading, question banks, and feedback.
- Admin dashboard for organization reporting, users, bulk import, feature flags, audit logs, API keys, exports, and impersonation.
- Certificate verification page.

## Background Jobs

The worker will support realistic async boundaries:

- Certificate artifact generation after completion.
- Notification and reminder delivery.
- Mock video processing for uploaded course videos.
- Outbox dispatch with retry policy.
- CSV export generation.

## Testing Strategy

The first pass focuses tests where business logic is easiest to break:

- Role and permission helpers.
- Completion rules.
- Quiz scoring and randomized question selection.
- Certificate eligibility.
- Transcript aggregation.
- Feature flag evaluation.

API and browser workflow tests are added once the first vertical workflows exist.

## Documentation Strategy

The docs directory is part of the deliverable:

- Architecture plan before implementation.
- Architecture journal while building.
- Feature plan before every backlog feature.
- Build journal appended after each feature.
- Code walkthrough docs for architecturally important files.
- Final system design document after the backlog is implemented.

## Plan versus code (2026-10 docs pass)

This plan was written before the code. Checked against this snapshot:

| Plan item | In the code |
|---|---|
| `apps/api`, `apps/web`, `apps/worker`, `packages/domain` | All four exist as npm workspaces (root `package.json`). |
| Backend modules `auth` … `platform` | Seven route files instead of twelve modules: `auth`, `catalog`, `authoring`, `learner`, `admin`, `certificate`, `platform` (`apps/api/src/app.ts:20-26`). Enrollments, quizzes and assignments live in `learner.routes.ts`; reporting, imports, impersonation, flags and API keys in `admin.routes.ts`. There are no service classes; routes call Prisma directly. |
| "JWT creation, demo auth" | Real JWT login with bcrypt (`apps/api/src/routes/auth.routes.ts:17-50`); no separate demo-auth path. |
| Principle: "Backend authorization is the source of truth" | Role checks exist per organization, but several routes do not check that the record belongs to that organization, and the learner routes do not check ownership (`docs/architecture/final-system-design.md`, findings 1 and 4). |
| Principle: "Audit logs capture security-sensitive events" | Audited: course creation, enrollment completion, bulk import. Not audited: impersonation, grading, feature flags, API keys, publishing. |
| Data model groups | All listed models exist (`apps/api/prisma/schema.prisma`, 36 models). Several have no API yet: cohorts, learning paths, prerequisites, content packages, video assets, transcript rows, notifications. |
| Frontend plan | All listed screens exist, but they render `apps/web/lib/demo-data.ts`; none calls the API. |
| Background jobs | Four queues and processors exist (`apps/worker/src`); CSV export generation has no processor, and nothing enqueues jobs for the other four. |
| Testing strategy | Domain tests cover roles, completion, quiz scoring and randomization, transcript summary and feature flags (`packages/domain/src/domain.test.ts`). API and browser workflow tests have not been added. |

## Exercise

**Goal:** pick one planned principle — "Content and learner state are separate" — and find the code that enforces it and the code that undermines it.

**Check:** you cite `Enrollment.courseVersionId` (the separation, `schema.prisma:192-216`) and the enroll route's choice of the highest version number regardless of status (`apps/api/src/routes/learner.routes.ts:56-59`), which can attach learner state to unpublished content.
