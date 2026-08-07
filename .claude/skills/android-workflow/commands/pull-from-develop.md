# Command: Pull from Develop

Status: **active.**

This is a simplified version of Sync Branches: same safety and conflict-handling behavior, but with the branch-selection questions skipped because the choice is fixed — always pull `develop` into the current branch.

## Step 1 — Auto-detect branches (no questions)

Run `git branch --show-current` to detect the current branch.
- **Target branch** = the detected current branch.
- **Source branch** = `develop`, always.

Do not ask the developer to choose either branch.

## Step 2 — Validate

Show the same summary shape as Sync Branches' Step 3, pre-filled:

```
Target Branch:
<detected current branch>

Source Branch:
develop
```

Ask for explicit confirmation (`AskUserQuestion`, Yes/No) before making any git changes. If the user declines, stop here without touching git state.

## Step 3 — Reuse Sync Branches for everything else

On confirmation, open `commands/sync-branches.md` in this same skill and continue with **Step 4 through Step 7 exactly as written there**, substituting:
- `<target>` = the detected current branch
- `<source>` = `develop`

Do not re-implement or duplicate the fetch/merge/conflict-detection/intelligent-resolution logic here — follow `sync-branches.md`'s Steps 4–7 and its Safety Rules section verbatim. This file only exists to skip Sync Branches' Steps 1–3 (branch selection) with a fixed choice.
