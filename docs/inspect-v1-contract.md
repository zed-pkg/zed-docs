# `zed inspect` JSON contract v1

Status: candidate contract under review in `zed-pkg/zed-cli#235`.

Tracking: Linear `DEN-2175`; downstream IDE adoption `DEN-2508` and `DEN-2278`; independent certification `zed-pkg-test/zed-pkg-e2e#119`.

`zed-pkg` is an independent package manager and is unrelated to the Zed editor.

## Command surface

```text
zed inspect --format json --root <absolute-existing-directory>
```

The v1 JSON document declares `schema_version` with major version `1`. Consumers may accept additive `1.x` fields but must fail closed on an unknown major version.

The report also carries a CLI safety declaration:

- `implementation = "zed-pkg"`
- `command = "inspect"`
- `offline = true`
- `mutates_project = false`
- `loads_credentials = false`

IDE adapters must reject a report that does not make those declarations or whose canonical report root does not match the root they requested.

## Startup boundary

Inspection is dispatched before ordinary CLI startup. The inspected path must not cause the CLI to:

- publish terminal/environment state;
- normalize normal command flags into environment configuration;
- construct credential-aware registry configuration;
- load saved credentials;
- contact a registry or download packages;
- run transaction recovery;
- write a manifest, lockfile, materialization directory, adapter output, staging entry, or store entry.

The independent `zed-pkg-test` contract uses an unreachable registry, malformed saved credentials, fake CLI/environment tokens, and a pending `.zpkg-staging` sentinel. It requires a byte-for-byte unchanged project snapshot after inspection.

## Read-only state

The v1 candidate reports local package state derived from reads only, including:

- `.zpkg.toml` manifest state;
- `.zpkg.lock` lock state;
- materialization presence;
- pending transaction state;
- workspace members;
- generated adapter-output presence;
- local content-addressed store presence;
- direct dependency lock consistency.

Parser source excerpts are not copied into diagnostics. This prevents a malformed local file from echoing credential-shaped content into IDE surfaces.

## Diagnostics

The candidate diagnostic vocabulary includes stable codes for:

- missing, invalid, or unreadable manifest;
- missing, invalid, unreadable, or stale lock state;
- missing materialization;
- a locked package not materialized;
- pending transaction recovery.

Locations are file-system paths plus optional line/column metadata. Consumers should treat diagnostic text as display material and the stable code as the machine-facing identity.

## Recommended actions

Inspection never executes a remediation. A recommendation is metadata containing:

- stable action id and display title;
- action kind;
- exact argv vector;
- exact working directory;
- `mutates_project`;
- `requires_network`;
- `executes_package_code`.

IDE integrations must show the executable, argument vector, and working directory and require explicit user confirmation before executing a mutating command. Shell-command strings are not part of the contract; consumers should execute argv directly without a shell.

### Package-code execution risk

`zed install` recommendations are marked `executes_package_code = true`.

The package manifest supports pre-install/post-install lifecycle hooks and build steps, so a future confirmed install can execute package-provided code even though `zed inspect` itself never does. Consumers must preserve and surface this risk bit rather than downgrading it based only on the recommended command name.

`zed init` does not carry that package-code risk.

## Consumer fail-closed rules

A v1 IDE adapter should reject or fall back when any of these conditions hold:

1. unknown schema major;
2. missing or unsafe CLI safety declaration;
3. returned root does not canonicalize to the requested root;
4. recommended executable is not the expected `zed` executable contract;
5. action safety booleans are incomplete;
6. action working directory escapes the inspected project root.

A deterministic local fallback is acceptable for older/unavailable CLIs, but it must remain read-only and confirmation-gated for all mutations.

## Promotion evidence

The contract is not considered promoted solely because a release build succeeds. Required evidence includes:

- `cargo fmt --all --check`;
- product library tests;
- inspect CLI integration tests;
- `cargo clippy --all-targets -- -D warnings`;
- fixed Ubuntu, macOS, and Windows black-box runs against one immutable product SHA;
- no project mutation under poisoned credential/offline/recovery fixtures;
- downstream real-binary interoperability evidence for adopting IDE adapters.

The current workflow in `zed-pkg-test/zed-pkg-e2e` dynamically resolves the self-repaired product branch, binds that SHA to open PR `zed-pkg/zed-cli#235`, certifies the immutable SHA, and fails if the product branch moves during the gate. A separate VS Code interoperability gate resolves both product heads and runs the real CLI through the real adapter.

## Adoption status

- VS Code: v1 adapter is under review in `zed-pkg/zed-vscode#2`; deterministic fallback remains.
- Qt Creator: promoted foundation still uses the earlier placeholder inspect invocation and requires a narrow v1 adapter update.
- Xcode, Eclipse, Visual Studio: promoted controller/model foundations remain separate from shared v1 adoption and vendor-native UI completion.
- Sublime Text and JetBrains: keep their existing safe boundaries while shared v1 adoption is evaluated; do not introduce independent dependency solvers.

This document intentionally leaves marketplace signing, update-channel publication, vendor-native UI shells, and GitHub Projects-v2 tracking as separate completion gates.
