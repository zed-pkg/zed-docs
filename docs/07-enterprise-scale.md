# 7. From concept to enterprise-grade

**Issue:** what does zed-pkg need so a team of 50 developers can use it across
a monorepo — executing binaries, and not filling their hard drives?

Four capabilities, and where each stands:

## 1. Disk efficiency at scale (implemented)

One content-addressed store per machine ([1](01-cas-and-symlinks.md)); every
project symlinks in. 50 developers × N projects with heavy overlap store each
dependency **once** per machine. `zed store status` reports usage;
`zed store prune` garbage-collects artifacts no live project references
(tracked in `refs.json`); `zed gc [--max-age-days N]` adds an age-aware LRU
sweep that removes store entries that are *both* unreferenced by a live
project *and* unused past the cutoff — tracked with a `.last-used` marker
alongside the immutable `pkg/` tree — plus stale download caches
([`Store::gc`](https://github.com/zed-pkg/zed-cli/blob/main/src/store.rs));
`zed cache clean` drops downloads.

## 2. Deterministic, concurrent installs (implemented)

`--frozen` installs exactly the locked sha256s
([4](04-lockfile-and-tag-immutability.md)); process locking makes parallel CI
runners and multiple terminals safe ([6](06-process-locking.md)). Same lock →
same bytes on every machine.

## 3. Executing binaries (design)

Packages may expose executables; `zed run <org>/<name> [args]` and a shimmed
`~/.zed-pkg/bin` on `PATH` (resolving to the locked version per project) are
specified and planned. Today, adapter-linked layouts
([2](02-store-project-bridge-oci.md)) let native runners find dependency
binaries.

## 4. Governance (partial → planned)

- **Namespaces:** orgs are claimed and tokens are org-scoped today
  ([`zed org claim`](https://github.com/zed-pkg/zed-cli),
  [api-server orgs](https://github.com/zed-pkg/zed-api-server.rs)).
- **Self-hosting:** the whole registry is two small Rust services + Postgres +
  any S3 bucket, so a company runs a private registry behind its firewall
  ([zed-infra](https://github.com/zed-pkg/zed-infra)); `ZED_PKG_REGISTRY`
  (or a `file://` mirror) repoints the CLI.
- **Planned:** RBAC/teams per org, audit logs, SSO, mirror/proxy of the public
  registry, and per-org storage quotas (tie-in with pricing).

## Monorepo ergonomics (planned)

A workspace mode (`zed install` at the repo root resolving many
`.zpkg.toml` members against one lock and one store) is the main missing piece
for large monorepos and is next after workspaces land in the resolver.
