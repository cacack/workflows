# workflows

Reusable GitHub Actions workflows shared across `cacack/*` repos.

Each workflow is `on: workflow_call`. Consumers keep a short stub that supplies the
trigger and the secrets; the logic lives here once, so a fix lands in one place
instead of being copy-pasted into every repo and drifting.

Consumers pin a tag, so a change here is opt-in per repo rather than an immediate
blast to all of them.

## `dependabot-automerge.yml`

Enables GitHub auto-merge on Dependabot PRs for non-major updates, and approves
them. Majors are left alone for manual review — and so is any bump Dependabot
declines to classify.

That second clause is load-bearing. Dependabot omits its `update-type` for
indirect (transitive) dependencies, so an unclassified bump is not "not a major",
it is a bump nothing has judged. The workflow falls back to comparing the version
numbers itself and skips anything that will not parse: absence of information
means review, not merge. A repo that compiles its dependencies into shipped
output would otherwise publish an unreviewed major.

This tightened in `v2.3.0`. A repo that was auto-merging transitive bumps will
see some of them start waiting for review — that is the correction, not a
regression, and the run log says which test a skipped bump failed.

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

### Merge commits and release-please

Under `merge-method: merge`, what lands in the *body* of the merge commit is decided
by the repo's `merge_commit_message` setting, whose GitHub default is `PR_TITLE`.
Dependabot titles its PRs conventionally (`fix(deps): bump …`), so that default
copies a conventional commit into the merge commit — and a consumer running
release-please over merge commits parses the same bump twice: once from the branch
commit, once from the merge commit. The result is a duplicated changelog entry and a
version bump driven by a commit that was already counted.

Two details make it sharper than it looks. GitHub snapshots the merge commit message
when auto-merge is **enabled**, not when it fires, so the body freezes to whatever
the title was at that instant. And Dependabot rewrites the PR title on every rebase,
and again when a newer version supersedes an open PR — so the value being frozen is
one that changes underneath you.

This workflow therefore passes an explicit subject and an explicit body rather than
letting either default. The subject reproduces GitHub's own
(`Merge pull request #N from owner/branch`); the body is empty, which leaves the
branch's own conventional commit as release-please's sole input — the correct number
of times to count it. `merge-commit-body` can override the body, but the empty
default is the point: a latent duplicate-changelog bug should not be opt-out. A repo
whose Dependabot prefix is not release-driving is unaffected either way and loses
nothing, since the merge commit body was not carrying information the branch commit
lacks.

`squash` and `rebase` are deliberately left alone. A rebase produces no merge commit
at all, and under `squash` the squash commit **is** the only commit release-please
sees — overriding its subject would delete the conventional commit rather than
protect it.

The subject is not exposed as an input. A repo that has set
`merge_commit_title: PR_TITLE` loses that preference here, deliberately: a PR-title
subject reintroduces the same double-parse this section exists to prevent.

Both halves are needed. Passing only `--body` would leave the title-derived subject
in play, and omitting `--body` is not the same as passing an empty one — `gh` sends
the field only when the flag is supplied, so an omitted body falls back to
`merge_commit_message` and the title reappears.

One caveat: this covers base branches **without a merge queue**. A queue composes the
merge commit itself, and `gh` says as much — it warns that the merge strategy is the
queue's to set. A queue-enabled consumer should check its own changelog rather than
assume this settles it.

### Requirements

