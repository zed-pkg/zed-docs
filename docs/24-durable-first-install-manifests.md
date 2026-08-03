# 24. Durable manifests on first dependency install

**Linear:** DEN-1413  
**Implementation:** `zed-pkg/zed-cli` PR #42

A project should not become permanently dependent on an invisible, in-memory
configuration merely because its first Zed command was convenient. Beginning
with DEN-1413, a normal dependency-bearing first install adopts the project
into Zed explicitly:

```bash
zed install acme/http-kit@^1
```

When no `.zpkg.toml` exists, Zed creates a small deterministic consumer
manifest at the inferred project root, records the requested direct dependency,
then resolves, locks, and materializes the graph through the ordinary installer.
The previous synthetic-manifest behavior remains available, but only when the
caller asks for it:

```bash
zed install acme/http-kit@^1 --do-not-write-new-manifest
```

This document defines the user-visible contract, generated metadata, failure
semantics, compatibility period, and CI expectations.

## Why the default changes

The original manifestless path was useful for experimentation and CI canaries,
but it had a hidden-state problem: `.zpkg.lock` could survive while the direct
requirements that produced it existed only in the invocation history. A lockfile
records a complete resolved graph; it cannot reliably distinguish the packages
the user chose from transitive dependencies. That makes later updates,
reviews, automation, and Nix/OCI exports less truthful than they should be.

Creating `.zpkg.toml` by default gives the project a durable declaration of
intent:

- direct requirements are reviewable in version control;
- `zed install --frozen` has an unambiguous manifest/lock pair;
- humans and agents see the same project state;
- adapters and target selection can be reproduced on another machine;
- release, Nix, SBOM, provenance, and policy tooling have a stable input;
- uninstall/reinstall cycles do not depend on shell history.

The escape hatch remains important for throwaway sandboxes, one-shot tooling,
and explicit lock-only restoration.

## Command behavior

### Missing manifest, one or more package operands

```bash
zed install acme/http-kit@^1 acme/log-kit@=2.0.0
```

Zed:

1. infers the project root using the same conservative native-manifest,
   lockfile, and directory-structure rules used by the old manifestless path;
2. parses all operands and rejects malformed or conflicting requirements before
   writing project state;
3. resolves an omitted requirement to the registry's current immutable version
   requirement;
4. infers the target and ecosystem adapter when possible;
5. writes `.zpkg.toml` with a same-directory atomic no-clobber operation;
6. runs the existing resolver, integrity checks, lockfile writer, store,
   materializer, adapters, build-hook policy, and project transaction;
7. removes the byte-identical generated manifest if installation fails.

A successful invocation leaves both `.zpkg.toml` and `.zpkg.lock`.

### Missing manifest with the explicit ephemeral flag

```bash
zed install acme/http-kit@^1 --do-not-write-new-manifest
```

Zed uses the established in-memory synthetic consumer manifest. It may create
or update `.zpkg.lock` and materialize packages, but it does not create
`.zpkg.toml`.

The flag suppresses only creation of a **missing** manifest. It does not disable:

- requirement and graph validation;
- checksum, signature, or provenance policy;
- frozen-mode drift checks;
- store locking or project transactions;
- package materialization;
- adapter output;
- explicitly permitted build hooks.

### Existing manifest

When `.zpkg.toml` already exists, `--do-not-write-new-manifest` is an
informational no-op. The command remains a normal manifest-backed install.
It must never turn an established managed project into an ephemeral one.

As before, package operands do not silently edit a human-authored existing
manifest:

```bash
zed add acme/http-kit@^1
```

is the explicit mutation command. A narrowly scoped exception handles two
concurrent first installs: if the first process created a recognized generated
consumer manifest while the second waited on the project lock, the second may
merge distinct direct dependencies into that same generated manifest. A
conflicting requirement fails and directs the caller to `zed add`.

### Frozen lock-only restoration

A lockfile without a manifest does not contain enough information to reconstruct
a truthful direct-dependency set. Therefore lock-only restoration must remain
explicit:

```bash
zed install --frozen --do-not-write-new-manifest
```

Without the flag, Zed fails with an actionable explanation instead of inventing
a manifest from all locked transitive packages.

## Canonical flag and compatibility aliases

The canonical spelling is:

```text
--do-not-write-new-manifest
```

The canonical environment variable is:

```text
ZED_PKG_DO_NOT_WRITE_NEW_MANIFEST=1
```

During the compatibility window, these continue to work and emit deprecation
guidance:

```text
--allow-no-manifest
--skip-manifest
ZED_PKG_ALLOW_NO_MANIFEST=1
```

Runtime parsing, terminal help, Bash completion, Zsh completion, and the
embedded flags-2-env contract must expose the same canonical spelling and
aliases. An explicit CLI option retains precedence over inherited environment
configuration.

## Generated manifest shape

Exact serialization follows the current manifest schema and standard writer.
A representative first-install manifest is:

```toml
[package]
org = "zed-local"
name = "my-project"
version = "0.0.0"
description = "Local Zed dependency manifest; edit package metadata before publishing"
keywords = ["zed-generated-consumer"]

[package.repository]
vcs = "git"
url = "https://localhost/zed-local/my-project"

[dependencies]
"acme/http-kit" = "^1"

[install]
adapter = "node"
target = "node"
```

