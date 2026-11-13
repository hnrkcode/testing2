# SemVer releases and deployments

Both deployments are manual and currently print the version and commit only.
Publishing a release refreshes the version dropdown; it does not deploy.

| Workflow | Purpose |
| --- | --- |
| Deploy dev | Deploy the latest `main` commit at checkout time. |
| Deploy prod | Select a published version and deploy its exact commit. |
| Sync release versions | Refresh the prod dropdown when releases change, or manually. |

Use a branch for each release line and an immutable tag for each version:

| Branch | Versions |
| --- | --- |
| `main` | Ongoing development, including production fixes |
| `release/0.1` | `v0.1.0`, `v0.1.1`, `v0.1.2` |
| `release/0.2` | `v0.2.0`, `v0.2.1` |

## Setup

1. Copy the three `.github/workflows/*.yml` files to your repository's default
   branch, **`main`**, keeping their filenames. They are self-contained: no helper
   scripts, dependencies to install, or application-specific files are required.
   The inline Bash uses tools available on GitHub's Ubuntu runners.
2. Configure environments **`dev`** and **`prod`** under Settings → Environments.
   If restricting deployment branches, allow **`main`** in both. Configure prod
   reviewers there if desired.
3. Allow merge commits into `main`. Release-to-main PRs must use **Create a merge
   commit** so the production ancestry check can recognize the release history.
4. Enable **release immutability** under Settings → General → Releases. This
   locks future releases' tags and assets after publication.
5. Configure the dropdown updater below, then run **Sync release versions** once.

### Why the dropdown needs a token

GitHub's `choice` inputs store a fixed list in YAML; they cannot query releases
when the form opens. The updater reads all published stable releases, sorts
versions numerically, and commits just the generated options block in
`.github/workflows/deploy-prod.yml` to `main`. Unchanged lists create no commit.
Both `0.1.0` and `v0.1.0` are supported and kept as distinct tags.

Create a fine-grained token restricted to this repository with **Contents: Read
and write** and **Workflows: Read and write**. Store it as the repository Actions
secret **`RELEASE_OPTIONS_TOKEN`**. Only the updater uses this token; deployment
jobs use the built-in read-only token. The updater must be permitted to commit
to `main`. It fails if branch protection rejects the write or the workflow file
changes concurrently; it never forces an update.

If your rules require every workflow change to go through a PR, maintain the
marked options block through PRs instead of configuring the updater. This avoids
weakening branch protection for a dropdown.

Run **Actions → Sync release versions → Run workflow** to populate the list.
Release events refresh it when the sync workflow exists at the tagged revision.
For older release branches without it, or a failed refresh, run sync manually
from `main`. Wait for it to finish, then reopen the Deploy prod form. An empty
list shows only “Select a release” and cannot deploy.

See GitHub's [choice inputs](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#onworkflow_dispatchinputs),
[workflow-file permissions](https://docs.github.com/en/rest/repos/contents#create-or-update-file-contents),
and [immutable releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases).

## First release: 0.1.0

Create the release branch from a tested commit already in `main`. Replace
`CHOSEN_MAIN_SHA` with that commit's full SHA:

```sh
git fetch origin
git switch -c release/0.1 CHOSEN_MAIN_SHA
git push -u origin release/0.1
```

1. Open **Releases → Draft a new release**. Create tag **`v0.1.0`** targeting
   **`release/0.1`**, add notes, review the commit, and publish without marking it
   as a prerelease.
2. Wait for **Sync release versions**, or run it manually if needed.
3. Open **Actions → Deploy prod → Run workflow**, select **`main`**, choose
   **`v0.1.0`** from the version dropdown, and run it.

## Hotfix: 0.1.1

```sh
git fetch origin
git switch -c hotfix/fix-login origin/release/0.1
# Make and test the fix, then commit and push it.
```

1. Merge the hotfix PR into **`release/0.1`**.
2. Merge a PR from **`release/0.1` into `main`** using **Create a merge commit**.
3. Publish **`v0.1.1`** targeting **`release/0.1`**, refresh the dropdown, and
   manually deploy that version.

Merge **release → main**. Do not use **Update branch** on that PR or merge/rebase
`main` into the release: it would include newer features. If conflicts or branch
rules require updating the PR, create a temporary branch from `main`, merge the
release into it, resolve and test there, then merge that PR into `main` with a
merge commit. Keep active release branches after merging their PRs.

This flow needs no cherry-picking when fixes start on the release. A fix already
written on newer features may need a backport. The ancestry check proves shared
history; tests must still catch bad conflict resolutions or reverted fixes.

## Next release and rollback

For **`v0.2.0`**, create **`release/0.2`** from a newer tested `main` commit and
repeat the first-release steps. Keep each supported release line separate.

To retry or roll back, manually deploy an existing version from the dropdown.
Never move a published tag or reuse a version. Drafts, prereleases, build
metadata, and versions with leading zeroes are excluded. The workflow rechecks
the release at deployment time, so a stale dropdown cannot bypass validation.

## Deployment guarantees

Prod validates the release, resolves its tag to a commit, and requires that
commit in `main`. Then a separate job waits for any prod approval and checks out
that exact SHA. It does not follow a moving release branch. Summaries record the
actual tag and SHA; GitHub's manual-run metadata refers to the triggering `main`
revision. A published release does not mean its deployment succeeded.

Dev and prod have separate concurrency groups. Runs for an environment cannot
overlap or cancel an active deployment. GitHub can replace an older pending run
when a newer run is queued; it is not a FIFO deployment queue.

When changing workflows, start a new manual run from `main`; rerunning an old run
uses its original workflow revision. Older tagged revisions containing the
previous automatic prod trigger can still run it when published. Update that
workflow on maintenance branches before cutting new tags.

## Moving to a frontend

Replace the dummy steps with install, test, build, and deploy commands using the
checked-out source. Configure environment secrets and suitable job timeouts.
Keep the production build with its version, SHA, and build-time configuration,
and deploy that saved artifact for rollback. This demo only checks out source;
it does not yet store application builds.

If attaching build artifacts to immutable GitHub Releases, upload them while the
release is a draft, then publish. Published assets cannot be changed.
