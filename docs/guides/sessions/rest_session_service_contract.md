# REST session service contract

Status: draft, for review before implementation.

`RestSessionService` stores ADK sessions through an HTTP service that you run. The agent never connects to the session database. It calls the REST API described here, and your service owns storage, authentication, tenancy and retention. Any backend that implements this contract works, whatever it stores sessions in.

This document is the contract between `RestSessionService` (the client) and that backend (the server). It uses the same paths as the session routes of the ADK dev server, so a backend can serve both, and adds one route that the dev server lacks: appending an event.

## Conventions

- All request and response bodies are JSON (`Content-Type: application/json`).
- `baseUrl` is the client's configured root, such as `https://sessions.example.com/v1`. Paths below are relative to it, with no trailing slash.
- Path segments (`appName`, `userId`, `sessionId`) are URL-encoded by the client.
- The client sends the headers it is configured with on every request. The contract does not define any authentication header. The server decides which headers it needs, such as `Authorization` or a tenant header.
- Error bodies are `{"error": "<message>"}`. The client does not parse the message beyond logging the status, so servers should not put secrets in it.
- Timestamps are numbers, in the same unit ADK uses for `Session.lastUpdateTime` and `Event.timestamp`.

## Data shapes

### Session

The shape of the `Session` interface in `core/src/sessions/session.ts`:

```json
{
  "id": "string",
  "appName": "string",
  "userId": "string",
  "state": {"key": "value"},
  "events": [],
  "lastUpdateTime": 1730000000000
}
```

- `state` is the merged view. Keys with the `app:` prefix are shared by every session of the app, keys with the `user:` prefix are shared by every session of the user, and all other keys belong to this session. The server stores each scope separately and returns them merged, with the prefixes in the key names. This is the same scoping `DatabaseSessionService` applies.
- `temp:` keys are never sent to the server. The client strips them before any request.
- `events` is ordered by append time, oldest first.

### Event

The serialized form of the `Event` interface in `core/src/events/event.ts`. The server should store it opaquely and return it unchanged, so that the client can round-trip fields the server does not understand. The server needs to read only these fields:

| Field | Use |
|---|---|
| `id` | Idempotency key. See `POST .../events`. |
| `timestamp` | Filtering with `afterTimestamp`. |
| `actions.stateDelta` | Applied to the stored state. Keys with `app:` and `user:` prefixes go to the matching scope. |

## Endpoints

### Create a session

```
POST /apps/{appName}/users/{userId}/sessions
POST /apps/{appName}/users/{userId}/sessions/{sessionId}
```

Request body (optional):

```json
{"state": {"key": "value"}}
```

The first form lets the server generate the session id. The second creates the session with the id the caller chose.

| Status | Meaning |
|---|---|
| `200` | Created. The body is the new `Session`. |
| `409` | A session with this id already exists. The body is `{"error": ...}`. |

The client treats `409` on the second form as "already exists" and loads the existing session. A server that cannot distinguish the cases may return `200` with the existing session instead.

### Get a session

```
GET /apps/{appName}/users/{userId}/sessions/{sessionId}
```

Query parameters, both optional:

| Parameter | Meaning |
|---|---|
| `numRecentEvents` | Return only the last N events. |
| `afterTimestamp` | Return only events with `timestamp` at or after this value. |

If both are given, the server applies `afterTimestamp` first, then keeps the last `numRecentEvents`.

| Status | Meaning |
|---|---|
| `200` | The `Session`. |
| `404` | No such session. The client returns `undefined`. |

### List sessions

```
GET /apps/{appName}/users/{userId}/sessions
```

Query parameters, all optional. They map to `ListSessionsRequest`:

| Parameter | Meaning |
|---|---|
| `limit` | Maximum number of sessions to return. |
| `offset` | Zero-based index of the first session. Ignored if `page` is set. |
| `page` | 1-based page number. Requires `limit`. |
| `order` | `asc` or `desc`, by `lastUpdateTime`. No ordering if omitted. |

Response `200`, the shape of `ListSessionsResponse`:

```json
{
  "sessions": [],
  "page": 1,
  "limit": 20,
  "totalItems": 1,
  "totalPages": 1
}
```

Each session in the list has `state` and `events` empty or omitted.

### Append an event

```
POST /apps/{appName}/users/{userId}/sessions/{sessionId}/events
```

Request body: one `Event`.

On success the server:

1. Stores the event.
2. Applies `actions.stateDelta` to the stored state, scoped by key prefix.
3. Updates the session's `lastUpdateTime`.

Response `200`:

```json
{"lastUpdateTime": 1730000000000}
```

| Status | Meaning |
|---|---|
| `200` | Stored. |
| `404` | No such session. |
| `409` | The session changed since the client last read it. Optional. See below. |

Rules:

- **Idempotent by `event.id`.** Appending an event whose `id` is already stored replaces that event and does not apply its `stateDelta` a second time. This makes retries safe.
- **Ordered.** The client sends events for one session one at a time, in the order they were produced. The server applies state deltas in arrival order.
- Partial (streaming) events are never sent.

`409` is optional. A server that wants optimistic concurrency can return it when the session was modified by another writer, and the client surfaces it as the existing stale-session error (`stale_session_error.ts`). A server without that need never returns it.

### Delete a session

```
DELETE /apps/{appName}/users/{userId}/sessions/{sessionId}
```

| Status | Meaning |
|---|---|
| `204` | Deleted. |
| `404` | No such session. The client treats this as success. |

## Client behavior on errors

| Condition | Default behavior |
|---|---|
| `408`, `429`, `5xx`, network error, timeout | Retried with backoff, then the call throws. |
| Other `4xx` | The call throws immediately. |
| `404` on get | Returns `undefined`. |
| `404` on delete | Succeeds. |

An opt-in mode can instead log the failure, queue the operation per session and continue the turn. Queued operations are sent in order on the next call. That mode trades durability for availability, so it is off by default.

## Out of scope

- Authentication and authorization. Supply headers through the client's `headers` option.
- Artifacts and memory. They have their own services.
- Server-side pagination limits, rate limits and retention.
