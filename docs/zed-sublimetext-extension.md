# zed-sublimetext package-insights extension

Updated: 2026-08-05

## Ownership and system mapping

| System | Record |
| --- | --- |
| GitHub organization | `zed-pkg` |
| Repository | `zed-pkg/zed-sublimetext` |
| Linear project | `github.com/zed-pkg` |
| Linear issue | `DEN-2326` |
| Linear delivery document | `zed-sublimetext extension delivery and project tracking contract` |
| GitHub Project | `zed-pkg-project` |
| Shared CLI dependency | `zed-pkg/zed-cli#191` |

`zed-pkg` is an independent multi-language package manager and is unrelated to
the Zed editor. This repository is the Sublime Text integration for the
`zed-pkg` package manager.

## Delivered implementation

The repository provides a native Sublime Text Python plugin that:

- discovers the active Zed package root;
- inspects `.zpkg.toml`, `.zpkg.lock`, materialization state, and interrupted
  `.zpkg-staging` transactions;
- distinguishes the package-manager CLI from the unrelated Zed editor launcher;
- renders diagnostics and recommended actions in Sublime Text;
- requires explicit confirmation before every mutating action;
- executes commands as argument arrays rather than shell-interpolated strings;
- redacts credentials and token-shaped values before displaying process output;
- supports Sublime's Python 3.8 compatibility environment and the current
  Python 3.14 host mapping through a vendored MIT-licensed TOML parser.

The extension performs a deterministic local read-only subset until
`zed-pkg/zed-cli#191` delivers the shared non-mutating JSON inspection contract.
It must not become an independent package resolver.

## Immutable delivery evidence

| Evidence | Value |
| --- | --- |
| Initial implementation commit | `2776656d4c3e24218b1b758f0462190b4fc1f5c6` |
| Python 3.8 TOML compatibility commit | `08e8255128d0c138ae2506bc267036584bc5908f` |
| Packaging PR | `zed-pkg/zed-sublimetext#1` |
| Reviewed PR head | `1e278d7373c3e138a871b3f09104200addd18138` |
| Squash merge commit | `358c746552601d472f7878b43882d9d0bec0bf2b` |
| Post-merge workflow | run `31037791942` |
| Local preflight | 18 tests passed; deterministic package build verified |
| Local preflight artifact SHA-256 | `3446389f72ffcd97286de712726c26397f01cbd268fb402b30bdfe0af7dc02e3` |

The local digest is preflight evidence only. The authoritative retained artifact
is the commit-addressed GitHub Actions artifact produced from the merge commit.
At the time of this record, workflow run `31037791942` was queued and had not yet
produced that retained artifact.

## Packaging contract

`scripts/build-package.py` creates
`dist/ZedPackageInsights.sublime-package` with deterministic file ordering,
timestamps, permissions, and compression. Development-only paths, test sources,
GitHub metadata, and Zed package-publication metadata are excluded.

The CI contract is:

1. compile and test under Python 3.8;
2. compile and test under Python 3.14;
3. build the deterministic Sublime Text package;
4. verify archive contents and byte-for-byte reproducibility;
5. upload `ZedPackageInsights-<commit-sha>` with bounded retention.

## GitHub Project operating record

All implementation, release, Package Control, compatibility, and CLI-contract
work belongs in the organization project titled `zed-pkg-project`. The canonical
registry currently records the project as `permission-blocked` because the
available GitHub app cannot mutate Projects v2 and the current runtime does not
provide working authenticated `gh project` access.

Required board fields:

- **Status:** Backlog, Ready, In Progress, In Review, Done
- **Priority:** Urgent, High, Medium, Low
- **Area:** IDE Integration, CLI Contract, Packaging, Documentation, Testing
- **Repository:** `zed-sublimetext`

## Remaining promotion gates

1. workflow run `31037791942` completes successfully;
2. retain and checksum its `ZedPackageInsights-358c746...` artifact;
3. install the artifact in a clean Sublime Text profile on supported platforms;
4. verify healthy, stale-lock, lock-only, invalid-TOML, and interrupted-
   transaction fixtures;
5. submit or update the package in Package Control;
6. add the delivery issue and release work to `zed-pkg-project` when Projects v2
   mutation access is available;
7. update `DEN-2326` with terminal workflow, artifact, installation, and
   publication evidence before marking Done.
