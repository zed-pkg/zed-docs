# 9. Multi-OS / multi-arch CLI distribution

**Issue:** shipping a compiled CLI across operating systems and architectures
— especially the shift to ARM64 via Apple Silicon and AWS Graviton — is where
Rust projects stumble.

## Design

The `zed` binary is released for a full target matrix on every `v*` tag via
[`zed-cli/.github/workflows/release.yml`](https://github.com/zed-pkg/zed-cli/blob/main/.github/workflows/release.yml):

| OS | Targets |
| --- | --- |
| macOS | `aarch64-apple-darwin` (Apple Silicon), `x86_64-apple-darwin` (Intel) |
| Linux | `aarch64-unknown-linux-gnu`, `x86_64-unknown-linux-gnu` |
| Linux (static) | `aarch64-unknown-linux-musl`, `x86_64-unknown-linux-musl` |
| Windows | `x86_64-pc-windows-msvc` |

- **ARM64 first-class.** Apple Silicon and Graviton both get native builds;
  Linux arm64/x64 cross-compile via [`cross`](https://github.com/cross-rs/cross).
- **musl static builds** have no libc dependency — drop them straight into
  `scratch`/`distroless` containers.
- Each target uploads a `zed-<target>.tar.gz` (or `.zip` on Windows) to the
  GitHub Release; release notes are auto-generated.

## Install channels

- **Homebrew:** [`Formula/zed-pkg.rb`](https://github.com/zed-pkg/zed-cli/blob/main/Formula/zed-pkg.rb)
  (`brew tap zed-pkg/tap && brew install zed-pkg`). Note the documented
  conflict with the Zed editor's `zed` binary.
- **From source:** `cargo install --path .` (or `--bin zed`).
- **Containers:** copy the musl binary into any base image; artifacts are
  pre-pruned so images stay small.

## Self-update

`zed update self` upgrades the CLI in place: it resolves the latest release by
following the `/releases/latest` redirect (no API token, no rate-limit),
compares semver against the running build, downloads the
`zed-<target>.{tar.gz,zip}` asset matching this platform (arch + OS + gnu/musl),
and atomically replaces the running binary — safe on Unix because the running
process keeps its open inode. `--check` reports without installing; `--force`
reinstalls. See
[`zed-cli/src/update.rs`](https://github.com/zed-pkg/zed-cli/blob/main/src/update.rs).

## Status: implemented

The release matrix, `cross` setup, musl static builds, archived per-target
uploads, and `zed update self` exist today. Planned: publishing the Homebrew
formula to a `zed-pkg/homebrew-tap` tap automatically on release, plus Scoop
(Windows) and a `curl | sh` installer that detects OS/arch.
