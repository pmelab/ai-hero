# Real-time in-app transport for Notifications

Research for [#6](https://github.com/pmelab/ai-hero/issues/6) (map: #1). Date: 2026-11-12.

**Question:** How should a live in-app Notification reach an open Cadence tab: SSE vs WebSocket vs polling, on this React Router 8 node server (`@react-router/serve`) with SQLite (better-sqlite3 + Drizzle)? What does the hosting allow for long-lived connections?

## Answer (short)

- **SSE through a resource route** works on the current `react-router-serve` server with **no custom server**. The Express adapter flushes `text/event-stream` headers and flushes every chunk, so the built-in `compression()` middleware does not buffer events. Abort on disconnect arrives as `request.signal`.
- **WebSocket needs a custom server.** `react-router-serve` exposes no hook for the HTTP `upgrade` event, and Node has no built-in WebSocket server. You must move to an own `@react-router/express` server plus a library such as `ws`.
- **Polling** works everywhere with zero new infrastructure, but adds latency and load per open tab.
- **Fan-out:** today there is one process (ADR 0006), so an in-memory event emitter is enough. SQLite has no LISTEN/NOTIFY; across processes on one machine, `PRAGMA data_version` polling detects commits by other connections. Across machines, SQLite is not an option at all (ADR 0006).
- **Hosting:** **no deploy/hosting config exists in the repo.** Nothing constrains long-lived connections yet, and nothing proves they will survive a proxy either.
- **Lean:** SSE (server -> browser only is what a Notification needs; mark-as-read etc. stay normal actions). Polling as fallback. WebSocket only if a later feature needs browser -> server streaming.

## 1. Repo facts

| Fact | Source |
| --- | --- |
| Prod start: `react-router-serve ./build/server/index.js`; dev: `react-router dev` (Vite) | `package.json` scripts |
| `@react-router/serve`, `@react-router/node`, `react-router` all `8.3.0` | `package.json` |
| SSR on, no `app/entry.server.tsx`, no custom `server.ts` | `react-router.config.ts`, `app/` listing |
| One SQLite file `data.db`, WAL mode, one lazily opened connection per process | `app/db/index.ts` |
| "Cadence runs as one server process ... the app cannot run on more than one machine against the same data" | `docs/adr/0006-one-sqlite-file-through-drizzle.md` |
| Only Dockerfile is `.sandcastle/Dockerfile`: agent sandbox (`ENTRYPOINT ["sleep", "infinity"]`), not an app image | `.sandcastle/Dockerfile` |
| No `fly.toml`, `render.yaml`, `vercel.json`, `Procfile`, `docker-compose*`, `.github/` workflows, or deploy section in README/docs | repo search (`find`, `grep -i deploy\|fly\|render\|railway\|vercel`) |

**Hosting conclusion: none found.** The README only documents local dev (`npm run dev`, `http://localhost:5173`). Any proxy/PaaS idle-timeout and HTTP/2 facts must be re-checked once a host is picked.

## 2. How each transport fits this server

### SSE via a resource route (works today)

- A route with a `loader` and no default component is a resource route; it may return any `Response` ([React Router: Resource Routes](https://reactrouter.com/how-to/resource-routes)). So a `loader` can return `new Response(readableStream, { headers: { "Content-Type": "text/event-stream", "Cache-Control": "no-cache" } })`.
- `react-router-serve` is Express with `compression`, `express.static`, `morgan` ([React Router: @react-router/serve](https://reactrouter.com/api/other-api/serve); source `node_modules/@react-router/serve/dist/cli.js`: `app.use(compression())`).
- `compression` alone breaks SSE unless `res.flush()` is called ([compression README, "Server-Sent Events"](https://github.com/expressjs/compression#server-sent-events)); `text/event-stream` matches its `^text\/` compressible rule (`compressible/index.js`). **But** the React Router adapter handles it:
  - `@react-router/express` `sendRemixResponse`: `if (Content-Type matches /text\/event-stream/i) res.flushHeaders();` (`node_modules/@react-router/express/dist/index.js`).
  - `@react-router/node` `writeReadableStreamToWritable`: after each `writable.write(value)`, `if (typeof flushable.flush === "function") flushable.flush();` (`node_modules/@react-router/node/dist/index.js`).
  - Belt and braces: also send `Cache-Control: no-cache, no-transform`; `compression` skips any response with `no-transform` (`compression/index.js`, `shouldTransform`).
- Disconnect: the adapter wires `res.on("close", () => controller.abort())` into the `Request` signal (`@react-router/express/dist/index.js`, `createRemixRequest`). Use `request.signal` to unsubscribe and clear keep-alive timers.
- Node timeouts do not kill an SSE response: defaults are `requestTimeout` 300000 ms (time to receive the request), `headersTimeout` 60000, `keepAliveTimeout` 5000 (idle between requests), `timeout` 0 = no socket timeout ([Node http docs](https://nodejs.org/api/http.html#serverrequesttimeout); verified by `node -e` on v25.9.0).
- Protocol notes ([WHATWG HTML §9.2](https://html.spec.whatwg.org/multipage/server-sent-events.html), [MDN: Using SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)):
  - `EventSource` reconnects by itself; `retry:` sets the delay in ms.
  - Set `id:` per event (e.g. Notification id); on reconnect the browser sends `Last-Event-ID`, so the loader can replay missed Notifications from SQLite.
  - Send a comment line (`:`) about every 15 s so idle proxies do not drop the stream (spec §9.2.7).
  - **Connection limit:** over HTTP/1.1 the browser allows ~6 connections per browser+domain across all tabs; with HTTP/2 the limit is negotiated (default 100) (MDN; spec §9.2.7 suggests a shared worker). `react-router-serve` uses `app.listen` = plain HTTP/1.1, so **many open Cadence tabs could starve each other** unless a TLS/HTTP/2 proxy fronts it, or one tab owns the stream (SharedWorker / BroadcastChannel).
- Session auth: `EventSource` is same-origin and sends cookies, so the loader can resolve the current user like other loaders.

### WebSocket (needs a custom server)

- `react-router-serve` has no options: "if you need to customize the underlying express server, we'd rather you manage the server completely" -> use `@react-router/express` ([React Router: @react-router/serve](https://reactrouter.com/api/other-api/serve)).
- Node: upgrade requests are ignored unless something listens to `server.on("upgrade")`; the request then bypasses the normal request flow ([Node http: 'upgrade' event](https://nodejs.org/api/http.html#event-upgrade)). React Router routes never see it.
- Node's global `WebSocket` is a browser-compatible **client** (stable since v22.4.0); no server API is documented ([Node globals: WebSocket](https://nodejs.org/api/globals.html#websocket)). A server needs a dependency such as `ws`.
- Cost: new `server.ts`, new `start` script, own static/compression setup, a dev-mode story (Vite dev server owns the HTTP server in `react-router dev`), cookie/session parsing outside React Router, own heartbeat/reconnect logic in the client.

### Polling

- Works with the current server and any host. Implement as a `useRevalidator`/`useFetcher` interval on the layout, or a tiny resource route returning the unread count.
- Cost: latency = interval; one request per tab per interval; each runs a SQLite query (cheap: synchronous, in-process, per ADR 0006).

## 3. Fan-out given SQLite

| Topology | What works |
| --- | --- |
| **One process** (today, ADR 0006) | In-memory `EventEmitter`: the code path that inserts a Notification emits `notification:<userId>`; each SSE loader subscribes and unsubscribes on `request.signal` abort. No DB change needed. |
| **Several processes, one machine, same `data.db`** | No push from SQLite: no LISTEN/NOTIFY; `sqlite3_update_hook` is per connection ([SQLite update_hook](https://www.sqlite.org/c3ref/update_hook.html)) and better-sqlite3 12.11.1 does not expose it. Use `PRAGMA data_version` per process: its value changes "if changes were committed to the database by any other connection", including other processes ([SQLite PRAGMA data_version](https://www.sqlite.org/pragma.html#pragma_data_version)). Poll it every ~1 s in each process, then query new Notifications and emit locally. |
| **Several machines** | Not possible with SQLite as decided (ADR 0006). Would need a networked DB or a broker (Redis pub/sub, Postgres LISTEN/NOTIFY) — out of current scope. |

Design that survives all three: Notifications are rows first (inbox is the source of truth, per #1), the transport only signals "something new after id X". Replay via `Last-Event-ID` / a `?since=` query makes missed signals harmless.

## 4. Comparison

| | SSE (resource route) | WebSocket | Polling |
| --- | --- | --- | --- |
| Works on current `react-router-serve` | Yes | No (custom server) | Yes |
| New dependency | None | `ws` (or similar) | None |
| Direction | Server -> browser | Both | Browser pulls |
| Latency | Instant | Instant | Interval |
| Reconnect / resume | Built into `EventSource` + `Last-Event-ID` | Hand-written | Implicit |
| Main risk | 6-connection limit on HTTP/1.1; proxy buffering/idle timeouts (unknown host) | Server rewrite, dev-mode wiring | Load per tab, delay |

## 5. Open points for the decision

1. Pick a host (none exists). Check its proxy: HTTP/2 to the browser, idle timeout, response buffering for `text/event-stream`.
2. Many-tabs policy: accept HTTP/2 requirement, or one stream per browser via SharedWorker/BroadcastChannel.
3. Whether any planned feature needs browser -> server streaming (only that would justify WebSocket).
