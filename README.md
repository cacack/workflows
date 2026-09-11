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

permissions:
  contents: read
  pull-requests: read

jobs:
  automerge:
    uses: cacack/workflows/.github/workflows/dependabot-automerge.yml@42f4a8b03bbeb4cda758290b3c8cdbb88ed6d33e # v2.0.0
    secrets:
      BOT_APP_ID: ${{ secrets.BOT_APP_ID }}
      BOT_PRIVATE_KEY: ${{ secrets.BOT_PRIVATE_KEY }}
```

`secrets: inherit` is shorter and also works, but it hands this workflow every
secret the calling repo holds when it declares exactly two. Map them.

The `permissions:` block is not optional — see [Permissions](#permissions).

By default every ecosystem is in scope. A repo that wants auto-merge for its
Actions bumps but hand review for its language dependencies sets `ecosystems`:

```yaml
permissions:
  contents: read
  pull-requests: read

jobs:
  automerge:
    uses: cacack/workflows/.github/workflows/dependabot-automerge.yml@42f4a8b03bbeb4cda758290b3c8cdbb88ed6d33e # v2.0.0
    with:
      ecosystems: github_actions
    secrets:
      BOT_APP_ID: ${{ secrets.BOT_APP_ID }}
      BOT_PRIVATE_KEY: ${{ secrets.BOT_PRIVATE_KEY }}
```

A PR outside the allowlist logs why it was skipped and the run ends green.

### Requirements

The calling repo needs a GitHub App — named `<repo>-steward` by convention — with
**Contents: write**, **Pull requests: write**, and **Workflows: write**, installed
on that repo only, exposed as the Actions secrets `BOT_APP_ID` and
`BOT_PRIVATE_KEY`. The stub maps them through by name.

`Workflows: write` is the load-bearing one. Most Dependabot PRs edit
`.github/workflows/*`, and GitHub refuses to enable auto-merge on such a PR
without it — a permission `GITHUB_TOKEN` cannot be granted through a workflow's
`permissions:` block. Using an App token also means the resulting merge commit
triggers CI, which a `GITHUB_TOKEN` push does not.

Registration tooling lives in the `git-repositories` infra repo
(`scripts/new-repo-steward.sh`).

### Permissions

The stub must grant at least what the called job declares:

```yaml
permissions:
  contents: read
  pull-requests: read
```

A called workflow's job cannot request more permission than its caller grants.
Ask for less and GitHub rejects the call with `startup_failure` — the run ends
before any step executes, so there is no log explaining it and no check to read.
`pull-requests` is the one people miss; `fetch-metadata` needs it to read the PR.

Omitting the block entirely is not a safe shortcut. It falls back to the repo's
default `GITHUB_TOKEN` permissions, and the hardened default — *Read repository
contents and packages permissions* — does not include `pull-requests`, so a
hardened repo fails the same way. Declare it.

This is worth stating plainly because the failure is invisible until a real
Dependabot PR arrives. `pull_request_target` evaluates the **base branch's**
workflow, so the stub does not execute on the PR that introduces it — the
conversion looks clean and the first bump weeks later does not merge.
`cacack/gedcom-go#371` sat green and open for five days on exactly this.

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

## `dependabot-watch.yml`

Fails, and files one deduplicated issue, when a Dependabot advisory has been open
longer than a grace period. Closes that issue again on the next clean run.

It exists because a repo can be entirely green — CI passing, site up — and still be
sitting on an open high-severity alert: nothing in a normal merge flow ever asks
GitHub whether alerts are open. In the incident that prompted it, an alert stayed
open for 25 days with no Dependabot PR, even though a satisfying fix had been
published ten days before the alert opened. It was found only because someone went
looking.

It watches the *symptom* — an alert that is open and staying open — not the cause.
Why Dependabot skips a given alert is recorded only in the update-job logs at
`/network/updates`, which no REST endpoint exposes, so the cause is not detectable
from automation at all.

**Consumer stub** — `.github/workflows/dependabot-watch.yml`:

```yaml
name: Dependabot watch

on:
  schedule:
    - cron: '43 13 * * *'
  workflow_dispatch:
    inputs:
      grace_days:
        description: Grace period in days (overrides the default)
        required: false

permissions:
  contents: read
  issues: write

jobs:
  watch:
    uses: cacack/workflows/.github/workflows/dependabot-watch.yml@<sha> # v2.1.0
    with:
      grace-days: ${{ inputs.grace_days }}
      threshold-doc-url: https://github.com/<owner>/<repo>/blob/main/docs/decisions/NNNN.md
    secrets:
      BOT_APP_ID: ${{ secrets.BOT_APP_ID }}
      BOT_PRIVATE_KEY: ${{ secrets.BOT_PRIVATE_KEY }}
```

**The schedule lives in the stub, not here.** A `workflow_call` workflow cannot
declare `schedule`, so each consumer owns its own cron — which is what you want
anyway, to stagger runs against whatever else that repo runs daily.

`grace-days` is a *string* whose default is empty rather than a number defaulting to
7, so the stub above can pass a dispatch input straight through. On a scheduled run
that expression is empty, and an explicitly-passed empty value does **not** fall back
to an input's default — only an omitted input does. The workflow therefore resolves
the fallback in shell (`${GRACE_DAYS:-7}`). Doing it in an expression with
`${{ inputs.grace-days || 7 }}` would look equivalent and quietly rewrite a
deliberate `grace-days: 0` into `7`, because GitHub treats the string `"0"` as falsy.

### Credentials

`GITHUB_TOKEN` **cannot** read the Dependabot alerts API — `security-events: read`
looks like the permission that covers it and does not; the API answers `403 Resource
not accessible by integration`. The alerts read therefore authenticates as the
caller's `<repo>-steward` App.

Only that one step uses the App token; the issue it files and closes still runs as
`GITHUB_TOKEN`. So the App needs exactly one permission, **Dependabot alerts: read** —
narrower than what `dependabot-automerge.yml` asks of the same App. A PAT would also
work and is not recommended: it is a long-lived credential tied to a person, and a
watch whose purpose is to prevent a silent failure should not be guarded by one.

### Permissions

As with `dependabot-automerge.yml`, the stub must grant at least what the called job
declares — `contents: read` and `issues: write`. Ask for less and GitHub rejects the
call with `startup_failure`, before any step executes, with no log explaining it.
`issues: write` is the one to get right here: without it the workflow detects a stale
advisory and then cannot tell anyone, which is the failure it exists to prevent.

### Inputs

| Input | Default | Notes |
|---|---|---|
| `grace-days` | *(empty → 7)* | Days an alert may stay open before the run fails. String, so a dispatch input passes straight through. |
| `severities` | `high,critical` | Comma-separated severities that fail the run, no spaces. Everything else open is logged as context. |
| `threshold-doc-url` | *(empty)* | Link to the consumer's own decision record for the grace period, cited in the filed issue. Empty omits the citation. |
| `egress-policy` | `block` | `harden-runner` policy. `audit` observes instead of enforcing. |
| `allowed-endpoints` | *(built-in list)* | Space-separated `host:port` allowlist used under `block`. Replaces the default rather than extending it. |

Leaving `threshold-doc-url` empty is fine, but if the repo has written its grace
period down somewhere, pass it: the filed issue then points at that decision instead
of implying this workflow is where the number lives.

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
