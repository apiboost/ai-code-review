# Apiboost AI Code Review

A reusable GitHub Actions workflow that runs an **on-request, advisory AI review** of a pull request. It
reviews the code change itself, and whether the contribution follows the repository's own contribution
guides (DCO sign-off, Conventional Commits, PR template, AI-assistance disclosure).

It is built to be safe on **public repositories**: nobody outside the project can trigger it, and no
pull request code ever runs in a job that holds a secret.

- **On request only.** A maintainer with write access applies the `ai-review` label.
- **Engine-neutral.** The `engine` and `model` inputs pick the reviewer. `claude` is implemented today.
- **Advisory.** The workflow posts a `COMMENT` review plus a sticky summary. It never approves or blocks.

## How it works

```
maintainer applies `ai-review` label
        │
        ▼
 gate    (write token, no secret)  validate inputs + event → remove label → verify requester has write access
        │
        ▼
 review  (secret, read-only token) checkout BASE → fetch PR head by SHA as data → diff + snapshot
                                   → deterministic contribution facts → run engine (read-only tools) → normalize
        │
        ▼
 post    (write token, no secret)  filter output → inline COMMENT review → sticky summary + contribution checklist
```

The sticky summary includes a **contribution checklist** with two kinds of rows, labelled separately:

| Source | Rows | How it's computed |
|---|---|---|
| `verified` | DCO sign-off, sign-off identity, Conventional Commits | Shell and `jq` in the workflow. PR content can't influence them. |
| `AI-assessed` | Template filled in, linked issue, type-of-change matches the diff, tests and docs updated, AI-disclosure consistency, and so on | The engine, judged against the guides you list in `contribution-docs` |

## Usage

### 1. Add a caller workflow

```yaml
# .github/workflows/ai-code-review.yml
name: AI Code Review

on:
  # Label-gated and write-access-checked. PR content is only read as data.
  pull_request_target: # zizmor: ignore[dangerous-triggers]
    types: [labeled]
    branches: [main]

permissions: {}

concurrency:
  group: ai-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  review:
    if: github.event.label.name == 'ai-review'
    uses: apiboost/ai-code-review/.github/workflows/review.yml@<full-commit-sha> # vX.Y.Z
    permissions:
      contents: read
      issues: write
      pull-requests: write
    with:
      engine: ${{ vars.AI_REVIEW_ENGINE || 'claude' }}
      model: ${{ vars.AI_REVIEW_MODEL || 'sonnet' }}
      prompt-file: .github/ai-review/prompt.md
      contribution-docs: CONTRIBUTING.md,.github/PULL_REQUEST_TEMPLATE.md
      require-dco: true
      conventional-commits: true
```

Pin `uses:` to a **full commit SHA**, and let Dependabot (`github-actions` ecosystem) bump it. Don't pass
the engine secret from the caller: keep it in the environment (step 3), so that environment is the only
place it can come from.

`pull_request_target` runs the copy of the caller workflow on the PR's **base** branch. Add it to every
branch that PRs target.

### 2. Write the review rules

Create `prompt-file` (for example `.github/ai-review/prompt.md`) with the repository-specific rules: what the
project is, the review categories (`[SECURITY]`, `[LICENSE]`, `[STANDARD]`, …), and conventions worth
enforcing. Leave out tool, output-format and contribution boilerplate, which the workflow supplies. This
file and every `contribution-docs` file are read from the **trusted base checkout**, so a PR can't change
the rules it is reviewed against.

### 3. One-time repository setup

1. **Environment.** Under Settings → Environments, create `ai-review`. Limit deployment branches to the
   branches PRs target, and add the engine secret there (for `claude`, `ANTHROPIC_API_KEY`). Use a key
   dedicated to this purpose, in its own workspace with a monthly spend limit.
2. **Label.**
   `gh label create ai-review --description "Maintainer: request advisory AI review" --color 5319e7`
3. **Optional variables.** `AI_REVIEW_ENGINE` and `AI_REVIEW_MODEL` switch the engine or model without
   a PR.

### Requesting a review

Apply the `ai-review` label. The workflow removes it as soon as the run starts. After new pushes,
re-apply it to review the new head. Each run reviews the exact commit that was labelled; commits pushed
afterwards are not included.

## Inputs

