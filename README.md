# workflows

Reusable GitHub Actions workflows shared across `cacack/*` repos.

Each workflow is `on: workflow_call`. Consumers keep a short stub that supplies the
trigger and the secrets; the logic lives here once, so a fix lands in one place
instead of being copy-pasted into every repo and drifting.

Consumers pin a tag, so a change here is opt-in per repo rather than an immediate
blast to all of them.

## `dependabot-automerge.yml`

Enables GitHub auto-merge on Dependabot PRs for non-major updates, and approves
them. Majors are left alone for manual review.

**Consumer stub** — `.github/workflows/dependabot-automerge.yml`:

```yaml
name: Dependabot Auto-Merge

on: pull_request_target

jobs:
  automerge:
    uses: cacack/workflows/.github/workflows/dependabot-automerge.yml@v1
    secrets: inherit
```

By default every ecosystem is in scope. A repo that wants auto-merge for its
Actions bumps but hand review for its language dependencies sets `ecosystems`:

```yaml
jobs:
  automerge:
    uses: cacack/workflows/.github/workflows/dependabot-automerge.yml@v1
    with:
      ecosystems: github_actions
    secrets: inherit
```

A PR outside the allowlist logs why it was skipped and the run ends green.

### Requirements

The calling repo needs a GitHub App — named `<repo>-steward` by convention — with
**Contents: write**, **Pull requests: write**, and **Workflows: write**, installed
on that repo only, exposed as the Actions secrets `BOT_APP_ID` and
`BOT_PRIVATE_KEY`. `secrets: inherit` passes them through.

`Workflows: write` is the load-bearing one. Most Dependabot PRs edit
`.github/workflows/*`, and GitHub refuses to enable auto-merge on such a PR
without it — a permission `GITHUB_TOKEN` cannot be granted through a workflow's
`permissions:` block. Using an App token also means the resulting merge commit
triggers CI, which a `GITHUB_TOKEN` push does not.

Registration tooling lives in the `git-repositories` infra repo
(`scripts/new-repo-steward.sh`).

### Inputs

| Input | Default | Notes |
|---|---|---|
| `merge-method` | `merge` | Passed to `gh pr merge`. Repos here use merge commits. |
| `ecosystems` | *(empty — all)* | Comma-separated allowlist of Dependabot package ecosystems, no spaces (e.g. `github_actions,cargo`). |

## Versioning

Two kinds of tag, following the `actions/*` convention:

- **`v1`** moves. It is only ever moved for backward-compatible changes — a new
  optional input, a bug fix, a clearer log line. Cut `v2` for anything that
  alters the calling contract (new required secret, changed input semantics,
  a default that behaves differently).
- **`v1.<minor>.<patch>`** is immutable, cut alongside each move of `v1`.

Which one to pin depends on what a repo wants:

| Pin | Behavior |
|---|---|
| `@v1` | Picks up compatible changes automatically. Fine for repos that want fixes without tending pins. |
| `@v1.1.0` | Frozen. Nothing changes until the repo bumps it deliberately. |

Both are tags in a repo you control, so neither is a supply-chain hazard the way
a third-party moving tag would be — the choice is about how much automatic
change a consumer wants, not about trust.
