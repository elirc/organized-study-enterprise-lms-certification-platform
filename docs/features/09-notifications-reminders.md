# Feature: Notifications and Reminders

## Goal

Schedule reminders for due training and other learner events.

## Impacted Areas of the Codebase

- Prisma model: `Notification`.
- Worker queue: `notification-delivery`.
- Seed data.

## Implementation Approach

Notifications store channel, payload, schedule, and status. The worker marks a notification sent.

## Design Considerations

Payloads are JSON for flexibility, but production systems often add templates and delivery-provider records.

## Build Journal

Implemented notification persistence and worker processing.

A junior developer should learn that reminders should be retryable and observable.

## As built (checked against this snapshot)

- `Notification` (`apps/api/prisma/schema.prisma:450-463`) with channel, JSON payload, `scheduledFor`, status and `sentAt`; one seeded notification (`apps/api/prisma/seed.ts:262`).
- `notificationQueue`: 5 attempts, exponential backoff from 30 s (`apps/worker/src/queues.ts:19-25`); the worker marks the notification `SENT` (`apps/worker/src/worker.ts:22-35`).

**Gaps.** Nothing enqueues `notification-delivery` jobs and nothing reads `scheduledFor`, so the seeded notification stays unsent. The worker does not check whether the notification was already sent, so a retried job "sends" twice.

**Exercise.** *Goal:* add a repeatable worker job that finds notifications with `status: "SCHEDULED"` (`NotificationStatus`, `schema.prisma:49-53`) and `scheduledFor <= now`, and enqueues one delivery job per row with `jobId` set to the notification ID. *Check:* the seeded notification becomes `SENT` once, and running the scan again does not enqueue it a second time.