Properties of generated content:

- the name is a deterministic lowercase slug derived from the selected project
  directory, with `project` as the safe fallback;
- version is the neutral local placeholder `0.0.0`;
- the repository URL is deliberately non-remote and non-authoritative;
- `zed-generated-consumer` marks inferred metadata;
- requested direct requirements are preserved;
- unversioned operands become a deterministic requirement derived from the
  registry response before the file is written;
- inferred target and adapter are recorded only when Zed has a supported value;
- timestamps, random identifiers, absolute machine paths, usernames, and home
  directories are forbidden.

## Non-publishable by default

A generated consumer manifest is immediately useful for installation but is
not a valid package publication declaration. `zed publish` fails closed while
the generated marker and placeholder identity remain.

To turn the project into a publishable package, the maintainer deliberately
edits at least:

- `package.org` and `package.name`;
- the real package version and version scheme;
- repository URL and VCS metadata;
- description and license as appropriate;
- language/ecosystem metadata when applicable;
- the generated marker keyword.

Removing the marker is an explicit acknowledgement that the inferred local
identity has been reviewed. `--skip-vcs-checks` does not bypass this guard.

## Atomicity and concurrency

### First writer

The generated bytes are written to a temporary file in the project directory,
flushed, synchronized, and persisted with no-clobber semantics. If another
writer creates `.zpkg.toml` first, Zed does not overwrite it.

### Failure rollback

Resolution and installation use the existing installer. If that operation
fails after the generated file is written, Zed removes the manifest only when
its current bytes still match the exact generated bytes. If another process or
editor changed the file, rollback refuses to erase that work and reports the
compound failure.

Updates to a recognized concurrently generated manifest retain the prior bytes
and restore them on ordinary installation failure, again only if no external
writer changed the replacement.

### Project-scoped serialization

First installs acquire an exclusive lock under the configured Zed home, keyed
by a hash of the canonical project path. The lock does not pollute the project
and is independent of the global content-store lock. It serializes manifest
creation/merge while allowing the existing installer to retain ownership of
store and project-tree transactions.

A hard process termination can still leave a complete generated manifest whose
lock/materialized state is recovered by the existing transaction journal on the
next invocation. It must never leave a partially written TOML file.

## Project-root inference

The generated manifest is written at the same root selected for installation,
not blindly in the current working directory. Selection order is:

1. nearest ancestor containing `.zpkg.toml`;
2. nearest ancestor containing `.zpkg.lock` or a recognized native manifest;
3. nearest ancestor with a recognized source/project structure;
4. one unambiguous nested project within the bounded scan;
5. the requested directory when no stronger signal exists.

The bounded scan skips hidden directories and generated/vendor trees such as
`node_modules`, `zed_modules`, `target`, `vendor`, `dist`, and `build`.
Ambiguous monorepos do not receive a guessed child manifest.

## Required test matrix

The implementation and release canaries must cover:

| Case | Expected result |
| --- | --- |
| Missing manifest + one package | deterministic manifest, lockfile, and install |
| Missing manifest + several packages | one manifest with every direct dependency |
| Nested invocation | manifest written at inferred project root |
| Canonical flag | install/lock allowed; no manifest written |
| Canonical environment variable | same ephemeral behavior |
| Legacy flags/environment | same behavior plus deprecation diagnostic |
| Existing authored manifest | ordinary install; no hidden mode switch |
| Existing manifest + new flag | informational no-op |
| Missing manifest + failed resolution | no manifest, lockfile, or project install tree |
| Concurrent first installs | one valid manifest, no lost distinct dependencies |
| Conflicting concurrent requirement | fail with no silent winner |
| Read-only project | fail before materialization with a clear path diagnostic |
| Frozen lock-only restore | requires explicit no-new-manifest flag |
| Generated-manifest publish | rejected even with VCS checks skipped |
| Bash/Zsh completion and help | canonical flag plus compatibility aliases |
| Linux and macOS | equivalent manifest and install semantics |

## Relationship to Nix interoperability

DEN-1411's Zed-to-Nix and Nix-to-Zed adapters depend on a trustworthy source of
direct dependency intent. Durable first-install manifests make those exports
reviewable and reproducible. The boundary remains strict:

- this feature does not require Nix for ordinary Zed use;
- it does not translate arbitrary Nix expressions;
- a Nix-imported dependency recorded by Zed follows the same durable-manifest
  rule unless the caller explicitly selects ephemeral installation;
- lockfile adapter provenance remains separate from inferred local package
  identity.

## Rollout

1. Ship the canonical command/help/completion contract and compatibility aliases.
2. Land the durable coordinator and focused Linux/macOS canaries.
3. Keep old aliases for at least one documented compatibility window.
4. Measure legacy alias/environment usage through non-secret aggregate CLI
   diagnostics where policy permits.
5. Remove legacy spellings only in a semver-significant CLI release with a
   migration note.

The durable default is a project-state improvement, not a removal of the
manifestless capability: ephemeral installation remains available whenever the
caller names that intent explicitly.
