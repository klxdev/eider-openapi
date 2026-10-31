# Building a REST service with OpenAPI

This guide builds a small REST service with an OpenAPI 3.1 description step by step, with
eider-openapi (operations, validation, `/openapi.json`) on
[eider-http](https://github.com/klxdev/eider-http) (HTTP/1.1 server and client, router,
middleware), on std's sockets, deadlines and thread pool. It was the second half of
eider-lang's "Services" guide; the first half, a JSON-RPC service, is
[eider-rpc](https://github.com/klxdev/eider-rpc)'s docs/guide.md.

[`examples/todo`](../examples/todo/todo.ei) goes further than this page: a CRUD REST service
described by OpenAPI, with bearer tokens, CORS, request ids, an access log, Prometheus
metrics, a per-route timeout, and an end-to-end test that checks the served description
against a golden file.

> **Note:** The code blocks on this page are marked `eider,ignore`: they import packages
> (`http`, `openapi`, `json`), which a code block alone has no `eider.toml` or `eider.lock`
> to find. Each one was built and run in a package set up as below, and the outputs shown
> are what it printed. examples/todo is tested by `scripts/test.sh`.

## What a service is made of

| Piece | Module | |
|---|---|---|
| `http.Request`, `http.Response`, `http.Handler` | `http` | a handler is a plain function value, `fn(http.Request) -> http.Response` |
| the server | `http/server` | `server.listen(addr, cfg)` then `srv.serve(handler, token)`: a pool of worker threads, deadlines on every read and write, limits on heads and bodies, graceful shutdown when the token is cancelled |
| the router | `http/router` | `r.get("/items/:id", h)`, `r.post(...)`; 404 and 405 with `Allow` |
| JSON-RPC 2.0 | `rpc/jsonrpc` (eider-rpc) | a method table, typed parameters, batches and notifications, the HTTP handler and a client |
| OpenAPI | `openapi`, `openapi/schema` (eider-openapi) | one operation value drives routing, validation and the served description |
| middleware | `http/middleware`, `http/metrics`, `http/problem` | request ids, access logs, timeouts, CORS, bearer and Basic auth, metrics, problem+json errors |

The server speaks plain HTTP. Run it on loopback or a private interface behind a
TLS-terminating proxy; TLS 1.3 written in Eider is planned (eider-lang's
`docs/design/roadmap.md`, P1.4).

## A REST service with OpenAPI, step by step

eider-openapi registers a route **together with its description**: one operation value
gives the router its pattern, validates parameters and bodies before your handler runs, and
produces `GET /openapi.json`. So the documentation can't drift from the code.

### 1. The package

```sh
$EIDER/scripts/eider-pkg -C todo init
cd todo
$EIDER/scripts/eider-pkg add http --git https://github.com/klxdev/eider-http.git --version 0.1.0 --uses net,io,clock,threads
$EIDER/scripts/eider-pkg add openapi --git https://github.com/klxdev/eider-openapi.git --version 0.1.0
```

openapi itself is pure, so it is granted nothing. To work on checkouts of these repositories
instead, name them in `EIDER_PKG_OVERRIDE` when you fetch:
`EIDER_PKG_OVERRIDE=http=../eider-http,openapi=../eider-openapi,json=../eider-json eider-pkg fetch`
(eider-lang's docs/site/tooling/eider-pkg.md).

### 2. State, schemas and encoders

```eider,ignore
// items.ei
import "std/sync"
import "http"
import "http/problem"
import "openapi"
import "openapi/schema"

pub type Item = { id: i64, name: List[u8] }

type Db = { next: i64, items: Map[i64, Item] }

pub type Store = { st: sync.Lock[Db] }

pub fn store() -> Store { Store(sync.lock(Db(1, Map.new()))) }

impl Store {
    fn create(self, name: List[u8]) -> Item {
        self.st.with(|d: Db| -> (Db, Item) {
            let mut t = d.copy()
            let item = Item(t.next, name)
            t.items.insert(t.next, item)
            t.next += 1
            (t, item)
        })
    }

    fn get(self, id: i64) -> Option[Item] {
        self.st.with(|d: Db| -> (Db, Option[Item]) (d, d.items.get(id)))
    }
}

// schemas, decoders and encoders: written by hand until JSON derive
fn item_schema() -> schema.Schema {
    let mut s = schema.object().closed()
    s = s.field("id", schema.integer().format("int64").min(1))
    s.field("name", schema.string().min_length(1).max_length(100))
}

fn new_item_schema() -> schema.Schema {
    schema.object().closed().field("name", schema.string().min_length(1).max_length(100))
}

fn item_json(t: Item) -> List[u8] { openapi.obj().int("id", t.id).str("name", t.name).done() }

fn decode_id(i: openapi.Input) -> Result[i64, problem.ApiError] { i.path_int("id") }

fn decode_name(i: openapi.Input) -> Result[List[u8], problem.ApiError] { i.str("name") }
```

Schemas are the subset of JSON Schema 2020-12 that service types need: objects
(`field` required, `prop` optional, `closed()` refuses unknown members), arrays, strings
with lengths and enums, integers with bounds, booleans, `nullable()`, and named
components that become `$ref`s.

### 3. Operations

```eider,ignore
/// The service: GET /items/{id}, POST /items, and GET /openapi.json.
pub fn app(s: Store) -> http.Handler {
    let mut svc = openapi.service("Items API", "1.0.0")
    let item = svc.component("Item", item_schema())

    let mut get = openapi.op("GET", "/items/{id}").operation_id("getItem").summary("Get an item")
    get = get.path_param("id", schema.integer().min(1))
    get = get.returns(200, "The item", item).fails(404, "No such item")
    svc.add(get, openapi.typed(decode_id, |id: i64| -> Result[Item, problem.ApiError] {
        s.get(id).ok_or(problem.not_found("no such item"))
    }, item_json))

    let mut create = openapi.op("POST", "/items").operation_id("createItem").summary("Create an item")
    create = create.body(new_item_schema(), "The new item").returns(201, "The item", item)
    svc.add(create, openapi.typed(decode_name, |name: List[u8]| -> Result[Item, problem.ApiError] {
        Ok(s.create(name))
    }, item_json))

    svc.handler()
}
```

`openapi.typed(decode, f, encode)` is the handler: `decode` reads the validated request,
`f` is the logic (no HTTP in it), `encode` writes the result. The answer is the
operation's first 2xx status, or the error's `application/problem+json` (RFC 9457).
`svc.add` panics at start-up on a programming error, such as a `{name}` in the path
without a `path_param`, or two operations with the same id.

> **Note:** Builders are written as reassignments (`get = get.path_param(...)`) because the
> committed seed can't continue a method chain on the next line. Lines starting with `.`
> continue the expression in the compiler sources, and will work after the next seed
> regeneration.

### 4. Run it

`main.ei` is the same as the JSON-RPC service's, with `items.app(items.store())`. Then:

```text
$ curl -s localhost:8080/items -H 'Content-Type: application/json' -d '{"name": "lamp"}'
{"id":1,"name":"lamp"}
$ curl -s localhost:8080/items/1
{"id":1,"name":"lamp"}
$ curl -s localhost:8080/items/2
{"type":"about:blank","title":"Not Found","status":404,"detail":"no such item"}
$ curl -s localhost:8080/items -H 'Content-Type: application/json' -d '{"name": "", "color": "red"}'
{"type":"about:blank","title":"Bad Request","status":400,"detail":"the request does not match the API description","errors":[{"in":"body","pointer":"#/name","detail":"must not be empty"},{"in":"body","pointer":"#/color","detail":"is not allowed here"}]}
$ curl -s localhost:8080/items -d 'x'
{"type":"about:blank","title":"Unsupported Media Type","status":415,"detail":"the body must be application/json"}
$ curl -s localhost:8080/items/abc
{"type":"about:blank","title":"Bad Request","status":400,"detail":"the request does not match the API description","errors":[{"in":"path","name":"id","detail":"must be an integer"}]}
$ curl -s localhost:8080/openapi.json
{"openapi":"3.1.0","info":{"title":"Items API","version":"1.0.0"},"paths":{"/items/{id}":{"get":{"summary":"Get an item", ...
```

The served description includes the 400 and 415 responses the validation can give, and a
`Problem` component for them.

## Going further: what examples/todo adds

[`examples/todo/todo.ei`](../examples/todo/todo.ei) and
[`main.ei`](../examples/todo/main.ei) build on the same pieces:

- **Authentication.** `svc.bearer_auth("bearerAuth", check)` and `.secured("bearerAuth")`
  on the write operations: a missing or unknown token is 401 with `WWW-Authenticate`.
  `middleware.token_table` compares tokens in constant time.
- **Limits and timeouts per route.** `.max_body(4096)` (413 above it) and
  `svc.add_with(op, layer, h)` with `middleware.timeout(time.seconds(2), pool, h)`: 503 if
  the answer isn't ready by the deadline.
- **Middleware around the whole service**, outermost first: `request_id` (keeps a
  client's `X-Request-Id` or makes one), `access_log` (one JSON line per request, never
  headers or bodies), `metrics.wrap` (Prometheus counters and a latency histogram served at
  `/metrics`), `problems` (the router's own 404 and 405 as problem+json) and `cors`.
- **Query parameters and partial updates**: `list.query("done", schema.boolean(), ...)`,
  a PATCH whose body distinguishes a missing field from `null`.
- **An end-to-end test** that checks CRUD, every error status, CORS, concurrency, metrics,
  the access log, and the served `/openapi.json` against `tests/openapi.golden.json`.

## Limits to know

- One OS thread serves one connection at a time: `cfg.workers` bounds concurrency, and
  connections beyond the queue get 503. Stackless tasks, which lift this without changing
  `serve`'s signature, are planned.
- A trap in a handler (an overflow, an index out of bounds, `unwrap` on `None`) ends the
  process. Validate input before arithmetic; eider-rpc's examples/kvstore checks for overflow
  before adding. Containing traps is planned.
- A handler that blocks forever holds a worker forever; `middleware.timeout` bounds the
  answer, not the thread. Pass `req.deadline` down to anything that waits.
- Bodies are read whole (up to `Limits.max_body`, 1 MiB by default); streaming bodies are
  planned. Floats aren't in the language yet, so there are no `number` schemas.
- No TLS in the server: put it behind a proxy.

eider-http's `docs/services.md`, eider-rpc's `docs/jsonrpc.md` and eider-openapi's
`docs/openapi.md` describe the architecture, the limits and defaults, and the security
posture in detail.
