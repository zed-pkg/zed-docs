# 6. Process-level locking

**Issue:** CLI commands are highly concurrent. A user runs `zed install` in
two terminals, or a CI pipeline spins up 10 parallel runners that all trigger
an install. Without locking, multiple processes race to download, extract, and
symlink the same package into the same store and corrupt it.

## Design

zed-pkg takes an advisory `flock` on lock files under `~/.zed-pkg/locks/`:

- **Per-artifact lock** around extraction: only one process extracts a given
  sha256 at a time; the rest block, then observe `has(sha)==true` and skip.
  Extraction still goes to a temp dir followed by an atomic rename as a second
  line of defense.
- **Per-install lock** held for the duration of `zed install`, serializing
  the `refs.json` and `.zpkg.lock` writes.

Advisory `flock` is the right primitive because the **OS releases it when the
process exits** — a crashed or killed `zed` never leaves a stale lock that
wedges the store. (A naive lockfile-with-PID would.)

## Acquisition: three phases

`ProcessLock::acquire` polls `try_lock_exclusive` in a loop tuned for CLI
latency rather than a single blocking `lock_exclusive`:

1. **Spin-wait (50ms)** while the wait is young (< 500ms): a busy lock is
   almost always released within a fraction of a second, so a tight 50ms
   retry acquires it with no perceptible lag.
2. **Exponential backoff with full jitter** past 500ms: the sleep grows
   `100ms · 2^n` capped at 2s, and each wait is drawn uniformly from
   `[0, cap]` so N parallel CI runners don't wake in lockstep and thundering-
   herd the lock.
3. **Observability yield at 3s**: once a wait crosses 3 seconds the process
   prints what it is blocked on (`waiting for <lock> (held by another zed
   process)…`) instead of freezing silently, then keeps backing off until an
   overall timeout (`ZED_PKG_LOCK_TIMEOUT`, default 600s), after which it
   fails with an actionable message rather than hanging forever.

## Implementation

[`zed-cli/src/store.rs`](https://github.com/zed-pkg/zed-cli/blob/main/src/store.rs)
— `ProcessLock::acquire` (built on `fs2::FileExt::try_lock_exclusive` with the
three-phase spin/backoff/notify loop above), `install_lock()`,
`build_lock()`, and the per-sha lock inside `add_artifact`.

The test `concurrent_installs_share_the_store_safely`
([`zed-cli/tests/e2e.rs`](https://github.com/zed-pkg/zed-cli/blob/main/tests/e2e.rs))
launches 8 installers against one shared `ZED_PKG_HOME` and asserts the store
ends with exactly one extracted copy.

## Status: implemented
