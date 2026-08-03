# 6. Process-level locking

**Issue:** CLI commands are highly concurrent. A user can run `zed install` in
two terminals, or CI can launch several installers against one shared
`ZED_PKG_HOME`. Without process coordination, callers can duplicate downloads,
publish partial cache entries, race extraction, or corrupt mutable project and
reference metadata.

## Design

zed-pkg uses descriptor-backed advisory lock files under
`~/.zed-pkg/locks/` through `fs2`:

- **Per-artifact locks** serialize acquisition of one SHA-256 while allowing
  unrelated hashes to download concurrently. Recursive prefetch and the
  transactional installer use the same acquisition function and therefore the
  same lock.
- **Install and build locks** protect lower-level operations that still mutate
  shared state and must not overlap unsafely.

The lock file is only a stable rendezvous path. Ownership belongs to the open
file descriptor. The operating system releases the lock when its owner drops
the descriptor or exits, so a crashed or killed `zed` process does not leave a
stale logical owner behind.

## Blocking, non-polling acquisition

`ProcessLock::acquire` opens the lock file and calls the operating system's
blocking exclusive-lock primitive directly:

- Unix uses descriptor-backed `flock`/`fcntl` semantics;
- Windows uses `LockFileEx` semantics.

A contended caller sleeps in the kernel and wakes when the owner releases the
descriptor or terminates. Production acquisition has no
`try_lock_exclusive` retry loop, spin interval, exponential backoff, jitter,
stale-PID reclamation, or periodic filesystem polling.

`ZED_PKG_LOCK_TIMEOUT` is intentionally not implemented by repeatedly probing
the lock. Operational cancellation belongs at the process or CI-job boundary;
the lock primitive itself remains event-driven.

## Per-artifact acquisition protocol

For an artifact with hash `<sha256>`:

1. Check the immutable content-addressed store.
2. Acquire `locks/artifact-<sha256>.lock`.
3. Re-check the store after waking; another process may have completed the
   artifact while this process slept.
4. Download to a temporary file in the cache directory.
5. Verify SHA-256 and declared artifact metadata.
6. Atomically rename the verified file into its final cache path.
7. Extract to a temporary directory and atomically rename the complete store
   entry into place.
8. Drop the lock guard.

A failed or interrupted download never becomes the final cache entry. A corrupt
cache entry is removed and replaced while the per-hash lock is held. Waiters
therefore observe either a complete immutable artifact or no artifact, never a
half-published one.

The lock is deliberately per content hash rather than global. Two processes
can make progress on different dependencies at the same time while still
downloading any one artifact at most once in aggregate.

## Project mutation boundary

Concurrent workers only resolve metadata and acquire immutable artifacts. The
existing install transaction remains responsible for lockfile writes, adapter
wiring, project references, rollback, and `zed_modules/` materialization.

On Unix, project packages remain symlinks into the global store by default.
Copy mode remains explicit for Docker/OCI and is the non-Unix fallback. Process
locking is not a reason to copy package trees into every project.

## Implementation provenance

- Core recursive graph resolution, five-worker queue, unified artifact
  acquisition, and tests originated in
  [`zed-pkg/zed-cli#53`](https://github.com/zed-pkg/zed-cli/pull/53).
- True parent/child artifact-lock tests and the permanent Windows workflow were
  reviewed in [`zed-pkg/zed-cli#65`](https://github.com/zed-pkg/zed-cli/pull/65).
- Kernel-blocking lower-level store/install/build locks plus orderly-release and
  forced-owner-exit regressions were reviewed in
  [`zed-pkg/zed-cli#67`](https://github.com/zed-pkg/zed-cli/pull/67).
- The composed recursive, recovery, workspace, uninstall, Linux, and Windows
  certification merged through
  [`zed-pkg-test/zed-pkg-e2e#14`](https://github.com/zed-pkg-test/zed-pkg-e2e/pull/14).

The #65 and #67 branch heads were later verified as ancestors of current
`zed-cli/main` and their stale stacked PRs were closed without replaying commits.
The implementation and regression behavior described here are therefore landed,
not merely proposed by those historical PR pages.

See also [24 — recursive installs and artifact locking](24-recursive-installs-and-artifact-locking.md).

## Required regressions

The permanent suites prove that:

- a separate process reaches acquisition and remains blocked while the owner
  holds the descriptor;
- the waiter wakes after an orderly guard drop;
- the waiter also wakes after the owning process is forcibly terminated;
- concurrent recursive prefetches sharing one home download one absent hash
  once in aggregate;
- recursive and transactional paths share one artifact lock;
- failed downloads leave no final cache/store publication or staging leak;
- the same process-lock contracts pass on Windows;
- multiple projects materialize symlinks to identical immutable store targets.

## Status

Implemented and externally certified. This document replaces the older polling,
backoff, jitter, stale-lock, and timeout design with descriptor-backed blocking
operating-system locks.
