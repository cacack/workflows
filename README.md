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
    uses: cacack/workflows/.github/workflows/dependabot-automerge.yml@42f4a8b03bbeb4cda758290b3c8cdbb88ed6d33e # v2.0.0
    secrets: inherit
```

By default every ecosystem is in scope. A repo that wants auto-merge for its
Actions bumps but hand review for its language dependencies sets `ecosystems`:

```yaml
jobs:
  automerge:
    uses: cacack/workflows/.github/workflows/dependabot-automerge.yml@42f4a8b03bbeb4cda758290b3c8cdbb88ed6d33e # v2.0.0
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

### Egress

The job runs `step-security/harden-runner` with `egress-policy: block`. Its egress
is small and knowable — GitHub's API and the hosts the runner pulls actions and
their assets from — so the built-in allowlist covers it, and a consumer that
standardizes on `block` keeps that posture without configuring anything.

A caller cannot re-harden a called workflow's steps from its stub, which is why
the posture is exposed as inputs rather than fixed. If an update to one of the
pinned actions reaches a host outside the list, the job fails closed: set
`egress-policy: audit` to see what it wanted, then pass the corrected list via
`allowed-endpoints`. Setting `allowed-endpoints` replaces the built-in list, so
restate the defaults alongside any addition.

### Inputs

| Input | Default | Notes |
|---|---|---|
| `merge-method` | `merge` | Passed to `gh pr merge`. Repos here use merge commits. |
| `ecosystems` | *(empty — all)* | Comma-separated allowlist of Dependabot package ecosystems, no spaces (e.g. `github_actions,cargo`). |
| `egress-policy` | `block` | `harden-runner` policy. `audit` observes instead of enforcing. |
| `allowed-endpoints` | *(built-in list)* | Space-separated `host:port` allowlist used under `block`. Replaces the default rather than extending it. |

## Versioning

Two kinds of tag, following the `actions/*` convention:

- **`v2`** moves. It is only ever moved for backward-compatible changes — a new
  optional input, a bug fix, a clearer log line. Cut the next major for anything
  that alters the calling contract (new required secret, changed input semantics,
  a default that behaves differently).
- **`v2.<minor>.<patch>`** is fixed by convention, cut alongside each move of
  `v2`. Convention is all it is — git does not stop a force-push to any tag.

Which one to pin depends on what a repo wants:

| Pin | Behavior |
|---|---|
| `@<sha> # v2.0.0` | The only pin git enforces. Preferred — Dependabot bumps the SHA and the comment for you. |
| `@v2.0.0` | Fixed by convention. Readable, but a force-push would move it. |
| `@v2` | Picks up compatible changes automatically. Fine for repos that want fixes without tending pins. |

Prefer the SHA. It is what the `actions/*` ecosystem settled on, it is what
OpenSSF Scorecard's `Pinned-Dependencies` check looks for — which does evaluate
reusable-workflow `uses:` refs, not just actions — and it costs nothing to
maintain, since Dependabot treats this reference like any other action.

That these are tags in a repo you control lowers the stakes but does not remove
them: an account compromise that can move a tag can move it here too.

`v2` was cut under the last clause of that first rule: `dependabot-automerge.yml`
now blocks egress by default where `v1` audited it. A repo whose Dependabot PRs
reach a host outside the built-in allowlist would see merges start failing on the
move alone, so it is opt-in per repo rather than delivered by moving `v1`. `v1`
stays on the auditing behavior.
