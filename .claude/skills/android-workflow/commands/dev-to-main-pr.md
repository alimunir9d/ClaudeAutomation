# Command: Dev → Main PR

Status: **active.**

You are acting as an experienced Android release manager preparing a production release. This is the first half of the Release Workflow: it prepares the release on the remote repository and opens the `develop` → `main` Pull Request, then stops.

Two properties define this command, and both must be stated to the developer up front:

1. **It operates entirely against the remote.** No branch is checked out, nothing is merged locally, no local commit is created, and the developer's working tree is never touched. The remote is the source of truth — local `develop` and local `main` are ignored completely, because in this clone they are routinely stale.
2. **It never merges the PR.** Creating the PR is where this command ends. A human reviews and merges it.

All shared behavior lives in `references/release-common.md`, referenced below as **§N**, and the tree-wide git rules live in `references/git-common.md`, referenced as **§GN**. Read both files before starting. Do not restate or re-implement their logic here.

Follow the steps below in order — do not skip or reorder them.

## Step 0 — Preflight

Run **§2 — Preflight and GitHub access gate**.

Tell the developer plainly, in one line, that this command will not touch their current branch or working tree, and name the branch they are on so it's concrete. Do not gate on a dirty working tree — uncommitted local work is invisible to this workflow and cannot be affected by it.

If `gh` is missing or unauthenticated, §2 sends you to **§8 — Manual fallback**. Take it; do not improvise a local-git path.

## Step 1 — Determine Release Version

Run **§3 — Version detection ladder** against `origin/develop`.

Show the developer what was detected, which source it came from, and the proposed version. Confirm before continuing. If the ladder finds nothing confident, ask for the version outright — that is the expected path on a first release.

Store the confirmed value as the **release version**. It is used verbatim in the README release line, the commit message, the PR title, and the PR body, and later by Create Release for the tag.

## Step 2 — Release Notes

Ask via `AskUserQuestion`:

> Would you like to provide the release notes, or should I generate them from the changes between develop and main?

1. `Generate release notes automatically (Recommended)`
2. `Provide release notes`

**If Generate:** run **§4 — Release-notes generation**, then **§5 — Release-notes confirmation**.

**If Provide:** ask a free-text follow-up for the notes, then still run **§5** — show them back and confirm. Supplied notes get the same confirmation loop as generated ones, so what lands in the PR is unambiguous.

Store the approved text as the **confirmed release notes**. Nothing downstream may use any other wording.

## Step 3 — Verify remote develop is ahead of main

There is no "prepare develop" step in the local sense — nothing is pulled into a local branch. "Latest" here means the fetch from §2 plus a check that there is actually something to release:

1. `git log --oneline origin/main..origin/develop` — the commits this release would deliver.
2. `git diff --stat origin/main...origin/develop` — the file footprint.

If the range is empty, remote `develop` has nothing `main` doesn't already have. Say so and stop — do not open an empty PR.

Report the commit count and changed-file count. Do not modify `main` in any way at this or any later step.

## Step 4 — Update README

Run **§6 — Remote README write** with the confirmed release version.

Show the exact before/after of the release line, then confirm before writing. If `README.md` does not exist on remote `develop`, §6 creates a minimal one — say so explicitly, since creating a file is a bigger change than editing a line.

## Step 5 — Release-preparation commit

There is no separate commit step. The Contents API call in §6 **is** the commit — it creates `chore(release): prepare release <version>` directly on remote `develop`.

Report the returned commit sha, then `git fetch origin --prune --tags` (§G2) and confirm `origin/develop` now points at it. If the API call conflicted because `develop` moved, follow §6's conflict handling — re-read, rebuild, show the developer what changed. Never retry by dropping the `sha` guard.

## Step 6 — Create the Dev → Main PR

First guard against duplicates: list open `develop` → `main` PRs per **§7**. If one already exists, do not open a second — report its URL and number and ask via `AskUserQuestion` whether to update that PR's body with this release's version and notes, or to stop and let the developer handle it.

Otherwise create the PR per **§7**:

- **Source (head):** `develop`
- **Target (base):** `main`
- **Title:** `Release <version>`
- **Body**, written to a scratchpad file and passed with `--body-file`:

  ```markdown
  ## Release <version>

  ### Release Notes

  <confirmed release notes, verbatim>

  ### Summary of Changes

  <commit count> commits, <file count> files changed between main and develop.
  <short prose summary grounded in the Step 3 material>
  ```

The release notes in the PR body must match the confirmed notes **exactly** — do not re-summarize, re-wrap, or tighten them for the PR.

Confirm with the developer before creating the PR. This is a visible, outward-facing action.

If they skip that confirmation, stop per **§0** — do not create the PR. Note that by this point the Step 5 README commit **has already landed on remote `develop`**: report the commit sha and say plainly that the release preparation is on `develop` but no PR was opened, so the developer knows the exact state to pick up from.

## Step 7 — Report and stop

Report, plainly:

- **PR created** — URL and PR number
- **Source branch:** `develop`
- **Target branch:** `main`
- **Release version:** `<version>`
- **Release notes used:** the confirmed notes, in full

Then state in one line that the PR was **not** merged and that the human developer must review and merge it manually.

Stop here. Do not offer to merge. Do not chain into Create Release — that is a separate command the developer runs after the merge lands.

## Safety Rules (apply throughout)

All of **§9 — Shared safety rules** applies. The ones this command is most likely to be tempted to break:

- **A skipped question ends the command** (§0). No selection is not permission to proceed with the recommended option or the version you detected. Stop, and disclose anything already written to the remote.
- **Never merge the PR**, and never offer to. Creating it is the end of the job.
- **Never checkout, merge, or commit locally.** The release-preparation commit is created on the remote through the API. The developer's branch and working tree must be exactly as they were when the command started (§1).
- **Never read local `develop` or local `main`** (`git-common.md` §G1). Every ref in this command is an `origin/...` ref, after a fetch that succeeded — a ref resolving is not evidence it is fresh (§G2). A bare `develop` anywhere here is a bug.
- **Never modify any file other than `README.md`** — no `versionName` bump, no `gradle.properties`, no CHANGELOG.
- **Never invent release notes.** If the diff doesn't support a line, the line doesn't ship.
- **Never proceed without `gh`.** If it isn't available, take the §8 fallback and say clearly that nothing was created.
