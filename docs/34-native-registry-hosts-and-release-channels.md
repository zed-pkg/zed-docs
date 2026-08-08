# 34. Native registry hosts and release channels

[Doc 18](18-multi-registry-release-fanout.md) establishes that native
registries are distribution mirrors of one reviewed source commit. It says
where a target goes. This document says how zed gets it there, and how a
release candidate is expressed once it arrives.

## Two different things are called "registry"

This document is about **native ecosystem registries**: npm, PyPI, Maven Central,
Hex, and the rest. Zed mirrors its packages *out* to them and resolves versions
*from* them.

That is not the same subject as zpkg's own registry protocol — registry
identity, signed checkpoints, the sparse NDJSON index, private dependencies —
which is where a package published *as a Zed package* lives. Both are "a
registry", both have publish and pull paths, and both have credentials, so the
words collide constantly.

The distinction that resolves it:

| | zpkg registry | native host (this document) |
| --- | --- | --- |
| Direction | inbound: where Zed packages live | outbound: where a slice is mirrored to |
| Identity | immutable `registry_id`, alias-independent | a fixed public host and its protocol |
| Coordinate | one canonical normalization | per-ecosystem, and they disagree |
| Credential | `ZPKG_TOKEN_<ALIAS>` | `ZED_<HOST>_TOKEN`, else the ecosystem's own variable |

The third row is the one that bites. A single lowercase-canonicalization rule is
correct for a zpkg coordinate and wrong for native names: CocoaPods is
case-sensitive (`Alamofire` resolves, `alamofire` 404s), PyPI folds `.`/`_`/`-`
per PEP 503, Go escapes uppercase as `!x`, and Maven and Clojars disagree on the
separator. `NativeRegistry::canonical_package` therefore normalizes per host and
must not be replaced by a global rule.

## Registries, not package managers

Zed reaches a native registry over that registry's own HTTP API. It does not
drive `npm publish`, `gem push`, `cabal upload`, or `twine`.

That is a deliberate boundary, and it is not the same boundary as validation.
Package-manager binaries remain the right tool for *validating* a package:
`npm pack` and `cargo package` encode packing rules that should not be
reimplemented, and `zed release preflight` still runs them where a toolchain is
present. They are the wrong tool for *reaching* a registry:

- every publish target would become a toolchain zed must install, pin, and keep
  on `PATH`;
- a polyglot repository's release job would need every toolchain at once;
- zed would be capped at the ecosystems whose CLI happens to be installed. A
  Haskell client cannot be published from a runner with no GHC.

The APIs are stable, documented, and toolchain-free. What zed needs from them is
narrow: list a package's versions, fetch one, and upload an artifact that has
already been built and validated.

## Three axes

`zed-interfaces::native_host` keeps three facts apart because they vary
independently.

**`NativeHost`** — the concrete registry, 29 of them. Includes the ones that
store no artifact: Zig, whose dependencies are URLs pinned by hash; a plain Git
or Mercurial remote on GitHub, GitLab, or Bitbucket, which zed supports
first-class; and Go's proxy, which caches what a VCS tag already published. A
release plan still has to name where those come from.

**`RegistryProtocol`** — the wire protocol, 22 of them. Hosts outnumber
protocols on purpose. Clojars and Maven Central both serve Maven 2. PyPI and
TestPyPI are one protocol at two hosts. Every mirror in `UniversalHost`
(Artifactory, Nexus, GitHub/GitLab/Bitbucket Packages, AWS CodeArtifact,
Cloudsmith, Azure Artifacts) re-serves an existing protocol at a different base
URL. One client implementation therefore serves many hosts, which is why the
axis exists at all.

Mirror compatibility keys on the *artifact format*, not the ingest protocol.
Maven Central's Central Portal upload — a zipped bundle plus a deployment poll —
describes only how Central accepts a release; the artifact is an ordinary Maven
jar that every Maven-hosting mirror serves.

**`ReleaseChannel`** — `stable`, `rc`, `beta`, `alpha`, `nightly`, `snapshot`.

## Why a channel is not a version suffix

Every one of these is a real, mutually incompatible way to ship a candidate:

| Host | A release candidate is… |
| --- | --- |
| npm | a SemVer prerelease **and** a dist-tag that is not `latest` |
| Hackage | a different endpoint (`/packages/candidates/`) |
| Maven | `-SNAPSHOT` in a different **repository** from releases |
| PyPI | PEP 440 (`1.4.0rc1`) — which is not valid SemVer |
| RubyGems | any letter anywhere in the version |
| Conan | part of the package reference, not the version |
| CPAN | a TRIAL release, marked by an underscore |
| crates.io | a SemVer prerelease, and nothing else — there is no channel |

Generalizing from any one of these produces an artifact the next host rejects,
or — worse — silently promotes a candidate to stable and moves every unpinned
consumer. `NativeHost::channel_route` resolves the triple
(host, channel, base version) into the exact version, dist-tag, and endpoint a
publish will use, so no caller re-derives any of it.

