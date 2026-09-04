# Command: Pull from Develop

Status: **active.**

This is a simplified version of Sync Branches: same safety and conflict-handling behavior, but with the branch-selection questions skipped because the choice is fixed — always pull the latest `develop` into the current branch.

Shared git behavior lives in `references/git-common.md`, referenced below as **§GN**. Read that file before starting. Everything it governs — the remote refresh, **§G3 (Gate A)** when the remote can't be reached, and the §G6 divergence prompt — is inherited through Sync Branches' Step 4 rather than restated here.

## Step 1 — Auto-detect branches (no questions)

First confirm this is a git repository: `git rev-parse --is-inside-work-tree`. If it is not, **stop cleanly** — say so plainly and change nothing. This command is a branch tool, so a git repository is the only requirement; it does not need an Android project.

Then run `git branch --show-current` to detect the current branch.
- **Target branch** = the detected current branch. This is a **local** branch — it is where the merge is performed.
- **Source branch** = `develop`, always — resolved per **§G5** to **`origin/develop`**, which is the source of truth for what gets merged. The local `develop` branch is a working copy and is not assumed to match it; in this clone it routinely doesn't.

Do not ask the developer to choose either branch.

## Step 2 — Validate

Show the same summary shape as Sync Branches' Step 3, pre-filled — naming the ref that will actually be merged, not the bare name:

```
Target Branch (merge happens here, local):
<detected current branch>

Source Branch (source of truth, remote):
origin/develop
```

If the detected current branch **is** `develop`, this becomes a pull of `origin/develop` into local `develop`. Say so explicitly before asking, per Sync Branches' Step 3 same-branch rule — it is a real operation, not the no-op it used to be.

Ask for explicit confirmation (`AskUserQuestion`, Yes/No) before making any git changes. If the user declines, stop here without touching git state.

The same applies to no answer at all: if they press Skip, or the answer comes back as `[No preference]` or empty, stop here without touching git state. A skipped confirmation is not a Yes.

## Step 3 — Reuse Sync Branches for everything else

On confirmation, open `commands/sync-branches.md` in this same skill and continue with **Step 4 through Step 8 exactly as written there**, substituting:
- `<target>` = the detected current branch (local — the merge destination)
- `<source>` = `develop`, which Step 4 resolves to `origin/develop`

Sync Branches' Step 4 is where the remote refresh, Gate A, the ref verification and the divergence prompt all live. Run it as written — including Step 4.4's question if local `develop` turns out to have commits the remote doesn't. That the branches were not chosen interactively is no reason to skip a question about which state to merge.

Do not re-implement or duplicate the fetch/merge/conflict-detection/intelligent-resolution/finalization logic here — follow `sync-branches.md`'s Steps 4–8 and its Safety Rules section verbatim, including its skipped-question rule and the mid-merge handling that rule requires. Step 8 is not optional: this command ends only when the merge is committed and the developer knows which branch they're on. This file only exists to skip Sync Branches' Steps 1–3 (branch selection) with a fixed choice.
