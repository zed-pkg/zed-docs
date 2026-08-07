# Terminal and shell execution context

`zed` is a package-management CLI, not the Zed editor. Its terminal behavior must remain safe for both humans and automation: detect execution context, but do not silently change machine-readable output, exit codes, or shell syntax because a terminal happens to be present.

This document records the implementation and certification that landed in [`zed-pkg/zed-cli#219`](https://github.com/zed-pkg/zed-cli/pull/219), with the opt-in flags2env companion in [`ORESoftware/flags-2-env#24`](https://github.com/ORESoftware/flags-2-env/pull/24).

Canonical planning:

- Linear Zed issue: `DEN-2591`
- Linear flags2env issue: `DEN-2581`
- Documentation tracking issue: `zed-pkg/zed-docs#54`
- Documentation delivery PR: `zed-pkg/zed-docs#55`
- Zed merge: `ad6ff369a763f0fbdc3b677894655c73b23e062c`
- flags2env merge: `8a978aef0cc9b12bdd0791d93bbf3a374c517ee2`

## Policy

The CLI may observe terminal and shell context to make explicit human-interaction paths safer. Detection is advisory except where an operation explicitly requires a human prompt.

The implementation records:

- stdin, stdout, and stderr TTY state;
- whether execution is in CI;
- whether `TERM=dumb` applies;
- whether the process is nested under an inherited Zed context;
- output mode (`human`, `plain`, or `machine`);
- shell family and the source of that inference;
- terminal family;
- per-stream color capability;
- Unicode and hyperlink capability;
- terminal width when available.

The snapshot is computed once and reused. Zed publishes a reserved `ZED_PKG_CONTEXT_*` snapshot to subprocesses so nested tools can make consistent decisions without repeatedly guessing from mutable process state.

## Prompt safety

An explicit interactive checkpoint is allowed only when all of these are true:

1. stdin is a terminal;
2. stderr is a terminal;
3. execution is not CI;
4. the terminal is not dumb.

stdout is deliberately not part of the prompt gate. A user may pipe or redirect command output while still receiving a prompt on stderr.

A piped `yes` on stdin does not satisfy `--interactive`. Non-interactive invocations fail closed instead of consuming redirected input as human confirmation.

## CI and pseudo-terminal tests

Production CI detection stays fail-closed. Tests that intentionally simulate a human terminal through a PTY may clear CI only for the child process created by the PTY harness. They must not disable CI detection globally for the job or process tree.

This distinction caught a real test-harness mismatch during review: the new prompt policy correctly rejected a PTY running inside GitHub Actions until the simulated-human child explicitly set `ZED_PKG_FORCE_CI=0`.

## Overrides

Zed supports deterministic `ZED_PKG_FORCE_*` overrides for tests and embedding. It also accepts the shared `F2E_FORCE_*` spellings so the Zed and flags2env context models can be exercised consistently.

The override surface includes forced stdin/stdout/stderr TTY state, CI, color, and Unicode behavior. Shell identity can be overridden with `ZED_PKG_SHELL` or `F2E_SHELL`.

These are deterministic controls, not a reason for ordinary production code to bypass safety checks.

## Shell detection confidence

Shell detection is best effort and source-labelled. An inferred current shell is not authoritative enough to replace explicit shell arguments for commands that emit shell syntax.

The implementation considers, in order, explicit Zed/flags2env overrides, conventional shell environment variables, PowerShell markers, `COMSPEC`, and inherited parent context. Unknown remains a valid result.

Supported normalized families include Bash, Zsh, Fish, Nushell, PowerShell, `cmd`, POSIX `sh`, and unknown.

## flags2env companion contract

flags2env exposes terminal context through an additive C99 ABI. Normal parser results and CLI output do not change, and the API does not mutate the caller's process environment.

The companion implementation reports the same broad execution-context vocabulary and returns either JSON or a string-only environment map. The vendored parser source files remain untouched so parser provenance and identity checks stay intact.

## Cross-platform certification

The exact Zed candidate `423c07d8204e1b409e27a28fe450abcdabbc668e` was certified before merge.

### Linux

- formatting passed;
- 511 nextest tests passed;
- doctests passed;
- Clippy passed with warnings denied;
- Node and Rust lifecycle tests passed;
- OCI round trips passed;
- Docker install/recovery boundaries passed, including interactive PTY checkpoints.

### macOS

- formatting passed;
- the full nextest suite passed;
- doctests passed;
- Node and Rust lifecycle tests passed.

### Windows

- formatting passed;
- terminal-context unit contracts passed on a native Windows runner;
- cross-process lock and crash contracts passed;
- shared-artifact deduplication passed;
- owner-termination recovery passed.

The flags2env companion passed its native sanitizers, formal methods, end-to-end, client packaging, Nix, shell/help, and Zed-package round-trip workflow groups before merge.

## Compatibility rules

This feature does not authorize automatic output-format changes. In particular:

- JSON or line-oriented output must not become human-oriented because stdout is a TTY;
- shell syntax generation continues to require explicit shell selection where correctness depends on it;
- exit-code contracts remain stable;
- redirected stdout remains compatible with an explicit stderr prompt;
- unknown shell/terminal identity is handled conservatively rather than guessed into a different command shape.

## Ownership and follow-up

`zed-cli` owns prompt gating, nested context publication, and use of terminal context inside Zed commands. `flags-2-env` owns its opt-in portable detector API. Consumers should reuse those surfaces instead of growing independent terminal heuristics.

Future terminal-sensitive work should link `DEN-2591` or `DEN-2581` as appropriate and preserve the Linux/macOS/Windows certification matrix.