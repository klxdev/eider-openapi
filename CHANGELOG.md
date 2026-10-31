# Changelog

eider-openapi's versions (semver). Before 1.0 the MAJOR.MINOR is the Eider toolchain's it
goes with (`eider` in eider.toml), the PATCH its own.

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