A host with no candidate track — CRAN, opam, LuaRocks, Racket, the PowerShell
Gallery, Stackage — **rejects** one. Failing the plan by target name is the
whole point: publishing a candidate as stable is the outcome worth preventing.

`snapshot` is the one mutable channel, and it is available on **three hosts
only**: Maven Central, Clojars, and Packagist. Maven serves snapshots from a
repository built for exactly that, and Packagist re-reads a `dev-*` branch on
every update, so the doc-18 invariant rejecting same-version/different-content
output is relaxed there and nowhere else.

Everywhere else a snapshot is refused rather than approximated. npm, crates.io,
PyPI, RubyGems, NuGet, Hex, pub.dev, and Go's proxy all reject a second upload
of a version outright — immutability is the property their consumers rely on.
Handing back a `1.4.0-SNAPSHOT` that one of them accepts exactly once and then
refuses forever is worse than refusing it up front, because the failure lands
after the release is already half-run.

## Commands

```sh
zed release plan --channel rc --iteration 2 --json
zed release publish --channel rc --iteration 2 --dry-run
zed release versions --target python
```

`--dry-run` prints the exact request each route would send — verb, URL, headers,
body shape — and sends nothing. It is the same construction path a real run
takes, so a dry run that looks right is evidence the real one will be.

One `--channel rc` resolves to four different version strings across a
four-language repository:

| target | host | version | dist-tag |
| --- | --- | --- | --- |
| `nodejs` | npm | `1.4.0-rc.2` | `rc` |
| `python` | PyPI | `1.4.0rc2` | — |
| `java` | Clojars | `1.4.0-RC2` | — |
| `ruby` | RubyGems | `1.4.0.rc.2` | — |

The release set still shares one source version and one Git tag. The channel
changes the destination, not the commit being released.

A route may pin its own default track in the manifest, for a client generated
from an API surface that is not yet stable:

```toml
[targets.nodejs.native]
registry = "npm"
package = "@acme/client"
channel = "beta"
```

An explicit `--channel` overrides it, so the same repository can still cut a
real release.

## Credentials

Credentials are read from environment variables only — the same ones the
ecosystem's own tooling reads, plus a `ZED_*` override that takes priority so a
repository publishing to two npm registries can redirect one without unsetting
the other. Zed does not consult credential helpers or ambient logins: a publish
token must be an explicit input.

A host that publishes by VCS tag has no registry credential and is never asked
for one. Prompting for a token that is never used is how a CI job ends up with
an unnecessary secret in scope.

Credentials never reach a printable string. They are carried as a secret-typed
header value whose only `Display` implementation redacts them, and that covers
LuaRocks and Packagist too — both put the token in the URL rather than a header,
so redaction is positional. Matching on the token *value* would fail open,
because a token is opaque.

## Current implementation boundary

Implemented:

- 29 hosts and 22 protocols, with endpoints, auth schemes, and channel rules;
- channel resolution, including endpoint moves and dist-tags;
- enterprise and forge mirrors keyed on protocol compatibility;
- version listing and download-URL construction for the protocols with a
  machine-readable per-package index;
- single-request uploads, with credential redaction and `--dry-run`.

Multi-request publishes, for the two hosts that need them:

- **pub.dev** — three requests. Ask for a signed upload form, post the archive to
  wherever that form points (object storage, not pub.dev), then fetch the
  finalize URL the upload returns in `Location`. The middle request is
  deliberately unauthenticated: the signed form *is* the authorization, and
  replaying the pub.dev token to a third-party storage origin would hand it to
  whatever host the grant names.
- **Maven Central Portal** — upload a bundle, then poll the deployment. A `201`
  from the upload means "accepted for validation", not "published"; reporting
  success there would tell a release job a version exists that Central may
  reject minutes later.

Not yet, and typed rather than silent:

- **Conan's create-revision-then-upload** returns an error naming the missing
  step. ConanCenter accepts no uploads at all — its recipes arrive by pull
  request — so this only matters through an Artifactory mirror;
- **indexes with no per-package endpoint** — LuaRocks (its manifest is a Lua
  table), ConanCenter, the Julia General registry, and opam return an error
  saying why;
- **native-manifest cross-checks** beyond Hex and the original nine ecosystems.
  A route zed cannot check is *reported* rather than passed quietly —
  `zed release plan` prints an `unchecked` line naming the target and the reason
  — because "nothing was verified" and "everything matched" must not look the
  same to whoever is cutting the release. Two cases are inherent rather than
  unfinished: JVM coordinates in Gradle, sbt, Leiningen, or deps.edn, and BEAM
  coordinates in `mix.exs` or `rebar.config`, all live in executable build code
  with nothing to parse. Gleam is the one BEAM manifest that is data
  (`gleam.toml`), so it is checked;
- **release-set attestations** covering native uploads, which doc 18 already
  lists as follow-up work.
