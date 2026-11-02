# Review rules: Apiboost AI Code Review

This repository is **Apiboost AI Code Review**, a public, Apache-2.0 reusable GitHub Actions workflow
(`.github/workflows/review.yml`) that runs a label-gated, engine-neutral AI review of pull requests. Other
repositories, including public ones, call it from `pull_request_target` workflows, pinned by commit SHA.
Its security model is described in `README.md` and must not regress.

Use these categories, most severe first:

## 1. `[SECURITY]` (critical)

- **Trust boundary:** the job that holds the engine secret must never execute, `source`, install or
  configure anything from the PR head. The workspace root must stay the trusted base checkout, and PR
  content may only be read as data from `.ai-review/`. Flag any `actions/checkout` of the PR head ref
  or SHA, `npm`/`pnpm`/`bun` install or run against PR content, and anything that removes the symlink
  replacement or the config-file stripping in the snapshot step.
- **Least privilege:** the API key and a write-scoped token must never be in the same job. Flag any
  widening of job `permissions`, `id-token: write`, `secrets: inherit`, or `persist-credentials: true`.
- **Template injection:** `${{ }}` expressions must not appear inside `run:` scripts. Values go
  through `env:`. Flag new event fields (title, body, branch names, labels, commit messages) that
  reach a shell, a GitHub output or the engine prompt without going through `env:`/files.
- **Gating:** public repositories must keep requiring an explicit label request from a user with write
  access. Flag anything that lets a fork PR trigger a review, skips the permission check, or reviews a
  SHA other than the one in the triggering event.
- **Output handling:** the review event must stay `COMMENT`. Model output must stay filtered (paths in
  the changed-file list, positive integer lines, the comment cap, neutralised `@mentions`).
- **Supply chain:** every third-party action must be pinned to a full commit SHA with a version comment.
  Downloaded tools must be version-pinned and checksum-verified.

## 2. `[LICENSE]` (blocking)

- Source files (workflows, scripts) carry the header `This source file is part of Apiboost AI Code
  Review.` / `Copyright (c) Apiboost, Inc.` / `SPDX-License-Identifier: Apache-2.0`.
- No code copied from sources with incompatible licenses.

## 3. `[WORKFLOW]` (important)

- **Engine neutrality:** vendor or model names appear only in engine-specific steps (the engine step, its
  secret, its normalize branch) and the README's engine section. Labels, environments, file names,
  markers and generic text stay vendor-neutral.
- **Compatibility:** callers pin SHAs, so renaming or removing an input or output, or changing a
  default, is a breaking change and needs a `!`/`BREAKING CHANGE` commit and a major version.
- **Shell robustness:** `set -euo pipefail`, quoted expansions, NUL-safe handling of file names, and no
  reliance on GNU/BSD differences beyond what `ubuntu-latest` provides.
- `actionlint` and `zizmor --persona=auditor` must stay clean. Suppressions need a written justification.

## 4. `[DOCS]` (important)

- `README.md` must stay in sync with the `workflow_call` inputs, secrets, setup steps and threat model.

## 5. `[QUALITY]` (advisory)

- Readability of long scripts, dead code, and confusing failure messages that a maintainer would see
  in the PR summary.
