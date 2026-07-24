# 7. From concept to enterprise-grade

**Issue:** what does zed-pkg need so a team of 50 developers can use it across
a monorepo — executing binaries, and not filling their hard drives?

Four capabilities, and where each stands:

## 1. Disk efficiency at scale (implemented)

One content-addressed store per machine ([1](01-cas-and-symlinks.md)); every
project symlinks in. 50 developers × N projects with heavy overlap store each
dependency **once** per machine. `zed store status` reports usage (store,
download cache, and build cache); `zed store prune` garbage-collects artifacts
no live project references (tracked in `refs.json`); `zed cache clean` drops
downloads; and `zed gc [--older-than 90d] [--dry-run]` is a least-recently-used
sweep over the store, build cache, and downloads by access time — everything it
removes is content-addressed and re-fetchable.

## 2. Deterministic, concurrent installs (implemented)

`--frozen` installs exactly the locked sha256s
([4](04-lockfile-and-tag-immutability.md)); process locking makes parallel CI
runners and multiple terminals safe ([6](06-process-locking.md)). Same lock →
same bytes on every machine.

## 3. Executing binaries (implemented)

Packages expose executables via `[bin]` (`name = "path"`). On install, zed
hoists each into `zed_modules/.bin/` (a real file in copy mode, a symlink in
symlink mode, so it works in containers too), and `zed run <name> [args]`
executes it with `zed_modules/.bin` prepended to `PATH` — the project's tools
resolve to the versions it installed, without polluting the global `PATH`.
See `hoist_bins`/`run` in
[`zed-cli/src/ops.rs`](https://github.com/zed-pkg/zed-cli/blob/main/src/ops.rs).

## 4. Governance (implemented core; SSO/audit/quotas planned)

- **Namespaces & RBAC:** orgs are claimed and tokens are org-scoped
  ([`zed org claim`](https://github.com/zed-pkg/zed-cli)); each token carries a
  **role** — `owner`, `publisher`, or `reader` — set with
  `create-token --org <slug> --role <role>`. Publishing enforces it: a
  `reader` token gets `403 insufficient_role`, a token for another org is
  rejected, and unscoped admin tokens publish anywhere (npm-style granular
  tokens). See
  [`api-server/src/rbac.rs`](https://github.com/zed-pkg/zed-api-server.rs).
- **Self-hosting:** the whole registry is two small Rust services + Postgres +
  any S3 bucket, so a company runs a private registry behind its firewall
  ([zed-infra](https://github.com/zed-pkg/zed-infra)); `ZED_PKG_REGISTRY`
  (or a `file://` mirror) repoints the CLI.
- **Planned:** multi-user teams with per-member roles, audit logs, SSO,
  mirror/proxy of the public registry, and per-org storage quotas.

## Monorepo ergonomics (implemented)

Workspace mode: a root `.zpkg.toml` with `[workspace] members = ["packages/*",
"apps/*"]` makes `zed install` resolve every member against one store and write
one `.zpkg.lock`. Member→member dependencies link by **path** to the member's
source (live editing, no publish), while external deps resolve from the shared
store; zed's "one version per package" rule holds workspace-wide. See
`install_workspace` in
[`zed-cli/src/ops.rs`](https://github.com/zed-pkg/zed-cli/blob/main/src/ops.rs).
