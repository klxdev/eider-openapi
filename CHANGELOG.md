# Changelog

eider-openapi's versions (semver). Before 1.0 the MAJOR.MINOR is the Eider toolchain's it
goes with (`eider` in eider.toml), the PATCH its own.

## 0.5.0

For Eider 0.5 (`eider = "0.5"`). Requires eider-http and eider-json 0.5.0. eider-openapi#1 is partly done and stays open.

- `Service.planner()`: an operation's `max_body` can be enforced by the server before the body is read (`cfg.plan = svc.planner()`, eider-http#2).

- A response from an operation carries its id in the context (`operation`), which `http/metrics.by_route` uses as the label.

- openapi.derive (eider-openapi#1): `Schematic`, a `@derive` trait, gives every record and enum
  a JSON Schema derived by reflection with `@json(...)` (`Todo.json_schema()`); schema.ei gains
  `oneOf` (`one_of`) and map schemas (`map_of`, `additionalProperties`).
- `json_body`, `json_body_with`, `json_out`, `body_as`: typed handlers over the derived JSON codecs.
- `check_response(svc, method, path, resp)`: a response against its operation's description.

- `eider-openapi gen` (eider-openapi#2): types, an `Api` trait with `register`, and a typed `Client`
  from an OpenAPI 3 JSON document (openapi/gen, cli/); examples/todo_gen is the round trip.
  `schema.prune`; `Input.query_text`, `header_text`, `path_text`; `text`.

## 0.4.0

For Eider 0.4 (`eider = "0.4"`). Requires eider-http and eider-json 0.4.0. Nothing else changed.

## 0.3.0

For Eider 0.3 (`eider = "0.3"`): every plain string literal interpolates (eider-lang U29), so
the literals with braces here are raw strings or have their braces doubled. Requires eider-http and eider-json 0.3.0.

The grant to eider-http, here and in examples/todo, is what its modules use: `clock, fs, io,
net, threads` (eider-lang #135).

## 0.2.0

For Eider 0.2 (`eider = "0.2"`): eider-lang's 0.2 builds programs only through LLVM (clang
17 or newer; U23) and bootstraps the compiler from a WebAssembly seed (seed/eiderc.wasm; U24).
scripts/env.sh recognises such a checkout (seed/eiderc.wasm, or seed/eiderc.c before).
Nothing else changed.

## 0.1.3

The scripts find the Eider toolchain through `eider` (eider-lang#133) instead of a fixed
eider-lang checkout: scripts/env.sh takes `$EIDER_TOOLCHAIN` (a toolchain directory or an
eider-lang checkout; eider-lang's scripts/check-tools.sh sets it to its checkout), else the
toolchain `eider toolchain show --json` names for this repository's `eider = "0.1"` (the
newest installed 0.1 release), else stops with exit 2 and says to install `eider`
(install.sh). eiderc and eider-pkg run through `eider` with that toolchain (`eider eiderc`,
`eider eider-pkg`), or directly when `eider` is not on the PATH. `EIDER_ROOT` and
`EIDER_LANG` no longer choose the toolchain, and `../eider-lang` is no longer the default.
The pinned `.vendor/<name>` fallback for a dependency applies when the toolchain is an
eider-lang checkout. Tested with the released Eider 0.1.0 and with eider-lang main. No
change to the library's code.

## 0.1.2

scripts/env.sh takes a dependency that has no `EIDER_<NAME>` and no sibling checkout from the
toolchain's pinned release in `$EIDER_ROOT/.vendor/<name>` when eider-lang's `vendor.lock`
pins it (its `scripts/vendor.sh fetch <name>` fetches and hash-checks it on first use), else
from its repository. The toolchain's old `vendor/` is no longer looked at (eider-lang #130 and
#124; no backwards compatibility). No change to the library's code.

Needs an Eider 0.1 toolchain; the pinned copy needs one with `vendor.lock` (eider-lang #122),
and without one a dependency comes from a sibling checkout or the network.

## 0.1.1

Imports in the dotted form (eider-lang C8.26, issue #91): `import std.bytes`, `import json`,
`import http.problem`, and this package's own modules by its name (`import openapi`,
`import openapi.schema` instead of `import "schema.ei"`, `import "../openapi.ei"`), in the
code, the tests, examples/todo (`import todo`), the README, docs/guide.md, docs/openapi.md
and the program scripts/build.sh generates to check every module.

Needs an Eider 0.1 toolchain whose compiler knows dotted imports (the eider-lang commit after
the next seed); an older 0.1 compiler stops at the first dotted import (E0001).

Also since 0.1.0: examples/todo changes its store in place with `sync.Lock.modify`
(eider-lang C320, issue #90).

## 0.1.0

The first release as a repository of its own, for Eider 0.1: eider-lang's
`packages/openapi` and `examples/todo`, with their history.

- http and json are git dependencies (eider-http 0.1.0, eider-json 0.1.0); examples/todo
  depends on this package by path (`../..`).
- docs/openapi.md (from eider-lang) and docs/guide.md (the REST half of eider-lang's
  Services guide).
- The todo example's description names eider-http and eider-openapi
  (`tests/openapi.golden.json` updated to match).
- `VERSION`, `eider = "0.1"`, `scripts/build.sh` and `scripts/test.sh` against an eider-lang
  checkout (`EIDER_ROOT`), refusing a toolchain of another MAJOR.MINOR, with
  `EIDER_PKG_OVERRIDE` set for sibling checkouts (scripts/env.sh).
- `scripts/version.sh` (`check`, `sync`): eider.toml agrees with VERSION; eider-lang's
  `scripts/version.sh check-siblings` and `bump` drive it.
