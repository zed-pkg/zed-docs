# 8. Fast Rust CI (under 3 minutes)

**Issue:** Rust's compile times are notorious; an unoptimized pipeline makes
every PR a 15-minute wait and destroys velocity. Keep CI under ~3 minutes.

## What zed-pkg does today

- **Dependency caching.** Every Rust repo's CI uses
  [`Swatinem/rust-cache`](https://github.com/Swatinem/rust-cache), which
  caches the registry index, downloaded crates, and `target/` keyed on
  `Cargo.lock` + toolchain. Most PRs recompile only changed crates.
- **Thin crates + a shared contract.** `zed-interfaces` is a tiny leaf crate
  the others depend on by path, so its compile is cheap and cached; the
  servers and CLI are separate build units that CI runs in parallel jobs.
- **Hermetic, fast tests.** The CLI's whole publish→install→test-local suite
  runs against a `file://` registry and a temp `ZED_PKG_HOME` — no network, no
  Postgres, no containers in the unit path. Wall-clock is dominated by
  `cargo build`, not test setup.
- **`fmt --check` as a separate fast gate** fails style issues in seconds
  without waiting on the full build.
- **Faster linking and test running.** `zed-cli`'s CI links with
  [`mold`](https://github.com/rui314/mold) on Linux — wired in as the default
  system linker via `rui314/setup-mold`, so `cargo`/`rustc` use it with no
  extra `RUSTFLAGS` — and runs tests with
  [`cargo nextest`](https://nexte.st) for parallel execution and cleaner
  output, keeping `cargo test --doc` alongside since nextest doesn't run
  doctests.

## Recommended additions (documented, opt-in)

- **`sccache`** with a shared/cloud backend for cross-run compilation caching.
- **`cargo-chef`** in Dockerfiles to cache the dependency-build layer so image
  builds only recompile app code (pairs with the parent-context build the
  Rust services already use).
- **`-Zshare-generics` / thin LTO off in dev**, `debug = 0` for test profiles,
  and splitting the workspace so unrelated crates don't serialize.
- Pinning the toolchain via `rust-toolchain.toml` so the cache key is stable.

## Status: implemented (baseline) + documented (advanced)

The caching + parallel-jobs + hermetic-tests baseline is in every repo's
`.github/workflows/ci.yml`. `mold` (Linux) and `cargo nextest` are now wired
into `zed-cli`'s CI and are being rolled out to the server repos (still on
plain `cargo test`). `sccache`/`cargo-chef` remain documented-only next steps,
kept out of the default CI to stay dependency-light.
