---
name: android-workflow
description: Entry point for Android developer workflow tasks on this repo (sync branches, pull from develop, AI PR review, and future commands). Use when the user invokes /android-workflow or asks for a workflow menu.
---

# Android Developer Workflow Assistant

You are the router for this project's Android developer workflow assistant. This file's only job is to present a menu and dispatch to the selected command — it must never contain command-specific logic itself.

## What to do

1. Immediately call `AskUserQuestion` with one question asking which workflow action to run. Use exactly these options (no file read, no scanning — the menu is fixed here):
   - `Sync Branches` — "Safely merge one branch into another, with interactive conflict resolution." → `commands/sync-branches.md`
   - `Pull from Develop` — "Merge the latest develop into your current branch — a simplified Sync Branches with no branch selection needed." → `commands/pull-from-develop.md`
   - `AI PR Review` — "Review the diff between a source branch and a target branch (default: develop)." → `commands/ai-pr-review.md`
2. Once the user picks an option, read *only* that command's file and follow its instructions completely. Do not read or load any other command file.

## Rules

- Do not implement command logic here. If a command file's instructions are incomplete or a stub, follow them as written (e.g. report that the command isn't implemented yet) rather than improvising the missing behavior yourself.
- Do not skip the menu, even if the user's request seems to imply a specific action — always let them pick from the menu unless they've already named the exact action in their invocation (e.g. `/android-workflow sync-branches`), in which case you may route directly to the matching command without asking.
- Adding a future command means adding one new file under `commands/` **and** one new bullet/option in the list above — this file is no longer a zero-edit router (there's no manifest to read), so expect to touch this file when growing the menu.
