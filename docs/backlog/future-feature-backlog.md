# Future Feature Backlog

The initial build intentionally includes a broad slice of the LMS product so later features have realistic places to land. The backlog below is organized as product phases. Each item has a companion feature document in `docs/features`.

## Phase 1 - Content Lifecycle

1. Course versioning
2. Scheduled cohorts
3. SCORM-like content packages
4. Video processing mock pipeline
5. Course search/filtering/pagination
6. Prerequisites and learning paths

## Phase 2 - Assessment Depth

1. Grading workflows
2. Assignment rubrics
3. Quiz question banks
4. Randomized quiz attempts
5. Retake policies
6. Due dates and late submissions
7. Completion rules

## Phase 3 - Learner Experience

1. Discussion forums
2. Notifications and reminders
3. Learner transcript
4. Certificate verification page
5. Instructor feedback tools

## Phase 4 - Enterprise Administration

1. Organization-level reporting
2. Public API for enterprise customers
3. Bulk learner import
4. CSV exports
5. Admin impersonation
6. Feature flags

## Phase 5 - Platform Maturity

1. Domain events
2. Outbox pattern
3. Background job retry policies
4. Observability and health checks
5. API contracts
6. End-to-end tests
7. Database migration discipline
8. Final system design documentation

## Implementation Status

All backlog items are represented in the initial expanded codebase. Some are deep product workflows, such as enrollments, grading, quiz attempts, and certificates. Others are production-shaped scaffolds that show where a real implementation would continue, such as SCORM import, video processing, public API keys, and CSV export workers.

## Implementation status by item (checked against this snapshot)

"Represented in the codebase" covers a wide range. This table sorts each item into **workflow** (an API route does the job end to end), **partial** (some code path exists with notable gaps) or **schema only** (a model and maybe seed data, no behaviour). Details are in each feature document's "As built" section.

| Item | Status | Feature doc |
|---|---|---|
| Course versioning | partial — versions and publish exist; enrollment ignores `currentVersionId` | 01 |
| Scheduled cohorts | schema only | 02 |
| SCORM-like packages | schema only | 05 |
| Video processing pipeline | partial — worker exists, nothing enqueues | 06 |
| Search, filtering, pagination | workflow (API); UI uses demo data | 11 |
| Prerequisites and learning paths | schema only | 08 |
| Grading workflows | workflow, without organization check or audit | 04 |
| Assignment rubrics | schema only for grading; rubrics are displayed, not scored | 15 |
| Question banks | schema only; attempts draw from the quiz's questions | 16 |
| Randomized attempts | workflow | 16 |
| Retake policies | workflow (count-then-insert) | 17 |
| Due dates and late submissions | schema only (dates stored, never read) | 17 |
| Completion rules | partial — helpers exist; the route uses its own logic | 17 |
| Discussion forums | read-only include; no posting | 03 |
| Notifications and reminders | partial — worker exists, nothing enqueues | 09 |
| Learner transcript | workflow for instructors/admins, without organization filter | 12 |
| Certificate verification | API workflow; the web page uses demo data | 13 |
| Instructor feedback | workflow (one text field) | 14 |
| Organization reporting | workflow | 07 |
| Public API | partial — keys can be created; nothing accepts them | 10 |
| Bulk learner import | workflow with gaps (existing users, shared password) | 18 |
| CSV exports | partial — request recorded, never generated | 18 |
| Admin impersonation | partial — session record only | 19 |
| Feature flags | partial — stored; evaluation helper unused | 19 |
| Domain events, outbox | partial — written outside transactions; never dispatched | 20 |
| Job retry policies | configured on queues | 20 |
| Observability, health | partial — database health only | 20 |
| API contracts | static summary only | 20 |
| E2E tests | none | 21 |
| Migration discipline | `prisma migrate dev` script; no migrations folder | 21 |
| Final design documentation | done (`docs/architecture/final-system-design.md`) | 21 |

## Exercise

**Goal:** choose the next three items to build so that the most "partial" rows become "workflow" with the least new code.

**Check:** a reasonable answer starts with one change that unblocks four rows — producing jobs from the API (video, notifications, outbox, CSV exports) — and explains which queue each route should add to.
