# Database Design and Data Ownership

The persistence design assigns each service its own data, constraints, and transaction boundaries. PostgreSQL will store accounts, posts, likes, and notifications; Neo4j will represent members and connection state.

[System architecture](system-architecture.md) · [API specification](api-specification.md) · [Kafka topics and events](kafka-topics-and-events.md)

## Identity and ownership

User Service creates the canonical numeric `userId` (a PostgreSQL `BIGINT`). The same value is a reference in Posts, Notification, and Neo4j; their local record IDs are separate. Services never generate another user ID for the same member and never write another service's store. There are no cross-database foreign keys: validate target users through a User Service lookup or a reliable local projection when needed. A `user.created.v1` event creates the Connections projection asynchronously, so a newly registered member may briefly lack a graph node. Handle this as a retryable condition and reconcile missing nodes. JWT `sub` contains the decimal string of that ID; API and event examples represent it as a number. All representations identify the same member.

## PostgreSQL

One PostgreSQL instance with service-owned schemas is a practical local start; separate databases can follow without changing ownership. Use migrations rather than automatic schema updates outside disposable tests. All timestamps are UTC.

| Owner/schema | Table | Essential fields and constraints |
| --- | --- | --- |
| User / `user` | `users` | `id BIGINT PK`, `email` unique and normalized, `name`, `password_hash`, `created_at`, `updated_at` |
| Posts / `posts` | `posts` | `id BIGINT PK`, `author_user_id`, `content`, nullable `media_url` or `media_key`, `created_at`; index `(author_user_id, created_at DESC)` |
| Posts / `posts` | `post_likes` | `id BIGINT PK`, `post_id` FK within Posts schema, `user_id`, `created_at`, unique `(post_id, user_id)` |
| Notification / `notification` | `notifications` | `id BIGINT PK`, `recipient_user_id`, `type`, `actor_user_id`, nullable `post_id`, `message`, `read_at`, `created_at`, unique `event_id`; index `(recipient_user_id, created_at DESC)` |
| Notification / `notification` | `processed_events` | `event_id` PK, `processed_at`; optional if `notifications.event_id` alone covers every consumed event |

Email and password data stay in User Service. Posts may retain a URL or provider object key, not image bytes. Notification stores a stable reference and short display text; it should not copy entire post content. An optional `outbox_events` table belongs in each publishing service in a later reliability phase, with event ID, payload, created time, and publish status. A cleanup or retention policy is needed after implementation.

## Neo4j

The graph will connect two Person nodes through one ConnectionPair node. This makes the unique member pair and its request state explicit:

```text
(:Person {userId, name})-[:PARTICIPATES_IN]->
(:ConnectionPair {pairKey, requesterUserId, recipientUserId,
                   status, createdAt, updatedAt})
<-[:PARTICIPATES_IN]-(:Person {userId, name})
```

`Person.userId` is unique. `ConnectionPair.pairKey` is unique and is built from `min(userIdA,userIdB) + ":" + max(...)`. A pair node is authoritative for one unordered pair. Querying an accepted pair returns the other participant. The pair node provides a single place to enforce request state and participant identity. A relational `connection_pairs` table is also viable for first-degree queries and would simplify operations. Neo4j supports the project's graph-modeling objective and future relationship traversal; performance comparisons will require measured workloads.

### Connection state and concurrency

| Current | Command | Result |
| --- | --- | --- |
| No pair | A sends to B | `PENDING`, requester A, recipient B |
| `PENDING` A→B | A sends again | `409` duplicate; no new pair |
| `PENDING` A→B | B sends to A | `409` reverse pending; UI may suggest accepting existing request |
| `PENDING` A→B | B accepts | `ACCEPTED`; both become first-degree connections |
| `PENDING` A→B | B rejects | `REJECTED`; pair remains as history |
| `PENDING` A→B | A withdraws | `WITHDRAWN` (later feature) |
| `ACCEPTED` | Either sends again | `409` already connected |
| `REJECTED` | Either sends later | Optional reopen policy; MVP returns `409` until a deliberate reset endpoint exists |

Reject self-requests before database access. Require both Person nodes to exist. Only the recipient may accept or reject. State transitions and pair creation run in Neo4j transactions; the unique pair key makes concurrent opposite-direction creates conflict, after which the losing request re-reads and returns a deterministic `409`. Acceptance is idempotent only if the same accepted request is repeated with the same ID; an unrelated request cannot change it. Avoid check-then-create without a constraint.

First-degree query, conceptually:

```cypher
MATCH (me:Person {userId: $userId})-[:PARTICIPATES_IN]->
      (pair:ConnectionPair {status: 'ACCEPTED'})<-[:PARTICIPATES_IN]-(other:Person)
WHERE other.userId <> $userId
RETURN DISTINCT other.userId AS userId, other.name AS name
ORDER BY name, userId
```

Return IDs for internal recipient selection and a paginated DTO for public reads. Keep `name` as a projection refreshed from User Service; User Service remains the source of truth. Degree two suggestions, blocking, removal, and privacy controls are later work.

## Data consistency rules

- User registration commits before a `user.created` event can be consumed. Without an outbox, a crash between save and publish can leave the graph projection missing; a repair job or outbox closes that gap.
- Post likes are unique by `(post_id,user_id)`, so concurrent double likes cannot duplicate rows or events. Publish `post.liked` only after the first successful insert.
- Notification inserts use `event_id` uniqueness, so Kafka redelivery cannot create a second notification. A consumer commits offsets only after its database transaction succeeds.
- A post remains readable if notification generation is delayed. The API should not promise immediate notification delivery.
