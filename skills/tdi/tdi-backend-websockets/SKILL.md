---
name: tdi-backend-websockets
description: Design and implement authenticated backend WebSocket endpoints in TDI using Django Channels, module sock_urls discovery, lifecycle-safe consumers, and an endpoint-appropriate direct, polling, or channel-layer delivery strategy. Use when adding or changing server-side WebSocket streaming in a pip-ade-* package.
---

# TDI Backend WebSockets

## Goal

Build backend WebSocket endpoints that share TDI's routing, authentication, message-contract, and connection-lifecycle conventions while allowing each endpoint to choose the delivery mechanism appropriate to its producer.

Also load `/skill:tdi-backend-task` for normal submodule and validation rules. Load `/skill:tdi-frontend-websockets` when implementing the browser side too.

## Existing architecture

TDI already provides the WebSocket transport layer:

- `bin/asgi.py` routes `websocket` connections through `AllowedHostsOriginValidator`, `AuthMiddlewareStack`, and `core.sock_urls`.
- `src/pip-ade-core/src/core/sock_urls.py` discovers `<installed_app>.sock_urls` and mounts it below `/sock/<app>/`.
- A backend package owns its consumers under `<app>/sock/` and exposes them from `<app>/sock_urls.py`.
- The reference direct-stream implementation is `src/pip-ade-ruckus/src/ruckus/sock/shell.py` with `src/pip-ade-ruckus/src/ruckus/sock_urls.py`.

The resulting convention is:

```text
/sock/<django-app>/<feature>/<resource identifiers>/
```

Do not add feature WebSocket routes directly to `bin/asgi.py`. Keep routing and consumers in the owning Python package.

## Standard package layout

```text
src/pip-ade-<domain>/src/<domain>/
├── sock/
│   ├── __init__.py
│   └── <feature>.py
└── sock_urls.py
```

Example route:

```python
from django.urls import path

from .sock import TaskConsumer


urlpatterns = [
    path(
        'parent/<str:parent_id>/task/<str:task_id>/',
        TaskConsumer.as_asgi()
    )
]
```

With app name `example`, this is available at:

```text
/sock/example/parent/<parent_id>/task/<task_id>/
```

Use nested resource identifiers when they help enforce ownership or parent-child relationships.

## Consumer structure

TDI's established consumer style uses Channels `AsyncConsumer` and ASGI event dictionaries:

- `websocket_connect`
- `websocket_receive` when client messages are meaningful
- `websocket_disconnect`
- `self.send({'type': 'websocket.accept'})`
- `self.send({'type': 'websocket.send', 'text': ...})`
- `self.send({'type': 'websocket.close', 'code': ...})`

A generic Channels consumer is also acceptable if it materially simplifies JSON handling, but preserve the same routing, security, and lifecycle behavior as neighboring TDI consumers.

## Required connection flow

1. Read `self.scope['user']`, populated by `AuthMiddlewareStack`.
2. Reject unauthenticated users before accepting.
3. Parse resource identifiers from `self.scope['url_route']['kwargs']`.
4. Load the requested resource.
5. Enforce object authorization and parent-child ownership explicitly.
6. Accept only after authentication and authorization succeed.
7. Send an initial snapshot or readiness event.
8. Start the endpoint-specific delivery mechanism.
9. Cancel background work and release resources on disconnect.
10. Send a final terminal snapshot before a normal close when the stream has a natural terminal state.

WebSocket consumers do not automatically execute DRF view permissions. Mirror the REST endpoint's relevant RBAC/object checks instead of assuming that an authenticated user may access every resource.

Suggested application close codes:

- `4401`: unauthenticated
- `4403`: forbidden or parent/resource mismatch
- `4404`: resource not found
- `1000`: normal/terminal completion
- `1011`: unexpected backend failure

Keep close-code behavior consistent with the frontend reconnect policy.

## Choose the delivery strategy per endpoint

All TDI WebSockets should use the common route, authentication, envelope, cleanup, and frontend lifecycle structure. They do not all need the same event source.

### 1. Direct producer owned by the consumer

Use when the consumer directly owns or reads the source, such as a shell process or remote socket.

```text
producer -> consumer -> WebSocket
```

Examples:

- PTY output
- a remote device stream
- a subprocess created by the consumer

Forward output as it arrives. Do not add datastore polling or a channel layer when the producer is already available in the consumer process.

### 2. Poll a durable source of truth

Use when another process updates durable state, there is no configured cross-process channel layer, traffic is modest, and update latency of several seconds is acceptable.

```text
worker -> Elasticsearch/database
                    ^
                    | periodic reload
             ASGI consumer -> WebSocket
```

Polling rules:

- Send the first snapshot immediately.
- Use an endpoint-appropriate interval; do not assume every endpoint needs the same interval.
- Bypass application caches when freshness is required.
- Run synchronous Elasticsearch/database calls and serialization through `sync_to_async`; never block the ASGI event loop.
- Compare a stable revision, `updated_ts`, or serialized snapshot and send only changes.
- Expect durable-store refresh latency in addition to the polling interval.
- Stop on terminal state, deletion, disconnect, authorization loss, or an explicit maximum lifetime.
- Add jitter/backoff when many simultaneous consumers could poll together.
- Estimate query load as active connections divided by polling interval before choosing this strategy.

