---
name: branch-operations
description: Direct entry point for the Android Workflow "Branch Operations" group — presents the choice between Sync Branches and Pull from Develop, then runs the selected one. Use only when the user explicitly invokes /branch-operations or asks for the branch workflow without naming which one. If they already named one, use sync-branches or pull-from-develop instead. Equivalent to entering Android Workflow and picking Branch Operations.
---

# Direct Entry: Branch Operations

This is an **entry point, not an implementation.** It skips the Android Workflow main menu and starts at the Branch Operations choice. It defines no menu of its own.

## What to do

1. Read `.claude/skills/android-workflow/SKILL.md`.
2. **Skip its main-menu question entirely** — the group is already chosen. Do not ask which workflow to run.
3. Ask exactly the question defined in that file's **`### Branch Operations follow-up`** section, using its options and their descriptions **verbatim**. That section is the single source of truth for this menu — do not reword, reorder, or reconstruct it from memory.
4. Then follow that file's **`### After any follow-up`** section for what happens next, including its rule that no selection ends the invocation.

## Rules

- This file contains no menu definition and no workflow logic, and never should. The options live in `SKILL.md`; the steps live in the command files.
- If this file and `SKILL.md` ever disagree, **`SKILL.md` wins.**
- Behavior must be identical whether reached from here or from the main menu. The only thing skipped is the main-menu question.
