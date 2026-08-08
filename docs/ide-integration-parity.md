# IDE integration parity and sandbox certification

Updated: 2026-08-08

## Purpose

This record compares the Zed Package Manager integrations for Sublime Text,
JetBrains IDEs, Visual Studio Code, Qt Creator, Xcode, Eclipse, and Visual
Studio against one shared capability, safety, testing, supply-chain, and
distribution bar.

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

## Current comparison

| Integration | Repository | Core | Native shell | Distribution | Current position |
| --- | --- | --- | --- | --- | --- |
| Sublime Text | `zed-pkg/zed-sublimetext` | Pass | Pass | Partial | Live; clean-profile platform evidence and Package Control publication remain. |
| JetBrains / IntelliJ | `zed-pkg/zed-intellij` | Pass | Pass | Partial | Live; Marketplace/clean-instance evidence remains. |
| VS Code | `zed-pkg/zed-vscode` | Pass | Pass | Partial | Dedicated repo, committed npm lock, immutable/fixed CI, clean Extension Development Host, and retained VSIX are implemented. Final shared inspect schema plus Marketplace signing/publication remain. |
| Qt Creator | `zed-pkg/zed-qtcreator` | Pass | Controller/model | Partial | Dedicated repo with immutable/fixed CI. `ExtensionSystem::IPlugin`, ProjectExplorer/Issues integration, clean instance and signed package remain. |
| Xcode | `zed-pkg/zed-xcode` | Pass | Companion model | Partial | Dedicated repo with immutable/fixed macOS CI. SwiftUI app, XcodeKit appex, App Group/signing and clean Xcode tests remain. |
| Eclipse | `zed-pkg/zed-eclipse` | Pass | Marker/model | Partial | Dedicated repo with immutable/fixed CI. PDE/OSGi UI/application tests and p2 publication remain. |
| Visual Studio | `zed-pkg/zed-visual-studio` | Pass | Tool-window model | Partial | Dedicated repo with immutable Windows CI. VS SDK/WPF shell, experimental-instance tests and signed VSIX remain. |

## Shared conformance and supply-chain bar

Certified review candidates require multi-root/state diagnostics, argv-only
execution, bounded runtime, redaction, explicit mutation confirmation,
deterministic fallback, native tests, fixed hosted-runner images, full-SHA
third-party Actions, unpersisted checkout credentials, and committed dependency
locks for package-build toolchains where applicable.

The shared JSON inspection endpoint is now implemented for review in
`zed-pkg/zed-cli#235`, tracked by `zed-pkg/zed-cli#191`; IDE integrations must
not become independent dependency resolvers. Its independent black-box contract
is `zed-pkg-test/zed-pkg-e2e#119`.

## Merged foundation evidence

| Change | Pull request | Merge commit |
| --- | --- | --- |
| Sublime multi-root and conformance parity | `zed-pkg/zed-sublimetext#3` | `c3153867dff0b560946c9b7e287ed50366b87e6c` |
| JetBrains safety and staging parity | `zed-pkg/zed-intellij#2` | `7282fdbc7046bb2ba58369b59ef627211033e145` |
| Shared parity contract and candidate implementations | `zed-pkg/.github#25` | `051c897578657f8993924f4ecd2a176f4448d404` |
| Initial independent seven-editor sandbox | `zed-pkg-test/zed-pkg-e2e#105` | `ee7d220807008e9222bbe80e2efbf3c39b21dfc7` |

## Promoted dedicated-repository foundations

Each promoted head had repository-local exact-head green CI, no unresolved
inline review threads, and an independent `zed-pkg-test` sandbox pass against
the same immutable product SHA before the expected-head guarded merge.

| Integration | Pull request | Certified exact head | Merge commit | Exact-head green evidence |
| --- | --- | --- | --- | --- |
| VS Code | `zed-pkg/zed-vscode#1` | `c1d3318a2cf8cd717f9b0af67117ddbac3e28d2e` | `77cf82d52413b07b7af66328f7e11b4dc3bdd0ba` | run `31239054128` passed Linux/macOS/Windows tests, locked `npm ci`, clean Extension Development Host, VSIX packaging and artifact upload |
| Qt Creator | `zed-pkg/zed-qtcreator#1` | `8e7958c7c85a305b59accfe82c28e7ff42bfecba` | `0308593bdb0aaf09559fa1442f855202ec0dda75` | run `31239125093` passed fixed Linux/macOS/Windows CMake/CTest CI |
| Xcode | `zed-pkg/zed-xcode#1` | `9b9304fee2ff236fc59407ddd97cc2c01225a69d` | `a5563588d8bdc63f3b99b133e8316b880597c2b2` | run `31239129386` passed fixed macOS 15 Swift CI |
| Eclipse | `zed-pkg/zed-eclipse#1` | `246b46048cced1d1072afd3bedc9849efe3a8a8f` | `9bff83bb0922f86535ebad78dc16c6730e9e8b6b` | run `31239136280` passed immutable/fixed-runner Maven/JUnit CI |
| Visual Studio | `zed-pkg/zed-visual-studio#1` | `d85a9c827140426a8d9401b5c269ff36f591a299` | `533a10364e181e587992b35a824c4e2eb9dfb4ac` | run `31239141244` passed fixed Windows 2025 .NET CI |

