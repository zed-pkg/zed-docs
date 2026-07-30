# 19. Polyglot publishing: what is actually verified

Doc 18 sets the release model for multi-registry fan-out. This doc is its
empirical counterpart: what CI now *proves*, what it *found*, and which of doc
18's conventions do not survive contact with the implementation.

Everything below was run against the nine `*-clients` repositories and a
five-language fixture, not reasoned about.

## Correction to doc 18: `[targets.repository] dir = "."` does not work

Doc 18's manifest example opens with:

```toml
[targets.repository]
dir = "."
name = "fiducia-clients-repository"
```

`zed pack` rejects it. `pack_all` bails on any target whose source root
contains the repository's own `.zpkg.toml`, and `dir = "."` always does:

```
target `repository` contains its own .zpkg.toml; declare packages only in
the repository-root manifest
```

The failure is not scoped to that target — `zed pack` aborts the whole
fan-out, so **no** target in such a manifest can be packed.
`sonus-auris-clients` shipped exactly this manifest and could not publish
anything at all until it was removed.

So the "complete-repository Zed artifact" in doc 18's release invariants is
not currently obtainable for a polyglot repo *via `[targets]`*. `[targets]` is
a partition of the repo into isolated language roots; a root-level entry both
swallows every sibling slice (defeating doc 17's argument) and trips the
nested-manifest guard. Goal 2 needs its own manifest concept — it is not a
target.

`scripts/check-native-parity.py` now rejects `dir = "."` and any target nested
inside another, so this fails at review time rather than at release time.

## What CI proves now

`zed-cli/.github/workflows/polyglot.yml` packs `tests/fixtures/polyglot` — one
repo, five languages — and asserts:

- exactly five artifacts, one per declared target;
- each slice contains its own language and **no** file extension belonging to
  the other four;
- each slice carries its ecosystem's native manifest at its *root*, so the
  native toolchain recognizes it without configuration;
- no slice carries a `[targets]` block (fan-out cannot recurse);
- dev/test files are stripped and `LICENSE` is hoisted into every slice;
- all five publish to a `file://` registry, and republishing skips
  byte-identical targets rather than erroring.

Then — the part nothing checked before — each slice is handed to the **real**
native tool: `npm pack --dry-run`, `python -m build` + `twine check`,
`cargo package`, `gem build`, `go build`. If npm would reject the tarball, CI
fails.

That is the load-bearing evidence for doc 18's claim that native registries can
mirror the same commit: the slice `zed pack` emits *is already* a publishable
native package.

## What it found

| Finding | Where |
| --- | --- |
| `test_*.py` shipped in every published Python slice — `DEFAULT_EXCLUDES` had `*_test.py` but not unittest's default `test*.py` pattern | `zed-interfaces/src/excludes.rs`, fixed |
| Version drift: `pubspec.yaml` at `1.0.0` against a repo version of `0.1.0` | `daedalus-clients`, fixed |
| Manifest unpackable via `dir = "."` | `sonus-auris-clients`, fixed |
| Go subdirectory modules cannot be published to the module proxy | fiducia, quaestor, athleto, daedalus, zed-clients — **open** |

## The Go blocker is the highest-leverage gap

A Go module in a subdirectory is fetched by the tag `<subdir>/vX.Y.Z`. That is
the module proxy's rule, not a convention. `clients/go` at `0.1.0` therefore
requires the tag `clients/go/v0.1.0`.

`[publish].tag_format` is a single repo-wide template, so zed can only produce
`v0.1.0`. Every polyglot repo here with a Go client in a subdirectory cannot
publish it to the proxy.

This is the same mechanism doc 18 reaches for with `git subtree split` and
`zed/<target>/v<version>` tags — and it generalizes to Swift and any ecosystem
that resolves a subdirectory by tag. Per-target tag formats are the smallest
change that unblocks the most.

## Identity cannot be derived, only declared

Doc 18 is right that native manifests stay authoritative. The audit shows *how
far* the names diverge, which is why no naming rule could ever recover them:

```
target      zed package                            native registry  native package
----------  -------------------------------------  ---------------  ----------------------------------
golang      fiducia/fiducia-clients-golang@0.1.0   Go module proxy  github.com/…/clients/go
java        fiducia/fiducia-clients-java@0.1.0     Maven Central    cloud.fiducia:fiducia-client@0.1.0
nodejs      fiducia/fiducia-clients-nodejs@0.1.0   npm              @fiducia/client@0.1.0
python      fiducia/fiducia-clients-python@0.1.0   PyPI             fiducia-client@0.1.0
…
31 target(s): 29 publishable to a native registry, 2 to zpkg.tech only
```

Two further constraints the proposed `[targets.<name>.native]` schema has to
accommodate:

- **Not every target has a registry.** `shell` and `matlab` are legitimately
  zpkg.tech-only. `registry` must be optional, not defaulted.
- **Not every ecosystem declares a version.** Go, Swift, Packagist, opam,
  Clojars and sbt take the version from the VCS tag alone, so "keep versions in
  sync" cannot be a uniform check. LuaRocks is the inverse: `0.1.0-1` is
  version `0.1.0` plus a rockspec revision, and naive comparison reports drift
  that is not there.

## Status

Verified in CI across nine client repositories and one fixture. The four items
under doc 18's "still follow-up work" remain open; per-target `tag_format`
should come first, because it is the one blocking a shipped ecosystem today.
