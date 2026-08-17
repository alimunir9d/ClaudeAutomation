---
name: sync-strings-fast
description: Direct entry point for the Android Workflow "Sync Strings — Fast" command — token-efficient incremental sync of Android string translations, scanning upward from the bottom of the English strings.xml and stopping at the synchronization boundary. Use when the user explicitly invokes /sync-strings-fast or names this command. Equivalent to Android Workflow → Sync Strings → Sync Strings — Fast, with no mode question.
---

# Direct Entry: Sync Strings — Fast

This is an **entry point, not an implementation.** It exists so the command can be run without walking the Android Workflow menu. The command itself is defined in exactly one place.

## What to do

1. Read `.claude/skills/android-workflow/commands/sync-strings-fast.md` and follow it completely, from its first step to its last.
2. Treat `.claude/skills/android-workflow/` as the command's **base directory**. Every relative path the command names — notably `references/strings-common.md`, which it cites throughout as **§N** — resolves from there, not from this skill's directory.
3. **Skip nothing.** Direct invocation skips menu navigation and nothing else. Every question, preflight check, confirmation and step inside the command runs in order, exactly as it would via the menu.

## Rules

- This file contains no workflow logic and never should. Do not reimplement, summarize, adapt, or shortcut anything in the command file.
- If this file and the command file ever disagree, **the command file wins.**
- Behavior must be identical whether reached from here or from the menu. Launching directly is not a reason to act differently.
