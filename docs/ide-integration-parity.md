# IDE integration parity and sandbox certification

Updated: 2026-08-07

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

An integration is described as fully user-facing parity only after all three
levels pass.

## Current comparison

| Integration | Repository | Core | Native shell | Distribution | Current position |
| --- | --- | --- | --- | --- | --- |
| Sublime Text | `zed-pkg/zed-sublimetext` | Pass | Pass | Partial | Live integration; multi-root parity, staging recovery, confirmation gates, deterministic `.sublime-package`, and independent Python 3.8/3.14 sandbox certification pass. Clean-profile platform verification and Package Control publication remain. |
| JetBrains / IntelliJ | `zed-pkg/zed-intellij` | Pass | Pass | Partial | Live integration; plugin verifier, package-state tool window/diagnostics, multi-package discovery, output redaction, wrong-launcher detection, staging recovery, and independent sandbox certification pass. Marketplace/clean-instance publication evidence remains. |
| VS Code | `zed-pkg/zed-vscode` | Pass | Pass | Partial | Dedicated repository with multi-root tree, Problems diagnostics, commands, settings, watchers, confirmation gates, clean Extension Development Host coverage, a committed npm lock, immutable Actions, fixed runner images, and retained VSIX packaging. Marketplace signing/publication and the final shared inspect schema remain. |
| Qt Creator | `zed-pkg/zed-qtcreator` | Pass | Controller/model | Partial | Dedicated repository; PR #1 adds multi-root package projection, confirmation-gated action previews, fixed runners and immutable checkout inputs. `ExtensionSystem::IPlugin`, ProjectExplorer/Issues integration and clean-instance packaging remain. |
| Xcode | `zed-pkg/zed-xcode` | Pass | Companion model | Partial | Dedicated repository; PR #1 adds a Swift multi-root companion workspace model, safe action previews, fixed macOS runner and immutable checkout. SwiftUI app, XcodeKit appex, App Group entitlements, signing and clean Xcode tests remain. |
| Eclipse | `zed-pkg/zed-eclipse` | Pass | Marker/model | Partial | Dedicated repository; PR #1 adds multi-root package state, Problems-marker projection, safe quick-fix previews, fixed runners and immutable Actions. PDE/OSGi shell, resource listeners, UI and p2 application tests remain. |
| Visual Studio | `zed-pkg/zed-visual-studio` | Pass | Tool-window model | Partial | Dedicated repository; PR #1 adds multi-root tool-window state, Error List projections, safe action previews, fixed Windows runner and immutable Actions. VS SDK `AsyncPackage`, WPF UI, experimental-instance tests and signed VSIX remain. |

## Shared conformance and supply-chain bar

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
- native unit/toolchain tests and retained artifacts where packageable;
- fixed hosted-runner images instead of `*-latest` labels for review gates;
- third-party Actions pinned to full commit identities;
- checkout credentials not persisted for read-only certification;
- committed dependency locks for package-build toolchains where applicable.

The final shared JSON inspection endpoint remains tracked in
`zed-pkg/zed-cli#191`; integrations must not become independent dependency
resolvers.

## Merged foundation evidence

| Change | Pull request | Merge commit |
| --- | --- | --- |
| Sublime multi-root and conformance parity | `zed-pkg/zed-sublimetext#3` | `c3153867dff0b560946c9b7e287ed50366b87e6c` |
| JetBrains safety and staging parity | `zed-pkg/zed-intellij#2` | `7282fdbc7046bb2ba58369b59ef627211033e145` |
| Shared parity contract and five candidate implementations | `zed-pkg/.github#25` | `051c897578657f8993924f4ecd2a176f4448d404` |
| Initial independent seven-editor sandbox matrix | `zed-pkg-test/zed-pkg-e2e#105` | `ee7d220807008e9222bbe80e2efbf3c39b21dfc7` |

## Dedicated repository review heads

