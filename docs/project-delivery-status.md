# zed-pkg organization delivery status

Updated: 2026-08-05

This document is the version-controlled GitHub-side companion to the Linear project `github.com/zed-pkg`. It records merged delivery evidence and the current promotion gates for work spanning repositories in the `zed-pkg` and `zed-pkg-test` organizations.

Linear tracking issue: `DEN-2211`.

## System-of-record boundary

- GitHub pull requests and immutable merge commits are the source of truth for implementation and review evidence.
- GitHub Actions exact-head runs are the source of truth for test and policy gates.
- Retained GitHub Actions artifacts are the source of truth for certified package bytes and their provenance metadata.
- Linear issues hold planning, ownership, dependency, and acceptance state.
- The organization GitHub Project should link the relevant PR or issue rather than duplicate implementation detail.
- A work item is marked Done only after its reviewed head is merged or an explicit non-code acceptance artifact is published.

## Recently completed

### Certified Zed package artifacts

- Repository: `zed-pkg/zed-interfaces`
- Pull request: `#42`
- Reviewed head: `b5608b49195b50de5a7068c37892e5f46710fc4e`
- Merge commit: `d36ac522915792539740cb105e928652503dfde2`
- Result: the reusable `certify-zed-package.yml` workflow packs the canonical package and every declared target, writes deterministic `SHA256SUMS` plus `zed-artifacts.json`, and uploads the exact validated archives as a retained GitHub Actions artifact with read-only permissions.

### Browser/WASM transport certification

- Repository: `zed-pkg/zed-clients`
- Pull request: `#13`
- Reviewed head: `93df9b5fa23691fb7f6aadd8854eb9b8dfd6387a`
- Merge commit: `cf6faede6835f790a1e92d0e22735bae2342818b`
- Linear: `DEN-1500`
- Result: the real Rust/WASM client is exercised in Chromium, Firefox, and WebKit against hostile transport fixtures while preserving the independently merged production hardening.

### Collision-safe shared-stack E2E

- Repository: `zed-pkg/zed-e2e`
- Pull request: `#12`
- Reviewed head: `241ec9f3510b6dba1ce147ca082834021f15ed04`
- Merge commit: `9097ffa7dd55537399c8b712e587cb47f3d61ce1`
- Linear: `DEN-1503`
- Result: PostgreSQL, API, and web ports are allocated as one run-scoped set; implicit collisions fall back safely while explicit overrides remain exact and fail closed.

### Three-driver browser harness reconstruction

- Repository: `zed-pkg/zed-e2e`
- Pull request: `#15`
- Superseded pull request: `#6`
- Reviewed head: `37371604d077f07be5b8990c37e00228b52cb93d`
- Merge commit: `2b2f7cb13398b90b4d0af843d51c209914bc3d8a`
- Linear: `DEN-1595`
- Result: Playwright, Puppeteer, and Selenium contracts were reconstructed on current `main`, preserving newer E2E work and changing only the intended harness files and shared-stack serialization settings.

### Independent release-report consumer

- Repository: `zed-pkg/zed-e2e`
- Pull request: `#11`
- Reviewed head: `7d5991770d4d53d8e6e4c5678498e50ddb5bb660`
- Merge commit: `a3af72538e6bc14a942fb0fedcfb0c6b3343c45d`
- Linear: `DEN-1499`
- Result: the retained release plan, HTML report, and integrity manifest are independently verified and reviewed through Playwright, Puppeteer, and Selenium.

### Bounded artifact-storage observability

- Repository: `zed-pkg/zed-api-server.rs`
- Pull request: `#11`
- Merge commit: `9f5bfd437c4bf3e32a6454db85ff443bf1dd2e48`
- Linear: `DEN-1709` — Done
- Result: low-cardinality Prometheus gauges for storage backend, object count, used bytes, capacity, and utilization.

### Registry process modularization

- Repository: `zed-pkg/zed-api-server.rs`
- Pull request: `#13`
- Superseded pull request: `#12`
- Reviewed head: `f77d77dfc5327ccdac49a207205a2c070ac4eb12`
- Merge commit: `4eaacda14d5c5e38b81404415631e364ac7bdb2c`
- Linear: `DEN-1726` — Done
- Result: `src/main.rs` is a thin Tokio entrypoint; process orchestration and pure command/healthcheck tests live in `src/server.rs`.

The replacement branch was reconstructed from current `main` after observability merged. It preserved both changes instead of resolving divergence by selecting one history side.

## Active promotion gates

The following work should remain open until its exact reviewed head is terminal-successful and review-clean:

- rollout of the certified-package reusable workflow to each interface repository;
- complete pinned Nix replay for the expanded `zed-clients` SDK matrix;
- complete current `mise.lock` identity and deterministic conflict-safe export;
- consolidated OCI planner, image-layout, and authenticated ORAS transport;
- reusable `zed-pkg-test` candidate and full-certification gates;
- project-local mutation serialization and its follow-up concurrency refinements.

## GitHub Project item format

For each organization-level item, use:

- **Title:** concise deliverable name;
- **Repository:** owning `owner/repo`;
- **Linear:** issue identifier;
- **PR:** owning pull request;
- **Candidate SHA:** exact reviewed head;
- **Merge SHA:** populated only after merge;
- **Status:** Backlog, In progress, In review, Blocked, or Done;
- **Gate:** the remaining exact-head CI, review, release, credential, or environment condition;
- **Evidence:** immutable workflow run, artifact digest, or merge commit.

Do not mark a project item Done merely because code exists on a branch or a pull request is mergeable. Do not merge a stale stack by choosing one conflict side when a current-main semantic reconstruction is safer.

## Organization mapping

| GitHub organization | Linear project | GitHub Project title | Scope |
| --- | --- | --- | --- |
| `zed-pkg` | `github.com/zed-pkg` | `zed-pkg-project` | Product, interfaces, SDKs, CLI, API, docs, infrastructure, and release evidence |
| `zed-pkg-test` | `github.com/zed-pkg` | `zed-pkg-test-project` | Independent consumer, browser, lifecycle, install-boundary, and release certification |

The test organization remains part of the same Linear program because it supplies independent evidence for product repositories.

For additional organizations, use the same convention: one organization project titled `<organization>-project` and one Linear project titled `github.com/<organization>`, unless a documented shared-program boundary deliberately maps multiple organizations to one Linear project as `zed-pkg-test` does here.

## Promotion procedure

1. Create or update the Linear issue and attach the canonical repository/PR.
2. Add the issue or PR to the owning organization GitHub Project.
3. Record the exact candidate SHA and remaining gate.
4. Require terminal exact-head checks and no unresolved review threads.
5. Reconstruct stale/conflicted work on current `main` when necessary, preserving the semantic union.
6. Merge with an expected-head guard.
7. Record the merge SHA and retained artifact identity in Linear, this record, and the GitHub Project item.
8. Mark all systems Done only after the remote merge or explicit acceptance artifact exists.
