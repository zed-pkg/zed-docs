# 19. Polyglot publishing: what is actually verified

Doc 18 sets the release model for multi-registry fan-out. This doc is its
empirical counterpart: what CI now *proves* and what it *found* when run
against real repositories.

Everything below was run against the nine `*-clients` repositories and a
five-language fixture, not reasoned about.

## Goal 2 works: `[targets.repository]` is supported

An earlier draft of this doc claimed `dir = "."` was rejected by `zed pack`.
That was wrong, and it is worth recording why: `pack_all` does guard against a
target carrying its own `.zpkg.toml`, but the guard **exempts the repository
root explicitly**:

```rust
// `dir = "."` is the explicit whole-repository target used alongside
// language slices. Its manifest is the source manifest by definition
// and is replaced with the derived single-target manifest below.
// Nested targets must still never carry a second manifest.
if section.dir != "." && source.join(MANIFEST_FILE).exists() {
```

with a test asserting the repository artifact contains every language slice.
So doc 18's convention is correct as written, and the whole-repository artifact
(goal 2) ships alongside the isolated slices (goal 3) from one manifest.

The parity gate therefore permits `dir = "."` and enforces isolation only
*between language targets*: a language target nested inside another language
target would put the inner language's bytes in the outer artifact, which is the
thing doc 17 exists to prevent.

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
| Go subdirectory modules need a target-specific tag, not the repository-wide tag | fiducia, quaestor, athleto, daedalus — **represented by native `tag_format`; authenticated release execution remains open** |
| The gate ran the ecosystem's tooling against the *source subtree* while claiming to check "exactly what the registry would receive". Packing strips dev files, hoists `LICENSE` into every slice, and substitutes the derived manifest — so it both missed real problems and invented ones: scintilla's dart slice was rejected for a missing LICENSE the artifact contains | all `zed-polyglot.yml`, fixed (`--json` now emits the artifact name; jobs unpack it) |
| `dir = "."` was assigned a native registry by inference. A repo keeping its own workspace `package.json` at the root had that mistaken for the whole-repository target's identity, and the dry-run tried to publish the entire repo to npm — contradicting `Manifest::validate`, which rejects a native route on `dir = "."` | `check-native-parity.py`, fixed |
| Default excludes stripped `CHANGELOG*` while `publish.exclude` can only *add* patterns, so no repository could ship one — and `dart pub publish` fails a package outright without it | `zed-interfaces/src/excludes.rs`, fixed (`include_readme` covers both) |
| Published slices did not declare which language they were for, so nothing could stop a `-java` client installing into a Node project | `zed-interfaces` `package.language`/`ecosystem` + the install guard, fixed |

## A retracted finding: "slices depend on things outside themselves"

An earlier revision of this doc listed three slices as unpublishable because
their manifests reach outside the slice — `athleto-clients` dart
(`path: ../../../athleto-sync`), `scintilla-clients` flutter (`path: ../dart`),
`fiducia-clients` rust (a `git =` dependency crates.io forbids). That finding was
wrong, and the reason is worth keeping.

Every one of those slices is **explicitly opted out of native publishing**:
`publish_to: none` in the two pubspecs, `publish = false` in the Cargo manifest.
The parity gate already honors both and reports them as `ok; publish_to: none` /
`ok; publish = false`. A package that is not going to a native registry may
legitimately depend on a sibling checkout.

athleto's dart manifest in fact already uses the idiomatic shape — hosted
versions under `dependencies`, paths only under `dependency_overrides`, with a
comment noting release automation swaps them. There was nothing to fix.

The failures that *looked* like this class had two other causes, both since
fixed: the dry-runs ran against the source subtree rather than the packed
artifact, and `zed-polyglot.yml` pinned a `zed-interfaces` predating the route
model, so manifests declaring `tag_format`/`forge` failed to deserialize.

The general lesson matches the `dir = "."` correction above: check what the
manifest *declares about its own publishing* before calling a dependency shape a
defect.

## The Go tag gap is now represented explicitly

A Go module in a subdirectory is fetched by the tag `<subdir>/vX.Y.Z`. That is
the module proxy's rule, not a convention. `clients/go` at `0.1.0` therefore
requires the tag `clients/go/v0.1.0`.

`[publish].tag_format` remains the repository-wide Zed release tag. A native
Go route now adds its own tag template:

```toml
[targets.golang.native]
registry = "go-modules"
package = "github.com/fiducia-cloud/fiducia-clients/clients/go"
tag_format = "clients/go/v{version}"
```

Manifest validation requires the directory prefix, and the release plan emits
`clients/go/v0.1.0` alongside the repository tag `v0.1.0`. Creating and
attesting that native tag belongs to the future authenticated release runner.
Target-only *source* mirrors remain a separate `git subtree split` concern.

The same mechanism generalizes beyond Go: Swift, and any ecosystem that
resolves a subdirectory by tag, are served by the same per-target
`tag_format` rather than a repo-wide template.

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

Two further constraints the implemented `[targets.<name>.native]` schema
accommodates:

- **Not every target has a registry.** `shell` and `matlab` are legitimately
  zpkg.tech-only. `registry` must be optional, not defaulted.
- **Not every ecosystem declares a version.** Go, Swift, Packagist, opam,
  Clojars and sbt take the version from the VCS tag alone, so "keep versions in
  sync" cannot be a uniform check. LuaRocks is the inverse: `0.1.0-1` is
  version `0.1.0` plus a rockspec revision, and naive comparison reports drift
  that is not there.

## Status

Verified in CI across nine client repositories and one fixture. Typed native
routes, forge compatibility validation, deterministic release planning, and
per-route tag formats are implemented. Authenticated registry uploads,
release-set attestations, and generic target-only source-mirror automation
remain open.