| Integration | Pull request | Current exact head | Current evidence |
| --- | --- | --- | --- |
| VS Code | `zed-pkg/zed-vscode#1` | `96c0789a446891d5cb502be799ed121930aee9d1` | predecessor `5171a9a7b38f8aca1a403894499bda8b79d46eb1` passed run `31238551321`, including clean Extension Development Host and VSIX; final lock-only/read-only head run `31238760837` queued at last observation |
| Qt Creator | `zed-pkg/zed-qtcreator#1` | `1780ee33eaf8ba3b5fffa8ce3ccc8fb2efea308e` | immutable/fixed-runner CI `31238398562` passed |
| Xcode | `zed-pkg/zed-xcode#1` | `e7e7030fa44496c14de7da684772c4113cfdd92f` | immutable/fixed-runner CI `31238409074` passed |
| Eclipse | `zed-pkg/zed-eclipse#1` | `58394c48e704f041bb29be7e9079806e25abd41e` | immutable/fixed-runner CI `31238416607` passed |
| Visual Studio | `zed-pkg/zed-visual-studio#1` | `4c4141d3ed595fd2b6a331a5a8756f1ed0cde455` | immutable/fixed-runner CI `31238426829` passed |

All five product PRs remain draft review candidates. There are no unresolved
inline review threads on these PRs.

## VS Code clean-host and locked-build evidence

The VS Code review branch now includes a clean Extension Development Host test
using a disposable workspace, disposable user-data directory and a deliberately
missing CLI path. The test proves activation, command registration, fallback
Problems diagnostics, and no creation of `.zpkg.lock` or `zed_modules`.

Run `31238551321` passed the Linux/macOS/Windows unit/repository checks, the
clean Extension Host job and VSIX packaging. Its lock candidate was subsequently
committed as `package-lock.json` with lockfile version 3 and exact root tool
identities `@vscode/test-electron@2.5.2` and `@vscode/vsce@3.9.2`. The one-shot
write-enabled lock materializer was then removed; permanent CI now uses `npm ci`
and read-only repository permissions.

## Independent dedicated-repository sandbox

The durable sandbox in `zed-pkg-test/zed-pkg-e2e#112` now:

- checks the five dedicated repositories instead of the old shared blueprints;
- pins every third-party Action by full commit SHA;
- uses fixed hosted runner images;
- disables persisted checkout credentials;
- uses `npm ci` for the VS Code locked toolchain;
- runs the clean VS Code Extension Development Host test before VSIX packaging;
- contains a policy ratchet that rejects mutable Action tags, `*-latest`
  runners, `npm install`/`npx --yes`, persisted credentials, and product pins
  that drift from the machine-readable parity matrix.

The current test-org review head is expected to certify these product heads:

- VS Code `96c0789a446891d5cb502be799ed121930aee9d1`;
- Qt Creator `1780ee33eaf8ba3b5fffa8ce3ccc8fb2efea308e`;
- Xcode `e7e7030fa44496c14de7da684772c4113cfdd92f`;
- Eclipse `58394c48e704f041bb29be7e9079806e25abd41e`;
- Visual Studio `4c4141d3ed595fd2b6a331a5a8756f1ed0cde455`.

The previous dedicated-repository sandbox run `31210197200` passed all jobs on
the earlier exact heads. A fresh exact-head run is required after this
supply-chain hardening before the new review heads are called certified.

The same evidence is recorded in the Linear `github.com/zed-pkg-test` project
as **IDE dedicated-repository certification — 2026-08-07**, keeping production
planning and independent certification ownership explicit.

## Remaining promotion work

The durable promotion backlog is `zed-pkg/.github#28`. Repository creation is
complete. Remaining work is:

1. complete each remaining editor-native UI shell;
2. run clean editor-instance GUI tests for Sublime, JetBrains, Qt Creator,
   Xcode, Eclipse and Visual Studio using vendor-supported test harnesses or
   experimental instances;
3. sign and publish VSIX, p2, app/appex, Qt Creator plugin, Package Control and
   Marketplace artifacts;
4. adopt the final `zed inspect` contract from `zed-cli#191`;
5. add repositories, PRs, and distribution work to `zed-pkg-project` when the
   available GitHub integration exposes Projects v2 mutation access.

Linear issue `DEN-2508` remains In Progress until distribution parity is
complete.