The calling repo needs [the Dependabot App](#the-dependabot-app), with its key in
the repo's `dependabot` environment. This job's token takes **Contents: write**,
**Pull requests: write**, and **Workflows: write**. The stub maps the two secrets
through by name.

`Workflows: write` is the load-bearing one. Most Dependabot PRs edit
`.github/workflows/*`, and GitHub refuses to enable auto-merge on such a PR
without it — a permission `GITHUB_TOKEN` cannot be granted through a workflow's
`permissions:` block. Using an App token also means the resulting merge commit
triggers CI, which a `GITHUB_TOKEN` push does not.

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
| `merge-commit-body` | *(empty)* | Body of the merge commit under `merge-method: merge`. Empty keeps Dependabot's conventional PR title out of the merge commit, so release-please counts a bump once. Ignored for `squash` and `rebase`. |
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
not accessible by integration`. The alerts read therefore authenticates as
[the Dependabot App](#the-dependabot-app).

Only that one step uses the App token; the issue it files and closes still runs as
`GITHUB_TOKEN`. So the token takes exactly one permission, **Dependabot alerts: read** —
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

## `dependabot-rebase.yml`

A one-at-a-time rebase train that lets `dependabot-automerge.yml` work under a
ruleset with **require branches to be up to date** (strict) turned on, so each
bump merges against the exact base its CI ran on and history stays semi-linear:
rebased branch commits, then a merge commit.

Without it, strict and auto-merge deadlock. GitHub's auto-merge never updates a
BEHIND branch, and Dependabot rebases only on conflicts, so the first bump merges
and every other open one sits BEHIND forever. A merge queue is the native fix,
but GitHub offers it only on organization-owned repositories.

Each run, acting as the caller's App:

1. Lists open Dependabot PRs into the default branch with auto-merge armed,
   oldest first. Unarmed ones — majors, unclassified bumps — stay with a human.
2. Does nothing if one is in flight: up to date, not conflicting, checks not
   failed. Rebasing a second PR now would only make it BEHIND again the moment
   the first lands.
3. Otherwise rebases the oldest BEHIND PR whose checks have not failed, with
   GraphQL `updatePullRequestBranch(updateMethod: REBASE)` guarded by
   `expectedHeadOid`.

A BEHIND PR that already failed is skipped rather than retried; being oldest, it
would otherwise be picked on every push and starve the rest. Conflicting (DIRTY)
PRs are left to Dependabot. Every run that does nothing says why in its log.

**Trade-off:** once the App rewrites a Dependabot branch, Dependabot no longer
updates that PR. On the happy path that is moot — it merges next. A PR that goes
red after the rebase needs fixing by hand, or closing so Dependabot reopens it.

**Consumer stub** — `.github/workflows/dependabot-rebase.yml`:

```yaml
name: Dependabot Rebase

on:
  push:
    branches: [main]
  pull_request_target:
    types: [auto_merge_enabled]
  workflow_run:
    workflows: [CI]
    types: [completed]

permissions:
  contents: read

jobs:
  rebase:
    # workflow_run fires for every CI run; only a failed Dependabot run can
    # free the train.
    if: >-
      github.event_name != 'workflow_run' ||
      (github.event.workflow_run.conclusion == 'failure' &&
       startsWith(github.event.workflow_run.head_branch, 'dependabot/'))
    uses: cacack/workflows/.github/workflows/dependabot-rebase.yml@<sha> # v2.5.0
    secrets:
      BOT_APP_ID: ${{ secrets.BOT_APP_ID }}
      BOT_PRIVATE_KEY: ${{ secrets.BOT_PRIVATE_KEY }}
```

Each trigger restarts the train at a different point: `push` when the in-flight
PR merges, `auto_merge_enabled` when a new bump is armed, and `workflow_run` when
the in-flight PR's CI fails. Name the repo's required-check workflow under
`workflows:`.

The `push` trigger relies on the merge being made by the App.
`dependabot-automerge.yml` arms auto-merge with the App token, so the merge
commit fans out events; a merge made with `GITHUB_TOKEN` would not, and the train
would stall until the next push.

### Requirements

[The Dependabot App](#the-dependabot-app), with its key in the repo's `dependabot`
environment. This job's token takes the same three permissions as
`dependabot-automerge.yml`. Workflows: write matters here too: the rebase rewrites
branches that touch `.github/workflows/*`. An App token is also what makes the
rebased branch's CI run at all, since a `GITHUB_TOKEN` push triggers nothing.

Strict must be on. With it off, GitHub never reports BEHIND and this workflow
never acts.

### Permissions

The stub grants `contents: read`. The job itself declares none: everything it
does is as the App.

### Inputs

| Input | Default | Notes |
|---|---|---|
| `egress-policy` | `block` | As for `dependabot-automerge.yml`. |
| `allowed-endpoints` | *(built-in list)* | As for `dependabot-automerge.yml`. |

## The Dependabot App

Every Dependabot workflow here authenticates as one GitHub App kept for that job
alone, and separate from release automation, so a problem with one cannot take
down the other. It holds:

| Permission | Used by |
|---|---|
| Contents: write | `dependabot-automerge.yml`, `dependabot-rebase.yml` |
| Pull requests: write | `dependabot-automerge.yml`, `dependabot-rebase.yml` |
| Workflows: write | `dependabot-automerge.yml`, `dependabot-rebase.yml` |
| Dependabot alerts: read | `dependabot-watch.yml` |
| Metadata: read | GitHub requires it |

Configure it with no webhook, no user authorization, and "Only on this account".
Install it on the repos that use these workflows, not on all repositories. Keep it
out of every ruleset's bypass list: rulesets then bind the App like anyone else, so
even a stolen token cannot push to the default branch or skip required checks.

### Keeping the key to these jobs

An App cannot be limited to Dependabot. Its permissions belong to the
installation, and anything holding the private key can mint a token with all of
them. So the protection is about who can reach the key.

A repo-level Actions secret is readable by any workflow in the repo, including one
added on a pushed branch. Instead, each caller keeps the key in an environment:

1. Create an environment named `dependabot`. Under *Deployment branches and tags*,
   allow only the default branch.
2. Add `BOT_APP_ID` and `BOT_PRIVATE_KEY` to that environment.
3. Delete the repo-level secrets with the same names.

Every job here that mints a token declares `environment: dependabot`, so only a
job running from the default branch can read the key. That fits all their
triggers: GitHub matches the policy against `GITHUB_REF`, which is the default
branch for `push`, `schedule`, and `workflow_run`, and the base branch for
`pull_request_target`. Reaching the key then means merging a workflow change
first.

The stub still maps both secrets by name. GitHub requires the caller to pass a
secret even when it exists only in the environment; one it does not pass
resolves to an empty string.

Each run records a deployment to the environment. That is the cost of the branch
policy: `deployment: false` would skip the record, but GitHub does not document
whether branch policies still apply then.

Each job also asks for only the permissions it uses, for the caller repo only, and
the token is revoked when the job ends. A token that leaks from one job is not the
whole App.

Environment secrets need a public repo on GitHub Free, or GitHub Pro for a
private one.

### Moving an existing repo

Nothing breaks before a repo is moved. A workflow that references a missing
environment creates it with no rules and no secrets, and the job then falls back
to the repo-level `BOT_*` secrets. The key is not protected until steps 1–3 above
are done.

One key serves every repo, so a rotation is: generate the new key, update
`BOT_PRIVATE_KEY` in every repo's environment, then delete the old key. A rotation
left half done fails every repo not yet updated, at the token step.

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
