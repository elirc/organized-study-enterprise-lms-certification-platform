# Feature: Events, Outbox, Observability, and API Contracts

## Goal

Make the system easier to integrate, observe, and evolve.

## Impacted Areas of the Codebase

- Prisma models: `DomainEvent`, `OutboxMessage`.
- API helpers: `recordDomainEvent`.
- Worker: `outbox-dispatch`.
- Platform routes: health and OpenAPI summary.

## Implementation Approach

Request handlers record domain events and outbox messages. The worker dispatches pending messages. Health and API summary routes expose operational state.

## Design Considerations

The outbox pattern avoids losing events when database writes and background dispatch need to stay consistent.

## Build Journal

Implemented domain event recording, outbox messages, retry-aware worker processing, and platform routes.

A junior developer should learn that production-shaped systems need operational surfaces, not only business endpoints.

## As built (checked against this snapshot)

- `recordDomainEvent` (`apps/api/src/events.ts:4-25`) creates a `DomainEvent` and an `OutboxMessage` as two separate writes, outside any transaction with the business change. Callers: enrollment (`learner.routes.ts:87-91`) and CSV export (`admin.routes.ts:160-164`).
- The outbox worker marks up to 25 `PENDING` messages `SENT` when an `outbox-dispatch` job runs (`apps/worker/src/worker.ts:52-73`).
- `GET /platform/health` runs `SELECT 1` (`apps/api/src/routes/platform.routes.ts`); `GET /platform/openapi-summary` returns a fixed list of resource names.

**Gaps.** Because the event is written after (and separately from) the enrollment, a crash between them loses the event — the guarantee the outbox pattern exists for. Nothing enqueues `outbox-dispatch`, so messages are never marked sent, and "sent" means only a status change; nothing is delivered. The OpenAPI summary is a static string list, not a contract. Health does not check Redis.

**Exercise.** *Goal:* pass a transaction client into `recordDomainEvent` and call it inside a `prisma.$transaction` together with the enrollment writes. *Check:* making the outbox insert throw (temporarily) leaves no enrollment row behind.
