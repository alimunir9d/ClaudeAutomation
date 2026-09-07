---
name: android-workflow
description: Entry point for Android developer workflow tasks on this repo (sync branches, pull from develop, AI PR review, release workflow, sync string translations, and future commands). Use when the user invokes /android-workflow or asks for a workflow menu.
---

# Android Developer Workflow Assistant

You are the router for this project's Android developer workflow assistant. This file's only job is to present a menu and dispatch to the selected command — it must never contain command-specific logic itself.

## What to do

1. Immediately call `AskUserQuestion` with one question asking which workflow action to run. Use exactly these options (no file read, no scanning — the menu is fixed here):
   - `Branch Operations` — "Merge one branch into another, or pull the latest develop into your current branch." → asks one follow-up question, see below.
   - `AI PR Review` — "Review the diff between a source branch and a target branch (default: develop)." → `commands/ai-pr-review.md`
   - `Release Workflow` — "Prepare the develop → main release PR, or create the GitHub Release after it's merged." → asks one follow-up question, see below.
   - `Sync Strings` — "Synchronize Android string translations across all supported languages, using English as the source of truth." → asks one follow-up question, see below.
2. Once the user picks an option, read *only* that command's file and follow its instructions completely. Do not read or load any other command file.
3. If the user makes **no selection** — they press Skip, or the answer comes back as `[No preference]` or empty — the invocation ends here. Do not read any command file, do not run any command, do not pick a default. Say in one line that no workflow was selected and stop.

Three of the four entries are grouping labels. They exist because `AskUserQuestion` allows at most four options per question, not because the commands underneath them are sub-steps of each other. Every command below is an independent top-level workflow.

### Branch Operations follow-up

- `Sync Branches` — "Safely merge one branch into another, with interactive conflict resolution." → `commands/sync-branches.md`
- `Pull from Develop` — "Merge the latest develop into your current branch — a simplified Sync Branches with no branch selection needed." → `commands/pull-from-develop.md`

### Release Workflow follow-up

- `Dev → Main PR` — "Prepare the release on remote develop and open the develop → main PR. Does not merge it." → `commands/dev-to-main-pr.md`
- `Create Release` — "After the develop → main PR has been merged, tag main and publish the GitHub Release." → `commands/create-release.md`

### Sync Strings follow-up

- `Sync Strings — Full` — "Complete audit: compare every English key against every language file. Accuracy over token cost." → `commands/sync-strings-full.md`
- `Sync Strings — Fast` — "Token-efficient incremental sync. Scans upward from the bottom of the English file and stops at the synchronization boundary." → `commands/sync-strings-fast.md`

### After any follow-up

Read only the selected command file and follow it.

If the user makes no selection on a follow-up, the invocation ends there too. Do not fall back to the main menu, do not re-ask, and above all **do not pick whichever command looks applicable given the project's state** — that two commands exist for different situations does not license choosing between them on the developer's behalf. Say in one line that no command was selected and stop, without running preflight or any `git`, `gh`, or file-reading step.

## Rules

- **A skipped question stops everything.** This applies to the main menu, to all three follow-ups above, and to every question inside every command — no selection means the workflow ends, immediately and without changes. Never treat a `(Recommended)` label as a default for someone who declined to choose. Every command added to this menu in future inherits this rule; each command file states it in its own Safety Rules section, and the reference files carry the full version in `references/release-common.md` §0 and `references/strings-common.md` §0.
- Do not implement command logic here. If a command file's instructions are incomplete or a stub, follow them as written (e.g. report that the command isn't implemented yet) rather than improvising the missing behavior yourself.
- Do not skip the menu, even if the user's request seems to imply a specific action — always let them pick from the menu unless they've already named the exact action in their invocation (e.g. `/android-workflow sync-branches`, `/android-workflow create-release`, `/android-workflow sync-strings-fast`), in which case you may route directly to the matching command without asking. A direct invocation of any command skips its grouping follow-up too.
- `commands/` is the menu-dispatch namespace: every file in it is a command, and every command has a menu entry. `references/` is not — files there are shared logic pulled in *by* a command, never dispatched to directly. A command reading its references is expected and is not a violation of the "read only that command's file" rule above. Which command reads what:
  - `references/git-common.md` — every command that reads **branch contents**: Sync Branches, Pull from Develop, AI PR Review, Dev → Main PR, Create Release. Its sections are cited as **§G0**–**§G6**; the `G` prefix exists so a command reading two references never has to guess which file a bare `§2` came from.
  - `references/release-common.md` — the two Release Workflow commands, cited as **§N**.
  - `references/strings-common.md` — the two Sync Strings commands, cited as **§N**. These are **purely local** commands: they read `git-common.md` not at all, must not fetch or read remote refs, and must never ask about remote access.
- **Remote branches are the source of truth for branch state — and only for branch state.** Any command that compares, reviews, merges or releases branch contents takes that content from `origin/...` refs and, if the remote can't be reached, tells the developer and asks rather than silently substituting a local branch (`references/git-common.md` §G1–§G4). This does not make every command network-dependent: a command that only reads and writes files in the working project stays local, and `git-common.md` §G0 names which commands are which.
- Adding a future command means adding one new file under `commands/` **and** one new bullet/option in the lists above — this file is no longer a zero-edit router (there's no manifest to read), so expect to touch this file when growing the menu. Each question is capped at four options, so a new top-level command either takes a free slot or joins/creates a grouping entry.
