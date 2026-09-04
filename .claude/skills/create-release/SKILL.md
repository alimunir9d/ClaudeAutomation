---
name: create-release
description: Direct entry point for the Android Workflow "Create Release" command — after the develop → main PR has been merged, tag main and publish the GitHub Release. Use ONLY when the user explicitly invokes /create-release or names this command by name; casual mentions of releasing, shipping, tagging, or versioning must not trigger it, because this command creates an irreversible tag and a public release. Equivalent to Android Workflow → Release Workflow → Create Release.
---

# Direct Entry: Create Release

This is an **entry point, not an implementation.** It exists so the command can be run without walking the Android Workflow menu. The command itself is defined in exactly one place.

## What to do

1. Read `../android-workflow/commands/create-release.md` — resolved against **this skill's own base directory**, which is given to you when the skill is invoked — and follow it completely, from its first step to its last.
2. Treat `../android-workflow/` as the command's **base directory**. Every relative path the command names — notably `references/release-common.md`, which it cites throughout as **§N** — resolves from there.

Paths here are sibling-relative on purpose, never project-relative. The skill tree is installed as a unit, so `../android-workflow/` resolves correctly whether that unit sits in a project's `.claude/skills/` or in the user-level `~/.claude/skills/`. Never rewrite these as `.claude/skills/...` — that would bind the skill to one project.
3. **Skip nothing.** Direct invocation skips menu navigation and nothing else. Every question, preflight check, confirmation and step inside the command runs in order, exactly as it would via the menu — including the `gh` access gate, the verification that the release actually landed in `main`, the existing-tag check, and the confirmation before the tag is created.

## Rules

- This file contains no workflow logic and never should. Do not reimplement, summarize, adapt, or shortcut anything in the command file.
- If this file and the command file ever disagree, **the command file wins.**
- Behavior must be identical whether reached from here or from the menu. Launching directly is not a reason to act differently — in particular, it is never a reason to skip the merged-into-`main` verification, and **never** a reason to release from `develop`.
