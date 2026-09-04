---
name: release-workflow
description: Direct entry point for the Android Workflow "Release Workflow" group — presents the choice between preparing the develop → main PR and creating the GitHub Release, then runs the selected one. Use only when the user explicitly invokes /release-workflow or asks to run the release workflow without naming a stage. If they already named one, use dev-to-main-pr or create-release instead. Equivalent to entering Android Workflow and picking Release Workflow.
---

# Direct Entry: Release Workflow

This is an **entry point, not an implementation.** It skips the Android Workflow main menu and starts at the Release Workflow choice. It defines no menu of its own.

## What to do

1. Read `../android-workflow/SKILL.md` — resolved against **this skill's own base directory**, which is given to you when the skill is invoked. This path is sibling-relative on purpose, never project-relative, so it resolves whether the skill tree is installed in a project's `.claude/skills/` or in the user-level `~/.claude/skills/`.
2. **Skip its main-menu question entirely** — the group is already chosen. Do not ask which workflow to run.
3. Ask exactly the question defined in that file's **`### Release Workflow follow-up`** section, using its options and their descriptions **verbatim**. That section is the single source of truth for this menu — do not reword, reorder, or reconstruct it from memory.
4. Then follow that file's **`### After any follow-up`** section for what happens next, including its rule that no selection ends the invocation without running preflight or any `git`/`gh` command.

## Rules

- This file contains no menu definition and no workflow logic, and never should. The options live in `SKILL.md`; the steps live in the command files.
- If this file and `SKILL.md` ever disagree, **`SKILL.md` wins.**
- Behavior must be identical whether reached from here or from the main menu. The only thing skipped is the main-menu question — never a confirmation, and never the choice between the two release stages, which are for different moments in the release cycle and must not be picked on the developer's behalf.