| Input | Default | Description |
|---|---|---|
| `engine` | `claude` | Review engine. Unknown engines fail fast in `gate`. |
| `model` | `sonnet` | Model identifier passed to the engine. |
| `prompt-file` | *(required)* | Repository-specific review rules (trusted base). |
| `contribution-docs` | `CONTRIBUTING.md,.github/PULL_REQUEST_TEMPLATE.md` | Comma-separated guides the PR is checked against (trusted base). Missing files are skipped with a warning. |
| `require-dco` | `false` | Verified rows for `Signed-off-by` presence (fail) and sign-off name matching the commit author (warn; emails often legitimately differ). |
| `conventional-commits` | `false` | Verified row for `type(scope)!: summary` subjects on non-merge commits. |
| `conventional-types` | `build,chore,ci,docs,feat,fix,perf,refactor,revert,style,test` | Allowed commit types. Match your commitlint `type-enum`. |
| `label` | `ai-review` | Request label. |
| `environment` | `ai-review` | Environment that holds the engine secret. |
| `skip-paths` | lockfiles, `LICENSE`, `THIRD-PARTY-LICENSES.md` | Extended regex of paths left out of the code review. |
| `max-turns` | `60` | Engine turn limit. |
| `timeout-minutes` | `15` | Engine step timeout. |
| `max-comments` | `30` | Maximum inline comments per run. Extra comments are dropped and counted in the summary. |

| Secret | Engine | Notes |
|---|---|---|
| `ANTHROPIC_API_KEY` | `claude` | Store it as an **environment** secret of `environment`. |

## Threat model

The review takes untrusted input (a stranger's pull request) and puts it in front of an AI with an API key
nearby. Each risk below has a specific control:

| Risk | Control |
|---|---|
| Anyone on the internet triggers paid reviews | `pull_request_target` + `labeled` only. Applying a label needs triage access, and `gate` also requires **write** access (`collaborators/{user}/permission`). On public repositories, `gate` refuses every other event. |
| PR code runs with the secret (e.g. a `bunfig.toml` preload, an `.npmrc`, agent hooks in `.claude/settings.json`, `.mcp.json`) | The workspace root is always the **base branch**. The PR head is fetched by SHA and only the changed files are extracted under `.ai-review/head/`. Agent config and instruction files and runtime config are deleted from that snapshot. Nothing installs or runs PR content. |
| A PR symlink exposes files outside the workspace (e.g. `/proc/self/environ`) | Every symlink in the snapshot is replaced with a text note. |
| Prompt injection through code, the description or commit messages | Instructions come only from the base branch. The preamble marks every PR-derived file as untrusted data, and injection attempts are reported. The engine has **read-only file tools** (no shell, network or edits). DCO and commit-format results are computed outside the model. |
| Model output abused (pinging people, commenting on unrelated files, approving) | `post` drops comments outside the changed files or with invalid lines, caps the count, neutralises `@mentions`, strips the summary marker, and always posts `COMMENT`. |
| The API key and a write token in the same place | `review` gets the secret and a **read-only** token. `gate` and `post` get the write token and no secret. Checkout uses `persist-credentials: false`. |
| A force-push after a maintainer's request | The labelled SHA from the event is fetched, diffed and used as the review's `commit_id`. Later commits need a new request. |
| Template injection in workflow scripts | No `${{ }}` inside `run:`. Everything goes through `env:`, and inputs are validated in `gate`. `actionlint` and `zizmor --persona=auditor` run in CI. |
| A compromised action | Every action is pinned to a full commit SHA. Callers pin this workflow by SHA too. |

Private repositories may also call the workflow from `pull_request` (`opened`, `synchronize`, `reopened`,
`ready_for_review`) for automatic reviews. There, the caller workflow comes from the PR itself, so only use
it where every PR author is trusted.

## Adding an engine

1. Add the engine's name to the `case` in `gate` → *Validate request*.
2. Add an optional secret for it under `on.workflow_call.secrets`, and a branch in *Check engine credentials*.
3. Add an engine step guarded by `if: inputs.engine == '<name>' && …`. Pin any action to a full SHA.
4. Add a branch to *Normalize engine output* that maps the engine's native output to the shared JSON
   schema, plus `outcome`, `subtype`, `cost` and `turns`.

Before a new engine is allowed, it needs its own sandbox review. It must run with **read-only file tools**
(no shell, network or file writes), must not load configuration or instructions from the PR content, and
must not be able to read files outside the workspace.

### `claude` engine notes

The `claude` engine runs `anthropics/claude-code-action` with `--allowedTools 'Read,Grep,Glob'` and an
explicit deny list (`Bash`, `Write`, `Edit`, `WebFetch`, `WebSearch`, `Task`, and the action's comment
tools). The action also checks that the actor has write access, and on PR events it restores `.claude/`,
`.mcp.json` and `CLAUDE.md` from the base branch. Both repeat controls this workflow already enforces
itself. The action is given the job's read-only `github_token`, which skips its GitHub App token
exchange.

## Versioning

Releases are tagged `vX.Y.Z`. Callers pin by SHA, so any change to an input, output, default or the
security behaviour is a **breaking change** and gets a major version.

## License

Apache License 2.0. See [LICENSE](./LICENSE). Contributions are accepted under the
[Developer Certificate of Origin](./DEVELOPER_CERTIFICATE_OF_ORIGIN.md); see [CONTRIBUTING.md](./CONTRIBUTING.md).
