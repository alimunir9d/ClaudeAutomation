---
name: design-xml
description: Direct entry point for the Android Workflow "Designs → XML" group — presents the choice between ConstraintLayout Only and Optimal, then runs the selected mode to implement a screen design as an Android XML layout. Use when the user explicitly invokes /design-xml or asks to implement a design as XML without naming a mode. If they already named one, use design-xml-constraint or design-xml-optimal instead. Equivalent to entering Android Workflow and picking Designs → XML.
---

# Direct Entry: Designs — XML

This is an **entry point, not an implementation.** It skips the Android Workflow main menu and the Designs question, and starts at the XML mode choice. It defines no menu of its own.

## What to do

1. Read `../android-workflow/SKILL.md` — resolved against **this skill's own base directory**, which is given to you when the skill is invoked. This path is sibling-relative on purpose, never project-relative, so it resolves whether the skill tree is installed in a project's `.claude/skills/` or in the user-level `~/.claude/skills/`.
2. **Skip its main-menu question and its Designs follow-up entirely** — both levels are already chosen. Do not ask which workflow to run, and do not ask XML or Compose.
3. Ask exactly the question defined in that file's **`#### Designs → XML follow-up`** section, using its options and their descriptions **verbatim**. That section is the single source of truth for this menu — do not reword, reorder, or reconstruct it from memory.
4. Then follow that file's **`### After any follow-up`** section for what happens next, including its rule that no selection ends the invocation. A skipped question here stops the command; it does not fall back to a mode.

## Rules

- This file contains no menu definition and no workflow logic, and never should. The options live in `SKILL.md`; the steps live in the command files.
- If this file and `SKILL.md` ever disagree, **`SKILL.md` wins.**
- Behavior must be identical whether reached from here or from the main menu. The only things skipped are the two menu questions above.
