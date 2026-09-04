---
name: sync-strings
description: Direct entry point for the Android Workflow "Sync Strings" group — presents the choice between Full and Fast mode, then runs the selected one. Use when the user explicitly invokes /sync-strings or asks to sync string translations without naming a mode. If they already named a mode, use sync-strings-full or sync-strings-fast instead. Equivalent to entering Android Workflow and picking Sync Strings.
---

# Direct Entry: Sync Strings

This is an **entry point, not an implementation.** It skips the Android Workflow main menu and starts at the Sync Strings choice. It defines no menu of its own.

## What to do

1. Read `../android-workflow/SKILL.md` — resolved against **this skill's own base directory**, which is given to you when the skill is invoked. This path is sibling-relative on purpose, never project-relative, so it resolves whether the skill tree is installed in a project's `.claude/skills/` or in the user-level `~/.claude/skills/`.
2. **Skip its main-menu question entirely** — the group is already chosen. Do not ask which workflow to run.
3. Ask exactly the question defined in that file's **`### Sync Strings follow-up`** section, using its options and their descriptions **verbatim**. That section is the single source of truth for this menu — do not reword, reorder, or reconstruct it from memory.
4. Then follow that file's **`### After any follow-up`** section for what happens next, including its rule that no selection ends the invocation.

## Rules

- This file contains no menu definition and no workflow logic, and never should. The options live in `SKILL.md`; the steps live in the command files.
- If this file and `SKILL.md` ever disagree, **`SKILL.md` wins.**
- Behavior must be identical whether reached from here or from the main menu. The only thing skipped is the main-menu question.
