# 5. Source caching vs build caching

**Issue:** source code is universal, but compiled artifacts are tied to OS and
CPU architecture. Extracting a tarball is not enough when a package has a C
extension or native code — it has to be built for `linux-x64` in the
container but `darwin-arm64` on the laptop.

## Two distinct caches

zed-pkg deliberately separates them:

1. **Source store (implemented).** `~/.zed-pkg/store` is
   content-addressed by the **source artifact** sha256 and is
   platform-independent. This is what `zed install` populates and symlinks.
   It never contains compiled output, so it is safe to share across machines,
   architectures, and OCI base images.

2. **Build cache (implemented).** A package's optional `[build]` step
   (a `command` plus the `outputs` to keep) runs after extraction, and its
   compiled output is keyed by `(source sha256, platform)` — where `platform`
   is the `os-arch` pair from
   [`current_platform`](https://github.com/zed-pkg/zed-interfaces/blob/main/src/paths.rs)
   — and stored separately at `~/.zed-pkg/builds/v1/<platform>/<sha>/pkg`
   ([`build_entry_rel`](https://github.com/zed-pkg/zed-interfaces/blob/main/src/paths.rs)).
   A cache miss triggers a build; a hit reuses it. Because the key includes
   the platform, `linux-x86_64` and `macos-aarch64` results never collide, and
   a CI container and a laptop maintain independent build caches over the
   *same* source store.

## Why keep them separate

- The source store stays universal and reproducible — the disk-efficiency and
  provenance guarantees ([1](01-cas-and-symlinks.md),
  [4](04-lockfile-and-tag-immutability.md)) depend on source bytes being the
  same everywhere.
- Build outputs are inherently non-portable; mixing them into the source
  store would break cross-machine sharing and bloat images.
- It maps cleanly onto multi-stage Docker: restore the build cache with
  `--mount=type=cache,target=/root/.zed-pkg/build/<target>`, install with
  copy-mode source ([2](02-store-project-bridge-oci.md)).

## Status: design

The source store is implemented. The build-cache keying, `zed build`
integration, and post-install build steps are specified here and planned;
today zed-pkg distributes source (and prebuilt-binary packages that need no
build step).
