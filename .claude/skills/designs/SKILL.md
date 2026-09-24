---
name: designs
description: Direct entry point for the Android Workflow "Designs" group — implement a screen design from a screenshot, a Figma frame, and/or a written description, into an existing Android project's own conventions. Presents the choice between XML and Compose, then runs the selected mode. Use when the user explicitly invokes /designs or asks to implement a design without naming a mode. If they already named one, use design-xml-constraint, design-xml-optimal, design-compose, or design-xml instead. Equivalent to entering Android Workflow and picking Designs.
---

# Direct Entry: Designs

This is an **entry point, not an implementation.** It skips the Android Workflow main menu and starts at the Designs choice. It defines no menu of its own.

## What to do

1. Read `../android-workflow/SKILL.md` — resolved against **this skill's own base directory**, which is given to you when the skill is invoked. This path is sibling-relative on purpose, never project-relative, so it resolves whether the skill tree is installed in a project's `.claude/skills/` or in the user-level `~/.claude/skills/`.
2. **Skip its main-menu question entirely** — the group is already chosen. Do not ask which workflow to run.
3. Ask exactly the question defined in that file's **`### Designs follow-up`** section, using its options and their descriptions **verbatim**. That section is the single source of truth for this menu — do not reword, reorder, or reconstruct it from memory.
4. If the answer is `XML`, ask the question defined in that file's **`#### Designs → XML follow-up`** section, again verbatim. Designs is the tree's one two-level group, so reaching a command from here can take two questions.
5. Then follow that file's **`### After any follow-up`** section for what happens next, including its rule that no selection ends the invocation — at *either* level. A skipped XML follow-up stops the command; it does not fall back to the Designs question or pick a mode.

## Rules

- This file contains no menu definition and no workflow logic, and never should. The options live in `SKILL.md`; the steps live in the command files.
- If this file and `SKILL.md` ever disagree, **`SKILL.md` wins.**
- Behavior must be identical whether reached from here or from the main menu. The only thing skipped is the main-menu question.
