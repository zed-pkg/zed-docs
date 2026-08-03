# 25. Complete one-version constraint solving

**Issue:** the recursive installer intentionally selects one version per
`org/name`. A first-match traversal is not sufficient for that policy: a broad
requirement can select a newer version before a later, narrower but compatible
requirement is discovered.

Implementation tracker: [DEN-1553](https://linear.app/denman/issue/DEN-1553/zed-cli-solve-overlapping-compatible-transitive-constraints-before).

## The false-conflict case

Consider this graph:

```text
root
├── left@1.0.0  -> shared ^1
└── right@1.0.0 -> shared <=1.5.0

published shared versions: 1.5.0, 1.9.0
```

If `left` is expanded first, a greedy resolver selects `shared@1.9.0`. Rejecting
`right` afterward would be incorrect: `shared@1.5.0` satisfies both paths.
Dependency declaration order and network completion order must not decide
whether the graph is considered solvable.

## Why simple range intersection is not enough

The requirements contributed by a package are version-dependent and live in
the selected artifact's `.zpkg.toml`. Downgrading one package can therefore:

- remove dependencies contributed only by the old version;
- add different dependencies from the replacement version;
- tighten or loosen constraints on other packages;
- require a later decision to be reconsidered.

A monotonic “keep every constraint ever observed” algorithm is incorrect
because it retains stale constraints from versions no longer selected. A loop
that repeatedly chooses the current maximum can also oscillate. The solver must
model candidate decisions and backtrack deterministically when a branch cannot
produce one complete graph.

## Required solver boundary

The graph solver owns:

- root and transitive requirement provenance;
- one candidate domain per package coordinate;
- deterministic candidate ordering;
- candidate manifest loading;
- addition and removal of version-specific dependency constraints;
- cycle handling and memoized failed states;
- a complete selected graph or a deterministic unsatisfiable explanation.

The artifact layer continues to own:

- the five-worker bounded queue;
- per-SHA descriptor-backed operating-system locks;
- verified temporary downloads and atomic cache publication;
- temporary extraction and atomic content-addressed-store publication.

The project transaction continues to own lockfile publication, adapters, hooks,
references, rollback, and symlink/copy materialization.

## Deterministic search model

A suitable implementation is a depth-first constraint solver with stable
ordering:

1. Seed the active requirements from the consumer and workspace members.
2. Choose the unresolved package with the smallest remaining candidate domain;
   break ties lexically by `org/name`.
3. Order non-yanked candidates newest first using the package's version scheme.
4. For each candidate:
   - acquire or reuse its verified manifest;
   - push the candidate decision;
   - add dependency requirements with full provenance;
   - remove those contributions automatically when backtracking;
   - propagate domains until either every selected version satisfies every
     active requirement or one domain becomes empty.
5. Memoize failed canonical states so cycles and repeated subgraphs terminate.
6. Return the first solution under the stable ordering, or an unsatisfiable
   explanation assembled from the empty domain's active requirement paths.

Candidate artifact acquisition may be cached across branches. Downloading a
candidate that is later rejected is acceptable because the content-addressed
store is immutable and reusable; project materialization still contains only
the final selected graph.

## One resolver, two install modes

Normal install and recursive prefetch must consume the same solved graph. They
must not independently re-run two greedy traversals.

A non-frozen flow should:

1. solve the graph once;
2. acquire the selected artifacts through the bounded queue;
3. hand the exact selected graph to the existing project transaction;
4. write that graph to `.zpkg.lock` during transaction commit.

A frozen flow should parse and verify the existing lock graph, acquire those
exact hashes, and skip solving entirely.

## Required diagnostics

When no solution exists, report:

- the package coordinate whose candidate domain became empty;
- every active requirement, with its originating root or dependency path;
- the candidates considered and why each was excluded;
- whether a candidate was yanked, missing, identity-mismatched, or outside a
  version requirement.

The output order follows stable package and provenance order, never worker
completion order.

## Required regression matrix

1. The `^1` plus `<=1.5.0` reproducer selects `1.5.0` in either declaration
   order.
2. Reversing artifact response timing does not change the graph or diagnostic.
3. Downgrading a candidate removes dependencies contributed only by the old
   version.
4. A two-coordinate graph requiring real backtracking resolves successfully.
5. An unsatisfiable graph reports every conflicting provenance path.
6. Cycles terminate and repeated states are memoized.
7. Diamond graphs still select and acquire one shared package.
8. Frozen replay uses exact lock entries and never invokes the solver.
9. The selected graph is identical between prefetch and transactional install.
10. Final acquisition still peaks at five workers and deduplicates every SHA
    across independent processes.

## Status

The recursive acquisition, locking, publication, and symlink contracts are in
review in `zed-cli` PRs 53, 65, 67, and 68. Complete constraint solving is a
separate correctness follow-up tracked by DEN-1553; until it lands, overlapping
ranges should not be described as fully solved merely because each individual
requirement is valid.
