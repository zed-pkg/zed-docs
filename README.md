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
| [6](docs/06-process-locking.md) | Process-level locking for concurrent CLIs | implemented; blocking OS locks + process regressions |
| [7](docs/07-enterprise-scale.md) | From concept to enterprise-grade | implemented* |
| [8](docs/08-fast-rust-ci.md) | Fast Rust CI (<3 min) | implemented |
| [9](docs/09-cross-platform-distribution.md) | Multi-OS / multi-arch CLI distribution | implemented |
| [10](docs/10-e2e-testing.md) | End-to-end testing across servers, CLI, and browsers | implemented |
| [11](docs/11-kubernetes-deployment.md) | Deploying the registry to Kubernetes (GitOps app-of-apps) | implemented |
| [12](docs/12-in-cluster-e2e.md) | In-cluster e2e (kind + in-memory profile + Argo CD) | implemented |
| [13](docs/13-remote-browser-grid-e2e.md) | Remote browser-grid e2e (ORES clusters, AWS + Hetzner) | partial |
| [14](docs/14-client-sync-and-opto-sync-clients.md) | Client-side sync patterns + opto-sync package adoption | package-ready; migration staged |
| [15](docs/15-manifest-and-dep-locations.md) | The manifest, where deps go, complementing npm/maven | implemented |
| [16](docs/16-zed-pkg-test-ci.md) | zed-pkg-test CI harness (GitHub Actions only) | node + rust proven |
| [17](docs/17-polyglot-client-libraries.md) | Polyglot client libraries: one repo, one package per language | implemented |
| [18](docs/18-multi-registry-release-fanout.md) | Native registry fan-out and target-only forge mirrors | direction set |
| [19](docs/19-polyglot-publishing-audit.md) | Polyglot publishing: what is actually verified (audit of 18) | verified in CI |
| [20](docs/20-repository-sync-and-semantic-merging.md) | Repository synchronization and semantic conflict resolution | operational runbook |
| [21](docs/21-offline-release-plan-review.md) | Deterministic offline release-plan review in a browser | implemented; three-engine + print/a11y verified |
| [22](docs/22-flags2env-browser-wasm.md) | Real flags2env C parser in browser WebAssembly and workers | implemented; three-engine verified |
| [23](docs/23-nix-zed-interop.md) | Nix–Zed interoperability through sealed, immutable adapter records | proposal; implementation staged |
| [24](docs/24-recursive-installs-and-artifact-locking.md) | Recursive dependency graphs, five-worker prefetch, and per-artifact locks | implemented; external E2E certified |
| [25](docs/25-complete-constraint-solving.md) | Complete one-version solving for overlapping transitive ranges | implemented; independent black-box certification |
| [26](docs/26-deterministic-nix-export-bundles.md) | Pure deterministic Zed → Nix flake-bundle rendering contract | implementation under review |
| [27](docs/27-durable-first-install-manifests.md) | Durable `.zpkg.toml` creation on first dependency install | implemented; Node + Go/Python/Rust certified on Linux/macOS |
| [28](docs/28-universal-environment-interop.md) | Flox, Devbox, mise/asdf, and scratch OCI interoperability | RFC; executable policy validated |
| [28](docs/28-zed-lock-evented-cross-platform-locking.md) | `zed-lock`: helper-thread async adapters over kernel-backed cross-process locks | architecture approved; extraction tracked |
| [29](docs/29-native-dependencies-and-install-hooks.md) | Native prerequisites, install hooks, consent boundaries, staging, caching, and Nix purity | manifest contract implemented; installer lifecycle under review |
| [30](docs/30-submodules-zed-packages-and-composition.md) | Git submodules vs Zed packages, workspaces, release composition, and deployment boundaries | operational policy; pack guard under review |
| [31](docs/31-zed-lock-evented-cross-platform-locking.md) | `zed-lock`: helper-thread async adapters over kernel-backed cross-process locks | architecture approved; extraction tracked |
| [32](docs/32-org-repository-package-pattern.md) | Organization clients/interfaces/lib/CLI/monorepo package pattern and fleet audit | operational policy; rollout audited |
| [33](docs/33-github-linear-project-registry.md) | Canonical GitHub organization, Linear project, GitHub Project, and artifact ownership registry | operational policy; Projects permission blocked |
| [29](docs/29-native-dependencies-and-install-hooks.md) | Native prerequisites, install hooks, consent boundaries, staging, caching, and Nix purity | manifest contract implemented; installer lifecycle under review |

Independent executable acceptance contracts that do not consume architecture
numbers are indexed separately. The first is the
[`zed develop` clean-room acceptance contract](docs/zed-develop-clean-room-acceptance.md),
which is already on `main` and remains deliberately unnumbered.
| [23](docs/23-universal-environment-interop.md) | Flox, Devbox, mise/asdf, and scratch OCI interoperability | RFC; implementation staged |
| [24](docs/24-deterministic-nix-export-bundles.md) | Pure deterministic Zed → Nix flake-bundle rendering contract | implementation under review |

<sub>\* Enterprise features are implemented except SSO and per-org **storage**
quotas, which remain planned. Audit logs shipped (`zed org audit`,
`GET /v1/orgs/{org}/audit`); the quota that exists today is the org-claim
squatting limit, which is a different thing. See
[doc 7](docs/07-enterprise-scale.md).</sub>

## The model in one paragraph

A package is `<org>/<name>` with a `.zpkg.toml` manifest (TOML only). Its
source of truth is a repo on any git/hg/jj/sapling/fossil/pijul host;
`zpkg.tech` is the primary artifact host and the forge is the mirror +
provenance anchor. `zed publish` packs a pruned, deterministic tarball (tests,
CI, `.github/`, READMEs stripped; licenses kept), verifies a matching VCS tag
at HEAD, and uploads. `zed install` resolves semver, downloads each artifact
once into a content-addressed store at `$HOME/.zed-pkg`, verifies its sha256,
and symlinks it into the project's `zed_modules/` — pnpm-style.

## Governance

Organization contribution and review rules are in [CONTRIBUTING.md](CONTRIBUTING.md).
In particular, a pull request may not be closed as superseded until its successor
incorporates and traces at least one substantive item from every predecessor.

This documentation is MIT licensed. See [LICENSE](LICENSE). Report suspected
security issues using the private-first procedure in [SECURITY.md](SECURITY.md),
not a public issue containing exploit details or credentials.
