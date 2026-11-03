# eider-openapi

OpenAPI 3.1 services for [Eider](https://github.com/klxdev/eider-lang), on
[eider-http](https://github.com/klxdev/eider-http) and
[eider-json](https://github.com/klxdev/eider-json). A route is registered together with its
operation, and that one value drives the router pattern, the validation of parameters and
JSON bodies against their schemas (a 400 problem+json listing every error, 413, 415, 401
for `secured` operations), typed handlers (`decode / fn(T) -> Result[U, ApiError] /
encode`) and `GET /openapi.json`, so the description can't drift from the code.

| Module | What |
|---|---|
| `openapi` | `service`, `op` (operations, parameters, bodies, responses, security), `typed`, `Input`, the JSON builders, `/openapi.json` |
| `openapi/schema` | the JSON Schema 2020-12 subset services need: build, render, validate |
| `openapi/derive` | `Schematic`: schemas derived from types and `@[json(...)]` (`Todo.json_schema()`); with `json_body` handlers and `check_response` no schema, decoder or encoder is written by hand |
| `openapi/gen` | Eider types, an `Api` trait with `register`, and a typed `Client` from an OpenAPI 3 document (`target/bin/eider-openapi gen spec.json`) |

- [docs/openapi.md](docs/openapi.md): the design, the schema subset, typed handlers, errors
  and the known gaps.
- [docs/guide.md](docs/guide.md): a REST service with OpenAPI, step by step.
- [examples/todo_gen](examples/todo_gen): the todo document run through `eider-openapi gen`,
  served and called with the generated client.
- [examples/todo](examples/todo): a CRUD service with bearer tokens, CORS, request ids, an
  access log, Prometheus metrics, a per-route timeout, and an end-to-end test that checks
  the served description against `tests/openapi.golden.json`.

## Using it

```toml
[deps.openapi]
git = "https://github.com/klxdev/eider-openapi.git"
version = "0.1.0"
# pure: granted nothing

[deps.http]
git = "https://github.com/klxdev/eider-http.git"
version = "0.1.0"
uses = ["net", "io", "clock", "threads"]
```

then `eider fetch` and `import openapi`.

## Developing

```sh
sh scripts/build.sh     # fetch the dependencies, check every module
sh scripts/test.sh      # build, then tests/*.ei and examples/*/tests/*.ei against their .out files
EIDER_TOOLCHAIN=../eider-lang sh scripts/test.sh   # the same, against an eider-lang checkout
```

The scripts use the toolchain [`eider`](https://github.com/klxdev/eider-lang/blob/main/docs/site/tooling/eider.md)
picks: `$EIDER_TOOLCHAIN` (an installed toolchain's directory, or an eider-lang checkout for
unreleased work), else the installed release for `eider.toml`'s `eider = "0.1"` (what
`eider toolchain show` prints here). Install `eider` and the latest toolchain with
eider-lang's `install.sh --toolchain` (from its latest release); without either, the
scripts stop with exit 2 and say so. The toolchain's `VERSION` must have this package's
MAJOR.MINOR (the `eider` field of `eider.toml`), else they stop with exit 2 and say which
release to use. eiderc and eider-pkg run through `eider` (`eider eiderc ...`, `eider eider-pkg
...`, with that toolchain), or directly when `eider` is not on the PATH.

The dependencies are git packages. To build offline or against checkouts with uncommitted
changes, eider-pkg's `EIDER_PKG_OVERRIDE` (`name=dir,...`) takes them from local
directories; `scripts/env.sh` sets it for you for http and json: `$EIDER_HTTP` /
`$EIDER_JSON`, else the sibling checkouts `../eider-http` and `../eider-json`, else (json) the
pinned release `.vendor/json` of an eider-lang checkout used as the toolchain (fetched there
and hash-checked on first use by its `scripts/vendor.sh`), else their repositories (network). eider.lock keeps the git sources either
way (a library doesn't commit its lock). `UPDATE_GOLDEN=1 sh scripts/test.sh` rewrites
examples/todo's `openapi.golden.json` (review the diff).

eider-lang's `scripts/verify.sh` builds and tests this repository against its own checkout
when it is next to it (`scripts/check-tools.sh`).

## Versions

`VERSION` and `eider.toml` hold this package's version (semver). Before 1.0 its MAJOR.MINOR
follows the Eider toolchain it goes with (`eider = "0.1"`) and its PATCH is its own;
CHANGELOG.md lists the releases. A release is tagged `v<version>`.
