# IDE integration parity and sandbox certification

Updated: 2026-08-07

## Purpose

This record compares the Zed Package Manager integrations for Sublime Text,
JetBrains IDEs, Visual Studio Code, Qt Creator, Xcode, Eclipse, and Visual
Studio against one shared capability, safety, testing, and distribution bar.

`zed-pkg` is an independent package manager and is unrelated to the Zed editor.

## Parity levels

1. **Core conformance** — multi-root discovery, package-state diagnostics,
   versioned inspection adapter, deterministic fallback, argv execution,
   bounded runtime, credential redaction, unsafe-action rejection, and native
   unit tests.
2. **Native shell** — editor-native diagnostics, package tree/tool window,
   file watchers, settings, exact command and working-directory preview, and
   explicit confirmation before mutation. A tested controller/model layer is a
   prerequisite but does not by itself count as a complete vendor UI shell.
3. **Distribution** — dedicated repository, reproducible package, retained
   artifacts, clean editor-instance tests, signing, and marketplace or update-
   channel publication.

An integration is described as fully user-facing parity only after all three
levels pass.

## Current comparison

| Integration | Repository | Core | Native shell | Distribution | Current position |
| --- | --- | --- | --- | --- | --- |
| Sublime Text | `zed-pkg/zed-sublimetext` | Pass | Pass | Partial | Live integration; multi-root parity, staging recovery, confirmation gates, deterministic `.sublime-package`, and independent Python 3.8/3.14 sandbox certification pass. Clean-profile platform verification and Package Control publication remain. |
| JetBrains / IntelliJ | `zed-pkg/zed-intellij` | Pass | Pass | Partial | Live integration; plugin verifier, package-state tool window/diagnostics, multi-package discovery, output redaction, wrong-launcher detection, staging recovery, and independent sandbox certification pass. Marketplace/clean-instance publication evidence remains. |
| VS Code | `zed-pkg/zed-vscode` | Pass | Pass | Partial | Dedicated repository with native multi-root tree, Problems diagnostics, commands, settings, watchers, safe CLI boundary, cross-platform CI and retained VSIX gate in PR #1. Clean extension-host testing and Marketplace publication remain. |
| Qt Creator | `zed-pkg/zed-qtcreator` | Pass | Controller/model | Partial | Dedicated repository; PR #1 adds multi-root package projection, confirmation-gated action previews and Linux/macOS/Windows CI. `ExtensionSystem::IPlugin`, ProjectExplorer/Issues integration and clean-instance packaging remain. |
| Xcode | `zed-pkg/zed-xcode` | Pass | Companion model | Partial | Dedicated repository; PR #1 adds a Swift multi-root companion workspace model, safe action previews, tests and macOS CI. SwiftUI app, XcodeKit appex, App Group entitlements, signing and clean Xcode tests remain. |
| Eclipse | `zed-pkg/zed-eclipse` | Pass | Marker/model | Partial | Dedicated repository; PR #1 adds multi-root package state, Problems-marker projection, safe quick-fix previews and cross-platform CI. PDE/OSGi shell, resource listeners, UI and p2 application tests remain. |
| Visual Studio | `zed-pkg/zed-visual-studio` | Pass | Tool-window model | Partial | Dedicated repository; PR #1 adds multi-root tool-window state, Error List projections, safe action previews and Windows CI. VS SDK `AsyncPackage`, WPF UI, experimental-instance tests and signed VSIX remain. |

## Shared conformance bar

All certified live integrations and candidates are evaluated for:

- multi-root package discovery;
- manifest, lockfile, materialization, stale-lock, and interrupted-transaction diagnostics;
- CLI availability and wrong-executable detection;
- executable-plus-argument-vector process execution without a shell;
- explicit working directory and bounded timeout/cancellation;
- token, authorization, password, secret, API-key, and GitHub-token redaction;
- versioned `zed inspect --workspace <absolute-root> --json` adaptation;
- deterministic read-only fallback when the inspection endpoint is unavailable;
- rejection of command actions that do not require explicit confirmation;
- native unit/toolchain tests and retained artifacts where packageable.

