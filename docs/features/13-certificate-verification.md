# Feature: Certificate Verification Page

## Goal

Allow external verification of issued certificates by verification code.

## Impacted Areas of the Codebase

- Prisma model: `Certificate`.
- Certificate route: `GET /certificates/verify/:verificationCode`.
- Next.js verification page.

## Implementation Approach

Certificates store status, artifact URL, issue date, and unique verification code.

## Design Considerations

Certificates are artifacts, not booleans. They can be revoked, verified, and rendered.

## Build Journal

Implemented certificate verification API and frontend page.

A junior developer should learn to model generated documents as first-class records.

## As built (checked against this snapshot)

- `Certificate` (`apps/api/prisma/schema.prisma:344-359`): unique `enrollmentId`, unique `verificationCode`, `status` (`ISSUED | REVOKED`), `revokedAt`, `artifactUrl`.
- Issued by the completion route with code `CERT-` + the last 8 characters of the enrollment ID (`apps/api/src/routes/learner.routes.ts:245-254`).
- `GET /certificates/verify/:verificationCode` (`apps/api/src/routes/certificate.routes.ts`) is public and returns course title, learner name, issue date and status.

**Gaps.** The web verification page never calls that API: it looks the code up in demo data and falls back to the first demo certificate, always showing "Verified" (`apps/web/app/certificates/verify/[code]/page.tsx:7`, `:17-19`). Nothing can revoke a certificate. The code is derived from the database ID rather than random, and the public endpoint has no rate limiting.

**Exercise.** *Goal:* make the page call `GET /certificates/verify/:code` and render "Not found" on 404 and "Revoked" for `REVOKED`. *Check:* `/certificates/verify/NOPE` shows "Not found", not "Verified".
