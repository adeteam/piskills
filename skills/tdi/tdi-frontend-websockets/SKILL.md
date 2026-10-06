---
name: tdi-frontend-websockets
description: Implement TDI Angular WebSocket clients with same-origin URL construction, typed event handling, initial REST snapshots, terminal shutdown, reconnect policy, and component/service lifecycle cleanup. Use when adding or changing browser-side WebSocket behavior in an npm-ade-* package.
---

# TDI Frontend WebSockets

## Goal

Build frontend WebSocket clients that are consistent in connection setup, security, event contracts, reconnection, and cleanup while allowing each feature to choose component-local or shared service ownership based on its needs.

Also load `/skill:tdi-frontend-feature-delivery` for normal Angular package conventions. Load `/skill:tdi-frontend-api-model-factory` when the streamed resource also has a REST/model-factory representation. Load `/skill:tdi-backend-websockets` when implementing the server side too.

## Existing architecture

TDI's established browser WebSocket examples include:

- `ui/modules/npm-ade-remote/src/components/xterm.component.ts`
- `ui/modules/npm-ade-remote/src/components/xterm/connection/base.ts`
- `ui/modules/npm-ade-chat/src/services/transport.service.ts` for an external LivePerson socket

TDI backend sockets are conventionally exposed under:

```text
/sock/<django-app>/<feature>/<resource identifiers>/
```

The development proxy handles `/sock/` with WebSocket upgrades. Build same-origin URLs so the same code works behind the deployed reverse proxy.

## Standard connection flow

For a durable backend resource:

1. Load the initial resource through the existing model factory/REST endpoint.
2. Decide from that snapshot whether live updates are needed.
3. Open a WebSocket only for active/non-terminal resources.
4. Apply typed incoming events to the displayed resource.
5. Close immediately after a terminal event.
6. Close and clear timers in `ngOnDestroy`.
7. Retry only failures that are plausibly transient.
8. Recover from reconnect using a fresh server snapshot rather than assuming no events were missed.

This hybrid REST-plus-WebSocket approach gives reliable page loading and efficient live updates:

```text
initial page load -> REST snapshot
active lifecycle  -> WebSocket updates
reconnect         -> WebSocket initial snapshot or REST reload
terminal state    -> final update, then close
```

A pure socket-first flow is acceptable for inherently ephemeral streams such as terminals, but should not replace REST/model factories for durable domain entities without a clear reason.

## URL construction

Use the current page scheme and host:

