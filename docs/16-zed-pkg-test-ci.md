# 16. The zed-pkg-test CI harness (GitHub Actions only)

**Issue:** doc 15 defines the manifest and the "complement, don't replace"
model. This doc is the proof: a set of real repos in the
[`zed-pkg-test`](https://github.com/zed-pkg-test) org, in different languages,
some depending on each other **via zed**, each tested with nothing but GitHub
Actions — no registry server, no external services.

## Layout: interdependent lib/app pairs

Each ecosystem is a pair: a `*-lib` published to the zed registry and a `*-app`
that sources it via zed while its native package manager owns everything else.

| Pair | Native manager zed complements | How the app consumes the zed dep |
| --- | --- | --- |
| [`node-lib`](https://github.com/zed-pkg-test/node-lib) + [`node-app`](https://github.com/zed-pkg-test/node-app) | **npm** | `[install].dir = ".vendor/.zed"`, `adapter = "node"` → `node_modules/@zedtest/node-lib` link + `.zed/node_path`; app `require("@zedtest/node-lib")` |
| [`rust-lib`](https://github.com/zed-pkg-test/rust-lib) + [`rust-app`](https://github.com/zed-pkg-test/rust-app) | **cargo / crates.io** | `[install].dir = ".vendor/.zed"`; app `Cargo.toml` path dep `rust-lib = { path = ".vendor/.zed/zedtest/rust-lib" }` |

Both are verified green locally and in CI: the app builds and runs, using the
zed-sourced dependency, with the native toolchain none the wiser.

## The workflow (no more than GHA)

Every `*-app` ships `.github/workflows/zed-e2e.yml` with the same shape:

1. **Build the zed CLI from source.** `git clone` `zed-cli` **and**
   `zed-interfaces` (they must be siblings — the CLI path-depends on the
   interfaces crate), then `cargo build --release --bin zed`. Both are public,
   so the clone needs no token.
2. **Publish the dependency to a hermetic `file://` registry.** `git clone` the
   `*-lib` repo and `zed publish --skip-vcs-checks` into
   `file://$RUNNER_TEMP/zed-reg`. No server — the file registry *is* the
   registry. `ZED_PKG_HOME` / `ZED_PKG_REGISTRY` are exported once via
   `$GITHUB_ENV`.
3. **Let the native manager run** (`npm install`, or cargo resolves crates.io) —
   this is the "complement" half.
4. **`zed install`** — materializes the select deps into `[install].dir` and
   runs the adapter (node_modules link + `.zed/node_path`, or just the tree for
   cargo's path dep).
5. **Build/run in the language** and assert the zed dep resolved.

Because the registry is `file://` and the only services are the language
toolchains' own `setup-*` actions, the whole thing runs on a stock
`ubuntu-latest` runner with no secrets.

## Extending to the rest of the matrix

New ecosystems follow the identical pattern; only step 4→5 changes, per the
per-language recipe in [doc 15](15-manifest-and-dep-locations.md):

- **typescript / nextjs** — same as node (add a `tsc`/`next build` step).
- **go** — `go.mod` `replace zedtest/x => ./.vendor/.zed/zedtest/x`; `go build`.
- **java / kotlin** — a `[build]` step in the `*-lib` compiles a jar; the app
  uses `javac -cp "$(cat .zed/classpath)" …` (adapter `java`); `setup-java`.
- **dart / flutter** — `pubspec.yaml` `path:` dependency; `dart`/`flutter test`.
- **elixir / erlang** — `mix.exs` / rebar `{path, …}` (or `ERL_LIBS`).
- **gleam** — path dependency in `gleam.toml`.
- **cpp** — `add_subdirectory(.vendor/.zed/zedtest/…)`; CMake.

The `*-lib` in each case carries both its `.zpkg.toml` (for zed) and its native
manifest (`package.json` / `Cargo.toml` / `pom.xml` / `pubspec.yaml` / …), so
the native toolchain recognizes it the moment zed drops it into `[install].dir`.

## Status: node + rust proven; the rest are template-ready

`node-*` and `rust-*` exist, run in CI, and prove the model end to end. The
remaining languages the org will cover (ts, nextjs, go, java, dart, flutter,
elixir, erlang, gleam, cpp) are stamped from the same lib/app + `zed-e2e.yml`
template with the doc-15 recipe swapped in for the build/run step.