The final shared JSON inspection endpoint remains tracked in
`zed-pkg/zed-cli#191`; integrations must not become independent dependency
resolvers.

## Merged foundation evidence

| Change | Pull request | Merge commit |
| --- | --- | --- |
| Sublime multi-root and conformance parity | `zed-pkg/zed-sublimetext#3` | `c3153867dff0b560946c9b7e287ed50366b87e6c` |
| JetBrains safety and staging parity | `zed-pkg/zed-intellij#2` | `7282fdbc7046bb2ba58369b59ef627211033e145` |
| Shared parity contract and five candidate implementations | `zed-pkg/.github#25` | `051c897578657f8993924f4ecd2a176f4448d404` |
| Independent seven-editor sandbox matrix | `zed-pkg-test/zed-pkg-e2e#105` | `ee7d220807008e9222bbe80e2efbf3c39b21dfc7` |

## Dedicated repository review heads

| Integration | Pull request | Exact head | Product CI |
| --- | --- | --- | --- |
| VS Code | `zed-pkg/zed-vscode#1` | `9754f10e44235828547cbeeb05e43c5786673af9` | run `31209922091` passed |
| Qt Creator | `zed-pkg/zed-qtcreator#1` | `0372ccd4100e369d9e4593d3df66d8b2b507886a` | run `31209486089` passed |
| Xcode | `zed-pkg/zed-xcode#1` | `654f96f9c1d3ee80afc3034a883cb9083caefe00` | run `31209562414` passed |
| Eclipse | `zed-pkg/zed-eclipse#1` | `e65e8086f328a20adf3f941e1be988d0d757dc0f` | run `31209660197` passed |
| Visual Studio | `zed-pkg/zed-visual-studio#1` | `a3520ea8aa19ecbdaa4c71e77d01af6407a7e05f` | run `31209772575` passed |

All five product PRs are intentionally draft review candidates. Independent
certification is in `zed-pkg-test/zed-pkg-e2e#112`, pinned to those exact full
commit identities rather than mutable branches.

## Independent dedicated-repository sandbox

The durable sandbox workflow now checks the five dedicated repositories instead
of the earlier shared `.github/blueprints`. Test candidate
`6dba5b87a9ec17d223255369a3bc9cd0ebe44d3b` in
`zed-pkg-test/zed-pkg-e2e#112` was certified by focused run `31210197200`.

**Run `31210197200` passed every job:**

- policy/machine-readable parity matrix validation;
- Sublime Python 3.8 and 3.14 tests, isolated fixtures, package builds, and
  artifact uploads;
- JetBrains `check buildPlugin verifyPlugin` and artifact upload;
- VS Code unit/repository-contract tests, VSIX packaging, and artifact upload;
- Eclipse Java 21 Maven/JUnit;
- Xcode Swift tests and release build on macOS;
- Visual Studio .NET 8 tests on Windows;
- Qt Creator CMake/CTest on Linux, macOS, and Windows.

The workflow has read-only repository permissions and uses immutable full commit
identities for the product candidates. It does not mutate product branches,
package registries, IDE state, or credentials.

The same evidence is recorded in the Linear `github.com/zed-pkg-test` project
as **IDE dedicated-repository certification — 2026-08-07**, keeping production
planning and independent certification ownership explicit.

## Remaining promotion work

The durable promotion backlog is `zed-pkg/.github#28`. Repository creation is
complete. Remaining work is:

1. complete each remaining editor-native UI shell;
2. run clean editor-instance GUI tests using the editor vendor's supported test
   harness or experimental instance;
3. sign and publish VSIX, p2, app/appex, Qt Creator plugin, Package Control and
   Marketplace artifacts;
4. adopt the final `zed inspect` contract from `zed-cli#191`;
5. add repositories, PRs, and distribution work to `zed-pkg-project` when the
   available GitHub integration exposes Projects v2 mutation access.

Linear issue `DEN-2508` remains In Progress until distribution parity is
complete.