```typescript
const protocol = window.location.protocol === 'https:' ? 'wss' : 'ws';
const path = `/sock/example/task/${encodeURIComponent(taskId)}/`;
const socket = new WebSocket(
    `${protocol}://${window.location.host}${path}`
);
```

Rules:

- Use `wss` when the page uses HTTPS.
- Use `window.location.host` so reverse-proxy routing and ports are preserved.
- Keep the `/sock/<app>/...` path aligned exactly with backend `sock_urls.py`.
- Encode dynamic path components.
- Prefer same-origin sockets so session cookies flow through `AuthMiddlewareStack`.

Browser WebSocket constructors cannot set arbitrary `Authorization` headers. If an environment relies on token-only authentication rather than a session cookie, define and review an explicit secure authentication mechanism instead of placing long-lived secrets in query parameters.

## Decide where the connection lives

### Component-owned socket

Use when one routed component is the sole consumer and the connection lifetime exactly matches that component.

Advantages:

- simple ownership,
- straightforward `ngOnDestroy`,
- no accidental sharing between resources.

### Feature service

Use when multiple panes/components consume the same stream, reconnection logic is substantial, or the feature needs a shared observable state.

Place it under the owning package's `src/services/` and follow that package's provider/export conventions. If a service is a root singleton, do not keep one unkeyed socket field that lets one resource replace another. Either enforce one active resource explicitly or key connections by resource ID.

### Reusable connection abstraction

Use for protocol families such as terminals where multiple features share framing and callbacks. Follow the `npm-ade-remote` connection classes rather than duplicating low-level wrappers.

## Event contract

Define TypeScript interfaces for the server envelope and event payload when the stream is more than trivial.

```typescript
interface CodingTaskEvent {
    type: 'coding_task.updated' | 'coding_task.terminal';
    task: JiraCodingTaskRecord;
}
```

Handle event types explicitly:

```typescript
socket.onmessage = (event: MessageEvent): void => {
    let packet: CodingTaskEvent;
    try {
        packet = JSON.parse(event.data) as CodingTaskEvent;
    } catch (error) {
        return;
    }

    if (packet.type === 'coding_task.updated') {
        // apply snapshot
    } else if (packet.type === 'coding_task.terminal') {
        // apply final snapshot and close
    }
};
```

Guidelines:

- Ignore unknown event types safely for forward compatibility.
- Do not assume every message contains the same payload.
- Distinguish snapshots from append-only output events.
- Use sequence/revision fields to reject stale or duplicate ordered events when needed.
- Avoid logging payloads that can contain sensitive output.

## Updating model-backed state

When REST loaded a TDI model record, preserve model behavior when appropriate:

```typescript
Object.assign(this.item, packet.task);
```

This keeps the existing record instance and its methods while updating fields. However, choose replacement instead when an `OnPush` component or `ngOnChanges` contract requires a new object reference:

```typescript
this.item = Object.assign(
    Object.create(Object.getPrototypeOf(this.item)),
    this.item,
    packet.task
);
```

Follow neighboring component behavior rather than applying one update style universally. Ensure nested data replacement is sufficient for the panes that consume it.

Do not create a second ad-hoc HTTP representation when a registered model factory already defines the resource schema.

## Terminal behavior

Each feature must define terminal state explicitly. Examples include:

- `completed`
- `errored`
- `cancelled`
- session ended by the remote system

For durable task streams:

- Do not connect when the initial REST snapshot is already terminal.
- Apply the terminal packet before closing.
- Stop reconnecting after terminal state.
- Treat a normal close (`1000`) as intentional.

The backend should normally send the final snapshot and initiate a normal close; the client may also call `close()` after processing the terminal event.

## Reconnection policy

Reconnect only when useful for the feature.

A reasonable durable-task policy is:

- no retry after normal close `1000`,
- no automatic retry for application authorization/not-found codes such as `4401`, `4403`, and `4404`,
- retry abnormal/network closure only while the resource is believed active,
- use bounded exponential backoff or a documented fixed interval,
- allow only one reconnect timer,
- clear the timer on destroy,
- cap retries or surface a disconnected state after a threshold.

A terminal/interactive socket may use a different policy because dropped input and resumed sessions have different semantics. Keep the policy local to the endpoint rather than creating a misleading universal reconnect rule.

## Lifecycle cleanup

Components that own sockets must implement `OnDestroy`:

```typescript
public ngOnDestroy(): void {
    this.destroyed = true;

    if (this.reconnectTimeout) {
        clearTimeout(this.reconnectTimeout);
        this.reconnectTimeout = null;
    }

    if (this.socket) {
        this.socket.close();
        this.socket = null;
    }
}
```

Also clean up:

- RxJS subscriptions,
- browser event listeners,
- heartbeat timers,
- pending reconnect timers,
- service registrations or resource-keyed listeners.

Guard callbacks from stale sockets so an older connection cannot overwrite the current connection's state.

## Errors and user experience

Separate initial-load failure from live-update failure:

- A failed initial REST request usually blocks the page and should show an error.
- A temporary socket failure should generally retain the last good snapshot and show a reconnecting/stale indicator.
- Authorization or not-found closure should stop reconnecting and show a durable error.
- Malformed or unknown packets should not crash the page.

Do not replace useful page content with a generic error merely because a transient stream disconnected.

## Snapshot streams versus output streams

### Snapshot stream

Use for task stage, progress, status, and aggregate results. Replacing/coalescing intermediate snapshots is acceptable.

### Output/event stream

Use for terminal output, log lines, or ordered workflow events. It requires:

- append rather than replace,
- sequence tracking,
- duplicate suppression,
- a bounded in-memory display buffer,
- optional resume cursor/replay behavior,
- output sanitization before rendering.

Never render streamed backend content as trusted HTML. Use text binding unless content has passed the project's explicit sanitization path.

## Endpoint-specific flexibility

Continuity means every endpoint follows common conventions for:

- `/sock/<app>/...` URLs,
- same-origin `ws`/`wss` selection,
- authentication expectations,
- typed event envelopes,
- explicit terminal semantics,
- cleanup,
- reconnect decisions,
- durable snapshot recovery where applicable.

Endpoints may still differ in:

- component versus service ownership,
- whether the client sends messages,
- heartbeat requirements,
- fixed versus exponential reconnect delay,
- snapshot replacement versus ordered append,
- whether a REST bootstrap exists,
- whether completion closes the stream.

Document those differences near the owning service/component rather than forcing terminal, chat, and background-task sockets into one behavior.

## Validation

From the owning Angular module:

```bash
yarn lint
yarn build
```

Before handoff, validate the app workspace when the change crosses package boundaries:

```bash
cd ui
yarn lint
yarn build
```

For manual verification, check:

1. HTTP pages use `ws`, HTTPS pages use `wss`.
2. The browser sends the expected session cookie.
3. The initial snapshot appears immediately.
4. Changed events update all relevant panes.
5. Navigation away closes the connection.
6. Terminal state applies before closure.
7. Abnormal closure retries according to policy.
8. `4401`/`4403`/`4404` do not loop indefinitely.

## Checklist

- [ ] Code lives in the owning `npm-ade-*` package.
- [ ] REST/model factory remains the initial source for durable entities.
- [ ] Socket opens only when live updates are needed.
- [ ] URL uses same-origin `ws`/`wss` and encoded identifiers.
- [ ] Incoming event types and payloads are explicit.
- [ ] State-update strategy preserves required model/change-detection behavior.
- [ ] Terminal state stops streaming and reconnection.
- [ ] Unexpected disconnect policy is bounded and endpoint-appropriate.
- [ ] Socket and timers are cleaned up on destroy.
- [ ] Sensitive streamed content is not logged or rendered unsafely.
- [ ] Module lint and build pass.
