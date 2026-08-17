---
name: sync-branches
description: Direct entry point for the Android Workflow "Sync Branches" command — safely merge one branch into another with interactive conflict resolution. Use only when the user explicitly invokes /sync-branches or names this command; ordinary git merge questions do not belong here. Equivalent to Android Workflow → Branch Operations → Sync Branches.
---

# Direct Entry: Sync Branches

This is an **entry point, not an implementation.** It exists so the command can be run without walking the Android Workflow menu. The command itself is defined in exactly one place.

## What to do

1. Read `.claude/skills/android-workflow/commands/sync-branches.md` and follow it completely, from its first step to its last.
2. Treat `.claude/skills/android-workflow/` as the command's **base directory**. Every relative path the command names resolves from there, not from this skill's directory.
3. **Skip nothing.** Direct invocation skips menu navigation and nothing else. Every question, validation, confirmation and step inside the command runs in order, exactly as it would via the menu — including its Safety Rules, and its requirement never to leave the repository mid-merge.

## Rules

- This file contains no workflow logic and never should. Do not reimplement, summarize, adapt, or shortcut anything in the command file.
- If this file and the command file ever disagree, **the command file wins.**
- Behavior must be identical whether reached from here or from the menu. Launching directly is not a reason to act differently.
