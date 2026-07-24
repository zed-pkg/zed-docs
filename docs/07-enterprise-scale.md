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

## 3. Executing binaries (implemented)

Packages declare executables in a `[bin]` table (command name → a path inside
the package). On install, each is hoisted into the project's
`zed_modules/.bin/<name>` as a relative symlink
([`hoist_bins`](https://github.com/zed-pkg/zed-cli/blob/main/src/ops.rs)), and
`zed run <bin> [args]` executes one with that directory prepended to `PATH`
([`run_bin`](https://github.com/zed-pkg/zed-cli/blob/main/src/ops.rs)) —
npx-style, scoped to the project's resolved versions and without polluting the
OS `PATH`. Adapter-linked layouts ([2](02-store-project-bridge-oci.md))
additionally let native runners find dependency binaries where they expect
them. Planned: an opt-in global shim on `PATH` so top-level tools run without
the `zed run` prefix.

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

## Monorepo ergonomics (implemented)

A workspace root manifest declares member globs in `[workspace]`
(`members = ["packages/*", "apps/*"]`). `zed install` walks up to find the
enclosing workspace and expands the globs
([`find_workspace`/`collect_members`](https://github.com/zed-pkg/zed-cli/blob/main/src/ops.rs)),
then resolves any dependency that matches a member by symlinking straight to
the member's **source** directory instead of going through the registry — so
an edit in one member is visible to its consumers immediately, while
non-member deps still resolve normally against the shared store and lock.
Planned: a single top-level lock spanning every member (today each member
install writes its own `.zpkg.lock`) and workspace-wide `zed run`.
