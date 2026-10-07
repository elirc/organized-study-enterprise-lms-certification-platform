# Feature: Discussion Forums

## Goal

Provide course and lesson discussion threads for learner and instructor collaboration.

## Impacted Areas of the Codebase

- Prisma models: `DiscussionThread`, `DiscussionPost`.
- Course detail API.
- Course workspace UI.

## Implementation Approach

Threads attach to course versions and optionally lessons. Posts attach to threads and authors.

## Design Considerations

Versioned discussions let a changed course keep separate conversation context.

## Build Journal

Implemented discussion models, seeded a thread, exposed discussions in the catalog detail include, and rendered them in the course workspace.

A junior developer should learn that collaboration features often need moderation, permissions, and pagination later.

## As built (checked against this snapshot)

- `DiscussionThread` and `DiscussionPost` exist in the schema (`apps/api/prisma/schema.prisma:372-392`); the seed creates one thread (`apps/api/prisma/seed.ts:221`).
- Read path: `GET /catalog/courses/:courseId` includes every version's threads with posts and **full author records** (`apps/api/src/routes/catalog.routes.ts:82`).
- The course page renders threads from `apps/web/lib/demo-data.ts` (`apps/web/app/course/[id]/page.tsx:61`), not from the API.

**Gaps.** There is no route to create a thread or post. Including `author: true` returns the whole `User` row — including `passwordHash` — to anyone who can read the catalog detail. That is the most serious finding in this area.

**Exercise.** *Goal:* replace `author: true` with `author: { select: { id: true, name: true } }`. *Check:* the catalog detail response no longer contains a `passwordHash` key anywhere (search the JSON).
