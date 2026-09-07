---
name: sync-strings-full
description: Direct entry point for the Android Workflow "Sync Strings — Full" command — a complete audit comparing every English string key against every supported language file, prioritizing accuracy over token cost. Use when the user explicitly invokes /sync-strings-full or names this command. Equivalent to Android Workflow → Sync Strings → Sync Strings — Full, with no mode question.
---

# Direct Entry: Sync Strings — Full

This is an **entry point, not an implementation.** It exists so the command can be run without walking the Android Workflow menu. The command itself is defined in exactly one place.

## What to do

1. Read `../android-workflow/commands/sync-strings-full.md` — resolved against **this skill's own base directory**, which is given to you when the skill is invoked — and follow it completely, from its first step to its last.
2. Treat `../android-workflow/` as the command's **base directory**. Every relative path the command names — notably `references/strings-common.md`, which it cites throughout as **§N** — resolves from there.

Paths here are sibling-relative on purpose, never project-relative. The skill tree is installed as a unit, so `../android-workflow/` resolves correctly whether that unit sits in a project's `.claude/skills/` or in the user-level `~/.claude/skills/`. Never rewrite these as `.claude/skills/...` — that would bind the skill to one project.
3. **Skip nothing.** Direct invocation skips menu navigation and nothing else. Every question, preflight check, confirmation and step inside the command runs in order, exactly as it would via the menu.

## Rules

- This file contains no workflow logic and never should. Do not reimplement, summarize, adapt, or shortcut anything in the command file.
- If this file and the command file ever disagree, **the command file wins.**
- Behavior must be identical whether reached from here or from the menu. Launching directly is not a reason to act differently.
