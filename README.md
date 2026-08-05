# Server & Network Tech Stack

A minimal client/server stack built from scratch in plain Java — no frameworks, no libraries. It implements its own wire protocol over raw TCP sockets, a tiny Express-style router, and a small HTTP-like client library (`HTTP.get/post/put/delete`), wired up to a toy backend and frontend to prove it end to end.

The point of the project is to learn the plumbing: sockets, threads, request/response framing, routing, and a client that talks to it — by writing every layer by hand.

## How it's laid out

```
protocol/    Reusable networking layer (the "stack" itself)
backend/     Example server app built on top of protocol/
frontend/    Example client app built on top of protocol/
```

### `protocol/` — the stack

| File | Responsibility |
|---|---|
| `Server.java` | Opens a `ServerSocket`, accepts connections on its own thread, spins up a `Channel` per client |
| `Channel.java` | Handles one client connection: reads a `Request`, matches it against the routing table, writes back a `Response` |
| `Connector.java` | Client-side socket wrapper: opens a `Socket`, sends a `Request`, reads a `Response` |
| `Network.java` | Thin façade that opens a `Connector`, does the request/response round trip, and closes it |
| `HTTP.java` | Public client API — `HTTP.get(url)`, `HTTP.post(url, data)`, `HTTP.put(url, data)`, `HTTP.delete(url)` |
| `URLComponent.java` | Parses `"http://host:port/path"` strings and matches request paths against route patterns (incl. `:param` segments) |
| `Request.java` / `Response.java` | Serializable message objects sent over the socket (via Java object serialization) |
| `RouterHandler.java` | Base class apps extend to register `"METHOD=path"` → handler mappings |

Requests and responses are plain `Serializable` Java objects sent with `ObjectOutputStream`/`ObjectInputStream` — there's no HTTP text framing on the wire, just Java serialization over TCP. Route matching supports path parameters (e.g. `users/:name`), which land in `Request.getParams(...)`.

### `backend/` — example server

- `ENV.java` — server config (`DOMAIN`, `PORT`)
- `Application.java` — entry point: builds a `Server`, attaches routes, starts listening
- `Route.java` — registers the app's routes against `RouterHandler`
- `Controller.java` — route handlers (`getUsers`, `getUserById`, `postUser`, `putUser`, `deleteUser`)
- `database/User.java` — an in-memory, hardcoded stand-in for a real data layer

### `frontend/` — example client

- `ENV.java` — client config (`BASE_URL`)
- `Client.java` — calls the backend over `HTTP.get/post/put/delete` and prints the responses

## Running it

No build tool is required — just the JDK.

**1. Compile everything:**
```bash
javac -d out $(find . -name "*.java")
```

**2. Start the server** (in one terminal):
```bash
java -cp out backend.src.application.Application
```

**3. Run the client** (in another terminal):
```bash
java -cp out Client
```

The client fetches `GET /users` from the server and prints the decoded response. `Client.java` has additional (commented-out) examples for `GET /users/:name`, `POST /user`, `PUT /users/:name`, and `DELETE /users/:name` — uncomment to try them.

## Status

This is a learning/reference project, not production infrastructure: the "database" is a hardcoded in-memory list reset on every call, there's no auth/TLS, and error handling is intentionally simple. Treat it as a readable example of how an HTTP-ish client/server stack fits together under the hood.

## Branches

`master` holds the version documented above. Each `version/vN` branch on `origin` is a checkpoint in an incremental series — every branch builds on the one before it, so `v6` contains everything from `v1`–`v5` plus its own additions. They live on the remote only (`origin/version/v1` … `origin/version/v6`); check one out with:

```bash
git checkout -b version/v4 origin/version/v4
```

| Branch | Builds on | Adds |
|---|---|---|
| `master` / `version/v1` | — | The base stack described in this README: `protocol/` (Server, Channel, Connector, HTTP, Router), plus the example `backend/`/`frontend/` apps. |
| `version/v2` | v1 | Splits the connector into `ClientConnector`/`ServerConnector`; introduces a request-dispatching pipeline (`Dispatcher`, `DispatcherHandler`, `ControllerHandler`) and a `MainTaskQueue` so controller work runs off a synchronized queue instead of inline in `Channel`. |
| `version/v3` | v2 | Adds a rate limiter, a `RouteBuilder` (moved into `backend`), and a `test/` package (`Main`, `Process`, `Resource`) exercising a singleton under multithreading. |
| `version/v4` | v3 | Big reorg: the stack moves under `WebServer/` (protocol + backend + frontend). Adds a `LoadBalancer/` module (`Balancer`, `EventLoop`, `MacroTaskQueue`, `Service`, `WorkerClient`) doing round-robin balancing with DDoS-style load testing from the client; a `Container.java` abstraction in the protocol layer; `Containerization/` (renamed from `test/`); and a "stop-the-world" buffer context switch plus a semaphore-backed service registry for scaling. |
| `version/v5` | v4 | Adds a `StreamServer/` module (`Server`, `Client`, `ThreadPool`) that streams multiple serialized objects over one connection (HTTP/2-ish). Refactors controller execution into `ControllerExecutor`/`ControllerExecutable`/`ServerErrorExecutable` running on a thread pool, replacing the older `ControllerHandler`. |
| `version/v6` | v5 | Renames the v4 load balancer to `InconsistentBalancer/` (round robin) and adds a `ConsistentBalancer/` implementing consistent hashing, including handling for adjacent and edge-server failures and a mechanism to simulate server failure for testing. |

Reading order for the series is v1 → v6; each step is a good diff to read on its own (`git diff origin/version/v2 origin/version/v3`, etc.) to see one concept added at a time — task queues, rate limiting, load balancing, streaming, then consistent hashing.
