# API Specification

This specification defines the planned HTTP interface, authorization rules, response formats, and error behavior. All public routes will be exposed through API Gateway.

[System architecture](system-architecture.md) · [Kafka topics and events](kafka-topics-and-events.md) · [Database design](database-design.md)

## Common rules

- Base path: `/api/v1`; JSON request and response bodies except multipart post creation. `Authorization: Bearer <access-token>` on protected routes.
- User identity comes from JWT `sub`, never a client supplied `userId` header or body field. Gateway removes inbound `X-User-Id` and injects a validated one for trusted internal calls.
- Use `201` for created resources, `200` for reads and state transitions, `204` for successful deletion, `400` for invalid input, `401` for absent or invalid token, `403` for wrong actor, `404` for missing resource, and `409` for duplicate or invalid state.
- Error body: `{"code":"CONNECTION_ALREADY_EXISTS","message":"Already connected","requestId":"..."}`. Do not expose internal exceptions. `requestId` is useful once gateway logging is added.
- Paginate collection endpoints with `?limit=20&cursor=...`, bounded maximum 100. Timestamps use UTC ISO 8601. Numeric IDs are represented as JSON numbers while they fit the chosen client platform; reconsider strings before JavaScript precision becomes a risk.

## User Service

| Method/path | Auth | Request | Success |
| --- | --- | --- | --- |
| `POST /api/v1/users/auth/signup` | Public | `{"name":"Asha Rao","email":"asha@example.com","password":"..."}` | `201` `{"id":42,"name":"Asha Rao","email":"asha@example.com"}` |
| `POST /api/v1/users/auth/login` | Public | `{"email":"asha@example.com","password":"..."}` | `200` `{"accessToken":"...","tokenType":"Bearer","expiresInSeconds":3600}` |
| `GET /api/v1/users/me` | JWT | None | `200` own `id`, `name`, `email` |
| `GET /api/v1/users/{userId}` | JWT | None | `200` public profile fields only; later milestone |

Normalize email, enforce uniqueness in PostgreSQL, hash passwords with an adaptive password encoder, and validate input lengths. Login returns a generic failure for an unknown email or wrong password.

## Connections Service

| Method/path | Auth | Request | Success and rule |
| --- | --- | --- | --- |
| `POST /api/v1/connections/requests` | JWT | `{"recipientUserId":73}` | `201` `{"id":"42:73","status":"PENDING","requesterUserId":42,"recipientUserId":73}` |
| `GET /api/v1/connections/requests?direction=incoming` | JWT | Optional cursor | `200` paginated pending requests for authenticated recipient |
| `POST /api/v1/connections/requests/{id}/accept` | JWT | None | `200` status `ACCEPTED`; recipient only |
| `POST /api/v1/connections/requests/{id}/reject` | JWT | None | `200` status `REJECTED`; recipient only |
| `GET /api/v1/connections/users/{userId}/first-degree` | JWT | Optional cursor | `200` paginated `[{"userId":73,"name":"..."}]` |

The request ID in this plan is the canonical pair key; a separate opaque ID may be chosen at implementation time, but use one convention across routes and persistence. Self-request is `400`. Same-direction pending, reverse pending, already connected, and rejected pair under the MVP no-reopen policy are `409`. Missing user or pair is `404`. A caller cannot accept their own outbound request.

## Posts Service

| Method/path | Auth | Request | Success |
| --- | --- | --- | --- |
| `POST /api/v1/posts` | JWT | `multipart/form-data`: `post` JSON part `{"content":"..."}`, optional `file` image part | `201` post DTO |
| `GET /api/v1/posts/{postId}` | JWT | None | `200` post DTO |
| `GET /api/v1/posts/users/{userId}` | JWT | Optional cursor | `200` paginated posts by author |
| `POST /api/v1/posts/{postId}/likes` | JWT | None | `201` like DTO or `204`; choose one before coding |
| `DELETE /api/v1/posts/{postId}/likes/me` | JWT | None | `204` |

Post DTO: `{"id":901,"authorUserId":42,"content":"Started a new role","mediaUrl":"https://...","createdAt":"2026-09-30T10:05:00Z","likeCount":1}`. `likeCount` is optional in the first slice; if provided, derive it from Posts-owned data. Require nonblank bounded content (or explicitly allow image-only posts), MIME type allowlist and size limit for images. Uploader validates the file again. Like uniqueness is enforced in the database; a second like returns `409` in the MVP. Unlike by a user who has not liked may return `204` idempotently or `404`; settle the policy before implementation.

## Notification Service

| Method/path | Auth | Request | Success |
| --- | --- | --- | --- |
| `GET /api/v1/notifications` | JWT | Optional cursor and `unreadOnly=true` | `200` own notifications only, newest first |
| `PATCH /api/v1/notifications/{id}/read` | JWT | None | `200` notification with `readAt` |

Only the authenticated recipient can read or mark a notification. No email, SMS, or push delivery is required for MVP.

## Uploader Service and infrastructure

`POST /uploads/file` is an **internal** multipart endpoint called by Posts Service; it returns a media URL or storage key. Do not publish it as a general client upload API until authorization, quotas, validation, and object lifecycle are designed. Discovery Server has service registry endpoints for infrastructure, not business API contracts. Health and readiness endpoints should be internal or operational routes with appropriate exposure.

## Example protected call

```http
POST /api/v1/connections/requests HTTP/1.1
Authorization: Bearer <access-token>
Content-Type: application/json

{"recipientUserId":73}
```

The authenticated user is requester `42` if the validated token subject is `42`; the body cannot override it.
