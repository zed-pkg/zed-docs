# Windows clean-room acceptance for `zed develop`

**Status:** implementation and independent certification in review.  
**Linear implementation correction:** [DEN-1616](https://linear.app/denman/issue/DEN-1616/zed-cli-suppress-powershell-profiles-in-zed-develop-command-mode).  
**Linear external certification:** [DEN-1614](https://linear.app/denman/issue/DEN-1614/zed-e2e-add-windows-clean-room-certification-for-zed-develop).  
**Linear contract:** [zed develop Windows clean-room acceptance contract](https://linear.app/denman/document/zed-develop-windows-clean-room-acceptance-contract-316cfafd9902).  
**CLI PR:** [`zed-pkg/zed-cli#100`](https://github.com/zed-pkg/zed-cli/pull/100).  
**E2E PR:** [`zed-pkg/zed-e2e#16`](https://github.com/zed-pkg/zed-e2e/pull/16).

The completed [Unix clean-room contract](zed-develop-clean-room-acceptance.md)
certifies `zed develop` on Ubuntu and macOS. This document adds the Windows
boundary that the original contract intentionally did not claim.

The external Windows suite checks out exact commits, builds the real `zed.exe`
on Windows Server 2022, and imports no implementation test helpers. It consumes
no repository secret and makes no registry, cloud, AI-provider, or user-account
request.

## Reviewed source stack

The initial review stack is:

```text
zed-cli        bd34f4f494e1cc402efa7abb009b83b638480153
zed-interfaces c2e049006453c26ca8ca291783f681fce75cb01f
flags-2-env    2f62e40932a0fcb8b9bf1b4c84473e34fa3c51c7
```

The E2E workflow pins those immutable revisions. A source correction requires a
new explicit CLI pin and complete replay; a moving branch name is not evidence.

## PowerShell command mode

For non-interactive command execution, Zed invokes any explicitly selected
`pwsh`, `pwsh.exe`, `powershell`, or `powershell.exe` as:

```text
<shell> -NoLogo -NoProfile -NonInteractive -Command <command>
```

The switches establish a deterministic automation boundary:

- `-NoLogo` removes banner output;
- `-NoProfile` prevents current-user, all-users, current-host, and all-host
  profiles from executing implicitly;
- `-NonInteractive` prevents prompts and interactive host behavior;
- `-Command` evaluates the caller-supplied command through PowerShell.

The switches apply only when `-c` / `--command` is present. If a user explicitly
requests an interactive PowerShell session, Zed does not inject command-mode
switches and preserves PowerShell's native interactive startup semantics.

### Why this is a security boundary

PowerShell profiles are arbitrary code. They can mutate the environment, load
credentials, contact networks, change aliases, or execute unrelated startup
logic. Before DEN-1616, Zed used `-NoLogo -Command` and therefore did not uphold
the established no-implicit-shell-profile contract on Windows.

The regression test does not merely inspect an argument vector. It:

1. redirects `HOME` and `USERPROFILE` to a temporary directory;
2. asks PowerShell for its actual current-user profile locations;
3. refuses to write if any returned path escapes that temporary home;
4. writes unmistakable fake canaries to the profile files;
5. proves those canaries load during ordinary PowerShell startup; and
6. proves the real `zed.exe` command path neither executes nor emits them.

The same assertion verifies `ZED_DEV`, project-root selection, and child exit-code
propagation.

## cmd.exe command mode

Zed retains the existing command boundary:

```text
cmd.exe /D /S /C <command>
```

`/D` disables AutoRun commands, `/S` applies cmd.exe's command-string parsing
rules, and `/C` executes the command and exits. The independent suite verifies
managed environment delivery, child exit-code propagation, and default
`COMSPEC` selection when `SHELL` is absent.

## Managed environment contract

The Windows clean-room suite verifies:

- byte-identical `zed develop --print-env` and `zed dev --print-env` output;
- global options before the alias;
- nested invocation selecting the owning project root;
- environment-only flags-2-env configuration;
- explicit CLI precedence over inherited Nix and mise modes;
- no implicit `.env`, `.envrc`, production dotenv, or PowerShell-profile
  values;
- no inherited token or credential canaries in stdout, stderr, JSON, or retained
  evidence;
- bounded `--no-install` writes under documented `.zed/dev` state; and
- no `.zpkg.toml` or `.zpkg.lock` creation.

## Python virtual environments

With an explicit interpreter, `--python-venv required`, and a project-local
custom path, the suite proves:

- `VIRTUAL_ENV` identifies the selected directory;
- the executed Python process uses it as `sys.prefix`;
- the executable is resolved from the Windows `Scripts` directory; and
- all created virtual-environment state remains inside the selected project.

## Isolated Windows user state

`--isolated-home` must set both:

```text
HOME=<project>/.zed/dev/home
USERPROFILE=<project>/.zed/dev/home
```

The managed home starts empty. The suite creates fake source-home credentials at
representative Codex, AWS, GCloud, GitHub CLI, npm, and registry locations, then
proves none is copied or emitted.

## Failure contract

The external test covers:

- child exit status through PowerShell and cmd.exe;
- a missing shell executable;
- an invalid enum value;
- conflicting `--print-env` and `--command` modes; and
- interactive invocation with redirected standard streams.

All diagnostics must remain actionable and free of every fake credential canary.

## Workflow policy

The independent workflow must remain:

- on the explicit `windows-2022` runner rather than a moving latest label;
- read-only with `contents: read`;
- free of secret inheritance, OIDC writes, and persisted checkout credentials;
- pinned to immutable Action, CLI, interface, and flags-2-env commits;
- bounded by a 40-minute timeout and concurrency cancellation;
- source-clean without reset or cleanup-based masking;
- limited to runner-temporary evidence retained for no more than seven days; and
- followed by a direct scan of retained evidence for every fake canary.

A static policy suite fails if a future edit weakens any of those rules or
removes the PowerShell, cmd.exe, profile, venv, HOME/USERPROFILE, failure, or
canary assertions.

## Evidence status

The exact CLI and E2E heads are running through GitHub Actions. Before these PRs
are ready to merge, this section and the Linear contract will record:

- final source and E2E heads;
- Windows workflow and repository-wide check run IDs;
- assertion count and managed-environment digest;
- evidence archive digest;
- direct canary-scan result;
- final CLI, E2E, and documentation merge commits.

## Change control

Any future change to Windows shell arguments, profile behavior, default shell
selection, HOME/USERPROFILE handling, virtual-environment activation, or
failure propagation requires:

1. a Linear implementation or contract issue;
2. focused repository-local tests;
3. a new immutable E2E candidate pin;
4. successful native Windows clean-room evidence; and
5. updated documentation explaining the intentional semantic change.

Future trusted activation or explicit profile-loading functionality must obtain
a separate, visible trust decision. It must not weaken command mode's default
no-implicit-profile behavior.
