---
name: design-compose
description: Direct entry point for the Android Workflow "Designs → Compose" command — implement a screen design as idiomatic Jetpack Compose, following the project's existing Compose conventions and design system, from a screenshot, a Figma frame, and/or a written description. Use when the user explicitly invokes /design-compose or names this command. Equivalent to Android Workflow → Designs → Compose, with no mode question.
---

# Direct Entry: Designs — Compose

This is an **entry point, not an implementation.** It exists so the command can be run without walking the Android Workflow menu. The command itself is defined in exactly one place.

## What to do

1. Read `../android-workflow/commands/design-compose.md` — resolved against **this skill's own base directory**, which is given to you when the skill is invoked — and follow it completely, from its first step to its last.
2. Treat `../android-workflow/` as the command's **base directory**. Every relative path the command names — notably `references/design-common.md`, which it cites throughout as **§DN** — resolves from there. This command does not read `references/design-xml-common.md`; that file belongs to the two XML modes.

Paths here are sibling-relative on purpose, never project-relative. The skill tree is installed as a unit, so `../android-workflow/` resolves correctly whether that unit sits in a project's `.claude/skills/` or in the user-level `~/.claude/skills/`. Never rewrite these as `.claude/skills/...` — that would bind the skill to one project.
3. **Skip nothing.** Direct invocation skips menu navigation and nothing else. Every question, preflight check, confirmation and step inside the command runs in order, exactly as it would via the menu — including its Step 0 requirement that at least one design reference be available, and its Step 2 rule that converting an existing XML screen to Compose is asked about, never assumed.

## Rules

- This file contains no workflow logic and never should. Do not reimplement, summarize, adapt, or shortcut anything in the command file.
- If this file and the command file ever disagree, **the command file wins.**
- Behavior must be identical whether reached from here or from the menu. Launching directly is not a reason to act differently — in particular, it is not a reason to impose a Compose architecture, Material generation, or state pattern the project does not already use.
