# Deterministic offline release-plan review

Status: **implementation in review** in
[`zed-pkg/zed-cli#31`](https://github.com/zed-pkg/zed-cli/pull/31), tracked by
Linear DEN-1301.

## Why this exists

`zed release plan` already computes the credential-free release model used to
explain what a publish would do. Terminal prose is useful interactively and
`--json` is the stable machine boundary, but neither is an efficient review
artifact for a multi-target release with native registries and forge mirrors.
A reviewer needs one portable document that can be retained by CI and opened
without a server, registry credential, or network connection.

The browser report must not become a second planner. The JSON emitted by the
Rust CLI remains authoritative; rendering is a pure, fail-closed projection of
that model.

## Review flow

After the implementation merges, a repository can generate and inspect the
same plan in three forms:

```sh
zed release plan
zed release plan --json > build/release-plan.json
node scripts/render-release-plan-html.mjs \
  --input build/release-plan.json \
  --output build/release-plan.html
```

The renderer also accepts JSON on standard input. The resulting HTML opens
through `file://` and contains source provenance, exact Zed/native/forge
counts, explicit empty states, artifact tables, and client-side filtering.
Escape clears the filter and restores the complete plan.

## Security and integrity contract

The renderer and report are intentionally narrow:

- validate the expected release-plan shape before rendering and reject missing
  or malformed fields;
- HTML-escape every plan-derived value before placing it in text or an
  attribute;
- authorize the static inline stylesheet and filter code with exact SHA-256
  Content Security Policy hashes rather than broad `unsafe-inline` rules;
- include no remote scripts, styles, fonts, analytics, images, or network
  calls;
- write through a private same-directory temporary file and atomically rename
  it into place;
- refuse an existing symbolic-link output instead of following or replacing
  its target;
- reject unknown renderer options and missing option values;
- never load registry tokens or invoke a publish endpoint.

The same-directory rename is important. A temporary file on another filesystem
cannot be atomically renamed, and a predictable world-readable temporary file
would expose a partially written plan. Symlink refusal prevents a chosen
output path from redirecting the writer to an unrelated file.

## Browser automation contract

The GitHub Actions workflow builds the locked Rust CLI, generates a realistic
npm plus crates.io plan with forge mirrors, renders the HTML, and opens it in
Chromium through `file://`. The test verifies:

- semantic landmarks and source provenance;
- exact counts and table rows;
- filtering and Escape reset;
- keyboard focus;
- narrow/mobile containment;
- no console or page errors;
- no HTTP or HTTPS requests;
- retention of the reviewed HTML as a CI artifact.

This is browser automation over the actual Rust-generated release model, not a
fixture-only screenshot test.

## Relationship to publishing

The report is advisory and read-only. It does not weaken any existing publish
preflight, clean-tree, tag, validation, authentication, or registry policy.
A protected release workflow can retain both the exact JSON plan and the HTML
review artifact, then run the ordinary publish command only after the normal
review and environment gates succeed.

## Drift rule

Any future renderer, UI, or automation must consume the authoritative JSON
model or another versioned contract emitted by the planner. Reimplementing
release routing in JavaScript, a web server, or CI configuration would create
an unreviewable second source of truth and is out of scope.
