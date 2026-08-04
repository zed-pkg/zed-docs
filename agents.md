# Agent instructions

## Scope and hierarchy

- These instructions apply to the whole `zed-pkg/zed-docs` repository unless a deeper lowercase `agents.md` adds narrower rules.
- Before editing, resolve the current working directory and load every readable ancestor `agents.md` from the filesystem root to the working directory. Do not search siblings. Resolve symlinks, deduplicate resolved files, and report unreadable or cyclic instruction files.
- `.claude/CLAUDE.md`, `.gemini/GEMINI.md`, and `.openai/AGENTS.md` are pointers only. Never duplicate instructions in tool-specific files.

## Repository role

This repository is the durable architecture, operations, contract, and contributor documentation set for Zed. Its job is to describe implemented behavior precisely and make cross-repository decisions discoverable.

## Working rules

- Verify technical claims against the current source repository, schema, workflow, or released artifact. Do not document planned behavior as implemented behavior.
- Keep commands executable, paths and repository names exact, and examples free of real credentials, production identifiers, and unsafe defaults.
- Distinguish normative contracts, implementation notes, operational procedures, proposals, and historical decisions.
- Update links, diagrams, tables of contents, version references, and cross-repository dependencies together.
- Prefer concise source-linked explanations over copied code or duplicated reference material that can drift.
- Preserve migration, rollback, failure-mode, and security context when simplifying operational documentation.
- Never commit tokens, private URLs, kubeconfigs, customer data, screenshots containing secrets, or production environment files.
- Run markdown, link, spelling, schema/example, and site-build checks already defined by the repository.
- Before closing a pull request as superseded, outmoded, obsolete, replaced, or duplicated by a successor, incorporate at least one substantive item from that pull request into the successor. Apply this once per predecessor and record the predecessor, salvaged item, and incorporation location in the successor body and predecessor closing comment, following `CONTRIBUTING.md`.

## Validation

The pinned `agents policy` workflow validates this hierarchy and the three tool pointers. Follow `README.md` and existing CI for documentation-specific validation before requesting review.
