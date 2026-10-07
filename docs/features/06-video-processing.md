# Feature: Video Processing Mock Pipeline

## Goal

Show how uploaded training videos would move through asynchronous processing.

## Impacted Areas of the Codebase

- Prisma model: `VideoAsset`.
- Worker queue: `video-processing`.
- Lesson model relation.

## Implementation Approach

Videos start as `UPLOADED`, the worker marks them `READY`, and lessons can reference a video asset.

## Design Considerations

Real encoding would involve object storage, callbacks, and multiple renditions. The mock pipeline teaches the boundary.

## Build Journal

Implemented `VideoAsset` and a BullMQ worker processor with retry policy.

A junior developer should learn that slow media work belongs outside request/response paths.

## As built (checked against this snapshot)

- `VideoAsset` (`apps/api/prisma/schema.prisma:430-438`) with status `UPLOADED | PROCESSING | READY | FAILED` (`VideoStatus`, `:62-67`). It has no `organizationId`.
- `videoQueue` with 3 attempts and exponential backoff (`apps/worker/src/queues.ts:27-33`); the `video-processing` worker sets `READY` and a fixed `durationSeconds: 420` (`apps/worker/src/worker.ts:37-50`).

**Gaps.** Nothing creates a `VideoAsset` or enqueues a `video-processing` job (no code calls `.add(...)`; the API does not import the worker's queues). The worker would also re-mark an already `READY` asset without complaint.

**Exercise.** *Goal:* add an authoring route that creates a `VideoAsset` and enqueues `{ videoAssetId }` on `video-processing`. *Check:* with the worker running, the asset goes from `UPLOADED` to `READY` with `durationSeconds` 420 within a few seconds.
