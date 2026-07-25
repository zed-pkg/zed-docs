# zed-docs

Architecture and design docs for [zed-pkg](https://zpkg.tech), the universal
package manager backed by the VCS hosts you already use.

Each doc below answers one of the [tracked issues](https://github.com/zed-pkg/zed-docs/issues),
in order. Where a design is already implemented, the doc links to the code.

| # | Topic | Status |
| --- | --- | --- |
| [1](docs/01-cas-and-symlinks.md) | CAS + symlinks: dependency fetching decoupled from build | implemented |
| [2](docs/02-store-project-bridge-oci.md) | Global store <-> project bridge under OCI | implemented |
| [3](docs/03-version-normalization.md) | Polyglot version strings (git/hg tags, commits) | implemented |
| [4](docs/04-lockfile-and-tag-immutability.md) | Lockfile vs mutable tags/branches | implemented |
| [5](docs/05-source-vs-build-cache.md) | Source caching vs build caching | implemented |
| [6](docs/06-process-locking.md) | Process-level locking for concurrent CLIs | implemented |
| [7](docs/07-enterprise-scale.md) | From concept to enterprise-grade | implemented* |
| [8](docs/08-fast-rust-ci.md) | Fast Rust CI (<3 min) | implemented |
| [9](docs/09-cross-platform-distribution.md) | Multi-OS / multi-arch CLI distribution | implemented |
| [10](docs/10-e2e-testing.md) | End-to-end testing across servers, CLI, and browsers | implemented |

<sub>\* Enterprise features are implemented except SSO, audit logs, and per-org
storage quotas, which remain planned (see [doc 7](docs/07-enterprise-scale.md)).</sub>

## The model in one paragraph

A package is `<org>/<name>` with a `.zpkg.toml` manifest (TOML only). Its
source of truth is a repo on any git/hg/jj/sapling/fossil/pijul host;
`zpkg.tech` is the primary artifact host and the forge is the mirror +
provenance anchor. `zed publish` packs a pruned, deterministic tarball (tests,
CI, `.github/`, READMEs stripped; licenses kept), verifies a matching VCS tag
at HEAD, and uploads. `zed install` resolves semver, downloads each artifact
once into a content-addressed store at `$HOME/.zed-pkg`, verifies its sha256,
and symlinks it into the project's `zed_modules/` — pnpm-style.
