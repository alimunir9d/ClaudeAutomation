---
name: design-xml-optimal
description: Direct entry point for the Android Workflow "Designs → XML → Optimal" command — implement a screen design as an XML layout with the minimum reasonable ViewGroup hierarchy, ConstraintLayout-rooted with other containers only where they genuinely pay for themselves, from a screenshot, a Figma frame, and/or a written description. Use when the user explicitly invokes /design-xml-optimal or names this command. Equivalent to Android Workflow → Designs → XML → Optimal, with no mode questions.
---

# Direct Entry: Designs — XML, Optimal

This is an **entry point, not an implementation.** It exists so the command can be run without walking the Android Workflow menu. The command itself is defined in exactly one place.

## What to do

1. Read `../android-workflow/commands/design-xml-optimal.md` — resolved against **this skill's own base directory**, which is given to you when the skill is invoked — and follow it completely, from its first step to its last.
2. Treat `../android-workflow/` as the command's **base directory**. Every relative path the command names — notably `references/design-common.md`, cited as **§DN**, and `references/design-xml-common.md`, cited as **§XN** — resolves from there.

Paths here are sibling-relative on purpose, never project-relative. The skill tree is installed as a unit, so `../android-workflow/` resolves correctly whether that unit sits in a project's `.claude/skills/` or in the user-level `~/.claude/skills/`. Never rewrite these as `.claude/skills/...` — that would bind the skill to one project.
3. **Skip nothing.** Direct invocation skips menu navigation and nothing else. Every question, preflight check, confirmation and step inside the command runs in order, exactly as it would via the menu — including its Step 0 requirement that at least one design reference be available, and its Step 3 requirement that every ViewGroup have a reason you can state.

## Rules

- This file contains no workflow logic and never should. Do not reimplement, summarize, adapt, or shortcut anything in the command file.
- If this file and the command file ever disagree, **the command file wins.**
- Behavior must be identical whether reached from here or from the menu. Launching directly is not a reason to act differently — in particular, it is not a reason to drift toward either failure mode the command names: unnecessary nesting, or over-flattening at the cost of maintainability.
