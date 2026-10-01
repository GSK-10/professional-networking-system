# Kafka Topics and Event Contracts

Kafka will connect account creation to the connection graph and deliver post activity to Notification Service. This document defines topic ownership, message payloads, recipient selection, and delivery rules.

[System architecture](system-architecture.md) · [Database design](database-design.md) · [API specification](api-specification.md)

## Topic map

| Topic | Producer | Consumer | Key | Purpose |
| --- | --- | --- | --- | --- |
| `user.created.v1` | User Service | Connections Service | `userId` | Create or update Neo4j Person projection |
| `post.created.v1` | Posts Service | Notification Service | `recipientUserId` | One notification intent per first-degree connection at creation time |
| `post.liked.v1` | Posts Service | Notification Service | `ownerUserId` | Notify the post owner when another member likes a post |

Topic names carry a version suffix. Producers will supply the keys listed above so related records are assigned consistently to partitions. Ordering is limited to a partition; consumers must not rely on ordering across topics.

## Common envelope

Every payload has `eventId` (UUID), `eventType`, `schemaVersion: 1`, `occurredAt` (UTC ISO 8601), `producer`, and `data`. Never put passwords, JWTs, or private media credentials in events. Consumers ignore unknown optional fields, reject unsupported major versions, and preserve the original event ID on retries.

### `user.created.v1`

```json
{
  "eventId": "c8d38536-8e0a-45da-b5ca-23b74e8e9258",
  "eventType": "user.created",
  "schemaVersion": 1,
  "occurredAt": "2026-09-30T10:00:00Z",
  "producer": "user-service",
  "data": { "userId": 42, "name": "Asha Rao" }
}
```

Connections upserts `Person` by unique `userId`. A duplicate event updates the same projection rather than creating another node. Email is excluded.

### `post.created.v1`

```json
{
  "eventId": "e8b4127d-f738-44f2-9d8a-1dc3d53617f2",
  "eventType": "post.created",
  "schemaVersion": 1,
  "occurredAt": "2026-09-30T10:05:00Z",
  "producer": "posts-service",
  "data": {
    "postId": 901,
    "authorUserId": 42,
    "recipientUserId": 73,
    "preview": "Started a new role"
  }
}
```

Posts asks Connections for first-degree IDs after saving the post and publishes one event per recipient, with a distinct `eventId` for each recipient. The recipient set is a snapshot at creation time: later connections do not get old post notifications. A future alternative is one post event plus recipient fan-out in Notification Service, but that moves graph lookup and failure handling there. `preview` is optional and length limited.

### `post.liked.v1`

```json
{
  "eventId": "7239db0c-60cc-4bcc-8c9c-74b293cdbbb2",
  "eventType": "post.liked",
  "schemaVersion": 1,
  "occurredAt": "2026-09-30T10:06:00Z",
  "producer": "posts-service",
  "data": { "postId": 901, "ownerUserId": 42, "likedByUserId": 73 }
}
```

Posts publishes only after a new `(postId, likedByUserId)` row is inserted. Notification chooses `ownerUserId` as recipient and skips a self-like. Unlike emits no event in the MVP; removing an already delivered like notification is a later product decision.

## Delivery, retries, and failure policy

Kafka may redeliver records. Each consumer handles a record in a local database transaction and records its `eventId` with a unique constraint. On duplicate delivery it returns success without repeating the effect, then commits the Kafka offset. Malformed or repeatedly failing messages should have bounded retries with backoff, then a dead-letter topic and an alert in a later reliability phase; record enough context to replay safely. Do not silently skip permanent errors.

A database commit and a Kafka send are separate operations: a crash between them can lose publication, and retries can duplicate delivery. The event integration milestone will log publication failures and use duplicate-safe consumers. The reliability milestone will add a transactional outbox to User and Posts and a publisher worker to recover pending messages. A retry must reuse the stored outbox `eventId`; generating a fresh ID would defeat deduplication. Outbox delivery plus consumer deduplication provides at-least-once processing with effectively one stored notification, not globally exactly-once execution.
