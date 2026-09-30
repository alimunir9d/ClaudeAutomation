---
name: dev-to-main-pr
description: Direct entry point for the Android Workflow "Dev → Main PR" command — prepare a release on remote develop and open the develop → main pull request, without merging it. Use ONLY when the user explicitly invokes /dev-to-main-pr or names this command by name; casual mentions of releases, PRs, or merging must not trigger it, because this command writes a commit to the remote repository. Equivalent to Android Workflow → Release Workflow → Dev → Main PR.
---

# Direct Entry: Dev → Main PR

This is an **entry point, not an implementation.** It exists so the command can be run without walking the Android Workflow menu. The command itself is defined in exactly one place.

## What to do

1. Read `../android-workflow/commands/dev-to-main-pr.md` — resolved against **this skill's own base directory**, which is given to you when the skill is invoked — and follow it completely, from its first step to its last.
2. Treat `../android-workflow/` as the command's **base directory**. Every relative path the command names — notably `references/release-common.md`, cited throughout as **§N**, and `references/git-common.md`, cited as **§GN** — resolves from there.

Paths here are sibling-relative on purpose, never project-relative. The skill tree is installed as a unit, so `../android-workflow/` resolves correctly whether that unit sits in a project's `.claude/skills/` or in the user-level `~/.claude/skills/`. Never rewrite these as `.claude/skills/...` — that would bind the skill to one project.
3. **Skip nothing.** Direct invocation skips menu navigation and nothing else. Every question, preflight check, confirmation and step inside the command runs in order, exactly as it would via the menu — including the `gh` access gate, the remote-only contract, and every confirmation before a remote write.

## Rules

- This file contains no workflow logic and never should. Do not reimplement, summarize, adapt, or shortcut anything in the command file.
- If this file and the command file ever disagree, **the command file wins.**
- Behavior must be identical whether reached from here or from the menu. Launching directly is not a reason to act differently — in particular, it is never a reason to skip a confirmation, and **never** a reason to merge the PR.
