# 24. Recursive installs and artifact locking

**Issue:** a Zed package can depend on another Zed package, which can depend on
more packages. `zed install` must resolve and install that complete transitive
graph without downloading shared dependencies repeatedly, deadlocking on
cycles, producing nondeterministic lockfiles, or copying immutable package
trees into every project.

Implementation tracker: [DEN-1505](https://linear.app/denman/issue/DEN-1505/zed-cli-recursively-install-package-dependency-graphs-with-a-five).

## Goals

The recursive installer must:

- expand dependencies declared by every selected package;
- choose one compatible version per `org/name`;
- terminate cycles and deduplicate diamond graphs;
- report conflicts in stable graph order;
- acquire independent artifacts concurrently, with five workers by default;
- coordinate threads and independent CLI processes without polling;
- download and extract each content hash once per shared Zed home;
- preserve immutable, content-addressed storage and symlink-first project
  materialization;
- keep lockfile writes, hooks, adapters, references, and rollback inside one
  deterministic project transaction.

## Two-phase architecture

Recursive installation separates immutable acquisition from mutable project
work.

### Phase 1: deterministic graph discovery and prefetch

Registry metadata resolution stays on the coordinator thread. Starting with the
consumer's direct requirements, the resolver repeatedly selects a package,
reads that package's manifest dependencies, and adds unseen requirements to the
pending graph.

The resolver maintains one selected version for each package coordinate. A
requirement that is incompatible with the already-selected version fails closed
with a deterministic diagnostic. A package already visited is not expanded
again, so cycles terminate and diamond dependencies converge on one node.

Only independent artifact work is submitted to workers. That work includes
cache lookup, download, digest verification, atomic cache publication, and
content-addressed-store extraction.

### Phase 2: transactional project materialization

After the complete graph is selected and its immutable artifacts are available,
the existing install transaction performs:

- frozen-lock verification or deterministic lockfile generation;
- pre/post hooks and native adapter wiring;
- project reference updates;
- rollback bookkeeping;
- `zed_modules/` materialization.

Workers do not independently rewrite `.zpkg.lock`, mutate project directories,
or run project hooks. Completion order therefore cannot leak into lockfile or
error order.

## Bounded worker queue

The current registry and store interfaces are synchronous. The implementation
therefore uses a small blocking worker pool rather than adding a Tokio runtime
around blocking I/O.

The default is five concurrent artifact tasks:

```text
ZED_PKG_INSTALL_CONCURRENCY=5 zed install
```

The supported range is 1 through 32. Zero and invalid values use the default;
oversized values are bounded according to the CLI configuration contract.

Idle workers block on a condition variable. They do not wake periodically to
check the queue. Each task carries a stable sequence number so a faster later
download cannot reorder user-facing failures ahead of an earlier graph task.

A panic at the worker boundary is caught and returned as a sequenced task
failure. The coordinator receives one terminal result for every dispatched
task and cannot wait forever because a worker unwound after popping work.

Conceptually:

```text
resolve direct requirements in stable order
while unresolved package metadata remains:
    select one compatible version per org/name
    enqueue unseen transitive requirements
    assign immutable artifact work a stable sequence number

run at most five artifact tasks at once
collect every result by sequence number
if any task failed: abort before project commit
materialize the selected graph through the existing transaction
```

## Per-artifact cross-process lock

Every artifact SHA-256 has one rendezvous file:

```text
$ZED_PKG_HOME/locks/artifact-<sha256>.lock
```

The lock is descriptor-backed and acquired with the operating system's blocking
exclusive-lock primitive through `fs2`. A waiter sleeps in the kernel until the
owner drops the descriptor or exits. There is no retry timer, exponential
backoff, jitter, stale PID file, lockfile deletion protocol, or filesystem
polling loop.

The critical section is deliberately keyed by content hash rather than package
name or whole install. Two processes may download different dependencies at
the same time, while two requests for the same bytes converge on one owner.

## Unified artifact acquisition

Recursive prefetch, the transactional installer, frozen installs, build
preparation, `zed add`, and `zed remove` use one acquisition function. No older
direct-download path is allowed to bypass the per-hash lock.

For a missing artifact:

1. Check the immutable store.
2. Acquire `artifact-<sha256>.lock`.
3. Re-check the store after waking.
4. Download into a temporary file inside the cache directory.
5. Verify SHA-256 and declared artifact metadata.
6. Atomically rename the verified file to the final cache path.
7. Extract into a temporary directory.
8. Atomically rename the complete content-addressed store entry into place.
9. Release the descriptor-backed lock.

The post-lock store re-check is essential. A waiting process usually wakes
because another process just completed the artifact; it should reuse that entry
instead of downloading again.

Temporary files live on the same filesystem as their final paths so publication
can use atomic rename. A failed or interrupted download never becomes the final
cache entry. A corrupt cache artifact is removed and replaced while the hash
lock is held.

## Symlink-first materialization

The global store contains the one extracted immutable package tree:

```text
~/.zed-pkg/store/v1/<shard>/<sha256>/pkg/
```

A consumer project points at it:

```text
project/zed_modules/<org>/<name>
    -> ~/.zed-pkg/store/v1/<shard>/<sha256>/pkg/
```

A diamond graph therefore stores its shared leaf once. Every consumer and every
path that requires the leaf resolves to the same immutable target.

Copy mode remains explicit for Docker/OCI images and is the non-Unix fallback.
Concurrency and process safety are not reasons to silently copy package trees.

## Determinism and conflict behavior

Graph traversal order, requirement provenance, task sequence, and final package
ordering remain stable across runs. The result must not depend on which network
request finishes first.

When two transitive paths require incompatible versions of one package, the
installer reports the first deterministic graph conflict and does not commit a
partial project tree. It must identify the selected coordinate and the
incompatible requirement paths rather than silently selecting whichever worker
completed first.

Frozen installation resolves from the exact lock graph and never rewrites the
input lockfile. A manifestless frozen replay may materialize the complete graph
from `.zpkg.lock` without synthesizing `.zpkg.toml` when the user selected the
supported no-manifest path.

## Failure and recovery rules

- A failed artifact task aborts project commit.
- Worker panics become ordinary sequenced errors.
- Cache publication occurs only after full digest verification.
- Store publication occurs only after complete extraction.
- Temporary downloads and extraction directories are cleaned on failure.
- Waiters re-check the store after acquiring the hash lock.
- Forced process termination releases descriptor ownership automatically.
- Project rollback remains the responsibility of the existing transaction.
- Lock acquisition itself does not implement cancellation through polling;
  process supervisors and CI timeouts own operational cancellation.

## Required regression matrix

### Graph semantics

- diamond dependency resolves and acquires every package once;
- cycle terminates after every package is expanded once;
- incompatible transitive requirements fail in deterministic graph order;
- frozen lock-only install restores the complete graph without a manifest;
- warm replay downloads zero artifact bytes.

### Concurrency

- default worker count is five;
- supported overrides remain within the configured bounds;
- a gated HTTP fixture with twelve cold artifacts observes a peak of exactly
  five live transfers and never exceeds five;
- a worker panic returns a sequenced error and does not deadlock completion.

### Threads and processes

- concurrent prefetches sharing one home download one absent hash once;
- recursive and transactional acquisition paths racing one hash share one
  lock and one download;
- a child process reaches lock acquisition, remains blocked while the parent
  owns the descriptor, and wakes after orderly release;
- a blocked waiter wakes after the owning process is forcibly terminated;
- the same contracts pass with Windows `LockFileEx` semantics.

### Publication and layout

- corrupt partial cache is replaced under the hash lock;
- failed downloads leave no final cache/store entry or staging leak;
- cache and per-hash lock cardinality match the selected graph;
- every default materialization is a symlink into the shared store;
- separate consumers resolve each package to identical store targets;
- copy-mode OCI contracts remain green.

## Certified multi-process scenario

The companion E2E suite publishes a 13-package graph: one root plus twelve
leaves. It launches four separate `zed install` processes against one shared
home.

Every process resolves all 13 packages, but aggregate cold downloads equal 13
rather than 52. Download ownership can be split across processes because locks
are per hash rather than global. All four consumers materialize symlinks to
identical store targets. A fresh frozen replay downloads 13 artifacts into a
new home, and a warm frozen replay downloads zero.

## Pull-request stack

Merge in order:

1. [`zed-pkg/zed-cli#53`](https://github.com/zed-pkg/zed-cli/pull/53) —
   recursive resolver, five-worker condition-variable queue, unified
   acquisition, symlink-first materialization, and core tests.
2. [`zed-pkg/zed-cli#65`](https://github.com/zed-pkg/zed-cli/pull/65) —
   true interprocess artifact-lock regression and Windows shared-home tests.
3. [`zed-pkg/zed-cli#67`](https://github.com/zed-pkg/zed-cli/pull/67) —
   kernel-blocking install/build/store locks and owner-exit recovery coverage.
4. [`zed-pkg-test/zed-pkg-e2e#14`](https://github.com/zed-pkg-test/zed-pkg-e2e/pull/14) —
   immutable external recursive-install certification.

See also [1 — content-addressable storage + symlinks](01-cas-and-symlinks.md),
[2 — store/project bridge under OCI](02-store-project-bridge-oci.md), and
[6 — process-level locking](06-process-locking.md).

## Non-goals

- zed-pkg does not reimplement native package-manager dependency graphs.
- It does not serialize all downloads behind one global artifact mutex.
- It does not add an async runtime solely to wrap synchronous registry APIs.
- It does not let worker completion order determine lockfile order.
- It does not copy package directories by default.
- It does not treat deleting lock files as releasing ownership.
- It does not hide partial success by committing a reduced dependency graph.
