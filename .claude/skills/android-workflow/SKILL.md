---
name: android-workflow
description: Entry point for Android developer workflow tasks on this repo (sync branches, pull from develop, AI PR review, release workflow, and future commands). Use when the user invokes /android-workflow or asks for a workflow menu.
---

# Android Developer Workflow Assistant

You are the router for this project's Android developer workflow assistant. This file's only job is to present a menu and dispatch to the selected command — it must never contain command-specific logic itself.

## What to do

1. Immediately call `AskUserQuestion` with one question asking which workflow action to run. Use exactly these options (no file read, no scanning — the menu is fixed here):
   - `Sync Branches` — "Safely merge one branch into another, with interactive conflict resolution." → `commands/sync-branches.md`
   - `Pull from Develop` — "Merge the latest develop into your current branch — a simplified Sync Branches with no branch selection needed." → `commands/pull-from-develop.md`
   - `AI PR Review` — "Review the diff between a source branch and a target branch (default: develop)." → `commands/ai-pr-review.md`
   - `Release Workflow` — "Prepare the develop → main release PR, or create the GitHub Release after it's merged." → asks one follow-up question, see below.
2. Once the user picks an option, read *only* that command's file and follow its instructions completely. Do not read or load any other command file.
3. If the user makes **no selection** — they press Skip, or the answer comes back as `[No preference]` or empty — the invocation ends here. Do not read any command file, do not run any command, do not pick a default. Say in one line that no workflow was selected and stop.

### Release Workflow follow-up

`Release Workflow` is a grouping label only — it exists because `AskUserQuestion` allows at most four options per question. The two commands underneath it are independent top-level workflow commands, not sub-steps of each other and not part of AI PR Review. When it's picked, ask one follow-up question with exactly these two options:

- `Dev → Main PR` — "Prepare the release on remote develop and open the develop → main PR. Does not merge it." → `commands/dev-to-main-pr.md`
- `Create Release` — "After the develop → main PR has been merged, tag main and publish the GitHub Release." → `commands/create-release.md`

Then read only the selected command file and follow it.

If the user makes no selection on this follow-up, the invocation ends here too. Do not fall back to the main menu, do not re-ask, and above all **do not pick whichever release command looks applicable given the repository's state** — that both commands exist for different moments in the release cycle does not license choosing between them on the developer's behalf. Say in one line that no release command was selected and stop, without running preflight or any `git`/`gh` command.

## Rules

- **A skipped question stops everything.** This applies to both menu questions above and to every question inside every command — no selection means the workflow ends, immediately and without changes. Never treat a `(Recommended)` label as a default for someone who declined to choose. Every command added to this menu in future inherits this rule; each command file states it in its own Safety Rules section, and the two Release Workflow commands carry the full version in `references/release-common.md` §0.
- Do not implement command logic here. If a command file's instructions are incomplete or a stub, follow them as written (e.g. report that the command isn't implemented yet) rather than improvising the missing behavior yourself.
- Do not skip the menu, even if the user's request seems to imply a specific action — always let them pick from the menu unless they've already named the exact action in their invocation (e.g. `/android-workflow sync-branches`, `/android-workflow create-release`), in which case you may route directly to the matching command without asking. A direct invocation of `dev-to-main-pr` or `create-release` skips the Release Workflow follow-up too.
- `commands/` is the menu-dispatch namespace: every file in it is a command, and every command has a menu entry. `references/` is not — files there are shared logic pulled in *by* a command, never dispatched to directly. `commands/dev-to-main-pr.md` and `commands/create-release.md` both read `references/release-common.md`; that is expected and is not a violation of the "read only that command's file" rule above.
- Adding a future command means adding one new file under `commands/` **and** one new bullet/option in the list above — this file is no longer a zero-edit router (there's no manifest to read), so expect to touch this file when growing the menu. The menu is capped at four options per question, so a fifth top-level command needs a grouping entry the way Release Workflow does.
