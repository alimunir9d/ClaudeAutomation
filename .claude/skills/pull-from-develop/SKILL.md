---
name: pull-from-develop
description: Direct entry point for the Android Workflow "Pull from Develop" command — merge the latest develop into the current branch, a simplified Sync Branches with no branch selection. Use only when the user explicitly invokes /pull-from-develop or names this command. Equivalent to Android Workflow → Branch Operations → Pull from Develop.
---

# Direct Entry: Pull from Develop

This is an **entry point, not an implementation.** It exists so the command can be run without walking the Android Workflow menu. The command itself is defined in exactly one place.

## What to do

1. Read `.claude/skills/android-workflow/commands/pull-from-develop.md` and follow it completely, from its first step to its last.
2. Treat `.claude/skills/android-workflow/` as the command's **base directory**. Every relative path the command names resolves from there, not from this skill's directory — including its delegation to `commands/sync-branches.md` for Steps 4–8, which is part of the command and must be followed as written.
3. **Skip nothing.** Direct invocation skips menu navigation and nothing else. Every question, validation, confirmation and step inside the command runs in order, exactly as it would via the menu.

## Rules

- This file contains no workflow logic and never should. Do not reimplement, summarize, adapt, or shortcut anything in the command file.
- If this file and the command file ever disagree, **the command file wins.**
- Behavior must be identical whether reached from here or from the menu. Launching directly is not a reason to act differently.
