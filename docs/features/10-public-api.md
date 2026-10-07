# Feature: Public API for Enterprise Customers

## Goal

Prepare for enterprise integrations using organization-scoped API keys.

## Impacted Areas of the Codebase

- Prisma model: `ApiKey`.
- Admin route: `POST /admin/api-keys`.
- Settings UI.

## Implementation Approach

API keys store a prefix and hash, while the raw key is returned only once.

## Design Considerations

Keys are credentials. They should not be stored in plaintext.

## Build Journal

Implemented API key storage and generation route.

A junior developer should learn why secrets are hashed and why prefixes are useful for lookup and support.

## As built (checked against this snapshot)

- `ApiKey` (`apps/api/prisma/schema.prisma:464-473`): `name`, `hashedKey`, `prefix`, `lastUsedAt`.
- `POST /admin/api-keys` (`apps/api/src/routes/admin.routes.ts:251-269`), for `ORG_ADMIN`/`ADMIN`, generates `lms_<32 hex>`, stores a bcrypt hash and the first 8 characters as `prefix`, and returns the raw key once.

**Gaps.** No route accepts an API key, so the "public API" has no entry point; `lastUsedAt` is never set. The prefix is `lms_` plus only four random characters, so it identifies very little. bcrypt is a slow password hash; for long random keys a fast keyed hash (HMAC-SHA-256) is the usual choice because every API request has to verify it. Creating a key writes no audit log.

**Exercise.** *Goal:* add middleware that reads `Authorization: ApiKey <raw>`, finds candidates by prefix, verifies the hash, sets `lastUsedAt`, and scopes the request to the key's organization; use it on a read-only `GET /public/courses`. *Check:* a freshly created key returns the organization's published courses; a modified key returns 401; `lastUsedAt` is set after the first call.
