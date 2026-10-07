# Feature: Admin Impersonation and Feature Flags

## Goal

Add enterprise support tooling and controlled rollout mechanisms.

## Impacted Areas of the Codebase

- Prisma models: `ImpersonationSession`, `FeatureFlag`.
- Admin routes.
- Domain helper: `isFeatureEnabled`.

## Implementation Approach

Impersonation sessions record actor, target, reason, and time. Feature flags store organization-scoped rollout settings.

## Design Considerations

Impersonation is powerful and must be auditable. Feature flags should be deterministic per subject.

## Build Journal

Implemented session storage, feature flag route, and deterministic flag helper.

A junior developer should learn that operational power needs guardrails.

## As built (checked against this snapshot)

- `POST /admin/impersonation` (`apps/api/src/routes/admin.routes.ts:193-207`) stores actor, target and reason in `ImpersonationSession`.
- `POST /admin/feature-flags` (`:170-191`) upserts a flag per `(organizationId, key)` with optional `rolloutPercent`; the seed creates `advanced-reporting` (100%) and `video-processing` (50%).
- `isFeatureEnabled` (`packages/domain/src/feature-flags.ts:7-21`) buckets `subjectId + key` with a simple hash into 0-99.

**Gaps.** Impersonation only writes a record: it does not issue a token, does not check that the target is in the organization, and writes no audit log; `canImpersonate` is unused. `isFeatureEnabled` is not called anywhere outside tests, so flags have no effect. Flag changes are not audited.

**Exercise.** *Goal:* write an `impersonation.started` audit row and reject targets without a membership in the header organization. *Check:* impersonating a user ID from another organization returns 404, and a valid request shows up in `GET /admin/audit-logs`.