These merges promote hardened package-state controller/model foundations and CI;
they do not claim the remaining vendor-native UI shell or distribution work is
complete.

## VS Code clean-host and locked-build gate

The VS Code implementation launches a disposable Extension Development Host with
a disposable user-data directory and fixture workspace, forces an unavailable
CLI, verifies activation and registered commands, verifies fallback Problems
diagnostics, and proves refresh creates neither `.zpkg.lock` nor `zed_modules`.

`package-lock.json` is committed with exact root identities
`@vscode/test-electron@2.5.2` and `@vscode/vsce@3.9.2`; permanent CI uses
`npm ci`. The one-shot write-enabled lock materializer was removed after the
lock was committed.

Exact-head Actions artifact `9016582558` has archive digest
`sha256:4d84f32c1bd7a267cff9fdad966a7a84cdfa4bdc80731143e0b175e4b18fc5a0`.
The contained `zed-package-insights.vsix` has SHA-256
`551d2376388c073d2f34c89f9a3ed0d83d786f4695fb5e9f33084f3e5e9f08ca` and
packages extension version `0.1.0` for publisher `zed-pkg` under the MIT license.

## Independent final-head sandbox

Draft `zed-pkg-test/zed-pkg-e2e#112` has head
`0c55eb576a18ca5946b7ffe5d28b9167a37a3f67` and pins exactly the five certified
product heads promoted above.

The focused final-product sandbox run `31239171374` passed those exact five
product SHAs. The subsequent test-harness change did not repin product code; it
made the broader fleet audit explicitly review three package namespaces:
`zed-pkg-test`, legacy `zedtest`, and upstream `zed-pkg`. The upstream namespace
is required by the two intentional source mirrors `zed-interfaces` and `zed-lib`;
their manifests are preserved rather than rewritten for the test organization.

The current-head organization inventory run `31240156197` is green, including
all Python contract tests, checked-in core lifecycle coverage, the complete live
public fleet probe, and all required package-manifest checks. The refreshed
current-head IDE sandbox run `31240156191` has a green policy job and completed
green Visual Studio, Qt Creator/Windows, and Sublime/Python 3.14 jobs; remaining
hosted jobs are still queued, so this document does not call that refreshed run
terminal-successful yet. The earlier focused run remains the independent
immutable-product evidence used for the guarded production merges.

The sandbox pins every third-party Action by full commit SHA, uses fixed runner
images, disables persisted checkout credentials, uses `npm ci`, runs the VS Code
clean host, and contains a policy ratchet rejecting mutable Actions/runners/tool
resolution and candidate/matrix drift.

## Shared inspect protocol work

`zed-pkg/zed-cli#235` adds `zed inspect --format json --root <absolute-path>` as
an early-dispatched read-only command. The implementation is designed to exit
before normal CLI startup can publish terminal environment state, construct
credential-aware configuration, contact the registry, or run transaction
recovery. It reports local manifest/lock/materialization/workspace/adapter/store
state plus stable diagnostics and confirmation-gated argv/cwd remediation
metadata under schema `1.0`.

`zed-pkg-test/zed-pkg-e2e#119` independently pins the exact product head and
builds/runs it on fixed Linux, macOS, and Windows images with an unreachable
registry, malformed credential state, fake token inputs, and a pending
transaction sentinel. Promotion of the inspect contract remains gated on both
product CI and this independent black-box lane.

## Remaining promotion work

1. finish Qt Creator, Xcode, Eclipse and Visual Studio vendor-native UI shells;
2. run clean editor-instance/distribution tests for Sublime and JetBrains and the
   remaining native editors;
3. sign and publish VSIX, p2, app/appex, Qt Creator plugin, Package Control and
   Marketplace artifacts;
4. merge and adopt the final shared `zed inspect` v1 schema after product and
   `zed-pkg-test` certification;
5. add/update delivery items in `zed-pkg-project` when Projects v2 mutation is
   available through the connected GitHub capability.

Linear `DEN-2508` remains In Progress until distribution parity is complete.
