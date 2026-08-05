# zed-pkg organization delivery status

Updated: 2026-08-05

This document is the version-controlled GitHub-side companion to the Linear project `github.com/zed-pkg`. It records merged delivery evidence and the current promotion gates for work spanning repositories in the `zed-pkg` and `zed-pkg-test` organizations.

## System-of-record boundary

- GitHub pull requests and immutable merge commits are the source of truth for implementation and review evidence.
- GitHub Actions exact-head runs are the source of truth for test and policy gates.
- Linear issues hold planning, ownership, dependency, and acceptance state.
- The organization GitHub Project should link the relevant PR or issue rather than duplicate implementation detail.
- A work item is marked Done only after its reviewed head is merged or an explicit non-code acceptance artifact is published.

## Recently completed

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

- canonical Zed-package validation for `zed-interfaces`;
- complete pinned Nix replay for the ten-runtime `zed-clients` SDK matrix;
- cross-browser integrity certification for retained release-plan reports;
- complete current `mise.lock` identity and deterministic conflict-safe export;
- consolidated OCI planner, image-layout, and authenticated ORAS transport;
- reusable `zed-pkg-test` candidate and full-certification gates.

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

| GitHub organization | Linear project | GitHub Project title |
| --- | --- | --- |
| `zed-pkg` | `github.com/zed-pkg` | `zed-pkg-project` |
| `zed-pkg-test` | `github.com/zed-pkg` | `zed-pkg-test-project` |

The test organization remains part of the same Linear program because it supplies independent consumer, browser, lifecycle, install-boundary, and release-certification evidence for product repositories.