Polling from the WebSocket consumer is a deliberate fallback, not an existing universal TDI pattern. Document why it is acceptable for the endpoint.

### 3. Cross-process Channels delivery

Use when a Celery worker or another process should push frequent/low-latency events and a shared Channels layer is configured.

```text
worker -> shared channel layer -> consumer group -> WebSocket
```

Before choosing it, inspect the actual checkout and deployment configuration for:

- `CHANNEL_LAYERS`
- a shared backend such as `channels-redis`
- `get_channel_layer()` and existing `group_send()` conventions

Do not infer that a cross-process channel layer exists merely because Django Channels and Redis caching are installed. An in-memory layer cannot bridge Celery and ASGI processes.

When available:

- Use a stable, sanitized group name derived from the resource ID.
- Join after authorization and discard on disconnect.
- Publish after durable state is successfully saved.
- Treat the channel event as a notification/snapshot, not the only source of truth.
- Load the durable initial snapshot on connect so reconnects do not miss state.
- Make duplicate events harmless.
- Send the terminal event before closing.

### 4. Broker/pub-sub outside Channels

Use only when the owning subsystem already has an established broker protocol and adding a Channels layer would duplicate it. Keep broker subscription details behind the consumer or a domain service, and retain the same public WebSocket contract.

## Message contract

Use JSON with an explicit event type and resource payload. Keep the contract small, typed, and forward-compatible.

Snapshot example:

```json
{
  "type": "coding_task.updated",
  "task": {
    "id": "task-id",
    "stage": "provisioning",
    "progress": 0.1,
    "updated_ts": 1740000000
  }
}
```

Terminal example:

```json
{
  "type": "coding_task.terminal",
  "task": {
    "id": "task-id",
    "stage": "completed",
    "progress": 1.0
  }
}
```

Guidelines:

- Use domain-qualified event names such as `<resource>.updated`, `<resource>.output`, `<resource>.terminal`, and `<resource>.error`.
- Include a sequence/revision when ordering or replay matters.
- Distinguish snapshot events from append-only output events.
- Prefer explicit typed fields over an unstructured catch-all payload.
- Never expose secrets, local credentials, environment variables, or unrestricted process output.
- Keep terminal semantics documented and deterministic.

## Snapshot versus output streams

A snapshot stream communicates the latest resource state. It may coalesce intermediate changes safely.

An output stream communicates ordered events or lines. It normally needs:

- sequence numbers,
- bounded buffering,
- reconnect/replay behavior,
- output sanitization,
- explicit stdout/stderr or event categories.

Do not claim to provide live command output when the producer uses buffered `subprocess.run()`. That requires incremental process reading, such as `Popen`, plus a callback/pub-sub path.

## Async and lifecycle safety

- Never call synchronous model, filesystem, subprocess, or network APIs directly on the event loop.
- Store references to background tasks created by the consumer.
- Cancel those tasks in `websocket_disconnect`.
- Handle `asyncio.CancelledError` as normal shutdown.
- Avoid mutable connection state shared through class attributes; initialize per-connection state before use.
- Avoid unbounded queues and buffers.
- Log backend failures with resource identifiers, then close with a meaningful code.
- Do not repeatedly send unchanged snapshots.

## Durable state and reconnection

Even for push-based endpoints, durable resource state remains authoritative:

1. Load current state when the socket connects.
2. Subscribe or begin polling.
3. Make updates idempotent.
4. Let the client recover by reconnecting and receiving a fresh snapshot.

If there is a subscribe-after-read race, either subscribe before loading and deduplicate events or load once more after subscribing.

## Validation

From the backend module root:

```bash
../pip-ade-testing/lint.sh
```

Use the repository test wrapper only when tests are requested:

```bash
../pip-ade-testing/test.sh <test-target>
```

From the TDI root, verify Django configuration:

```bash
bin/platform check
```

Also verify under `DJANGO_MODE=asgi` that `core.sock_urls` discovers the app route. A normal Django check may not exercise dynamic socket discovery.

## Checklist

- [ ] Consumer and `sock_urls.py` live in the owning `pip-ade-*` submodule.
- [ ] Route follows `/sock/<app>/...` conventions.
- [ ] Authentication and object authorization occur before accept.
- [ ] Delivery strategy is justified for the producer and expected traffic.
- [ ] Synchronous work is moved off the event loop.
- [ ] Initial, changed, error, and terminal behavior are defined.
- [ ] Background work is cancelled on disconnect.
- [ ] Messages use an explicit stable JSON contract.
- [ ] Durable state supports reconnect recovery.
- [ ] Backend lint and ASGI route discovery pass.
- [ ] No unrelated submodule pointer changes are included.
