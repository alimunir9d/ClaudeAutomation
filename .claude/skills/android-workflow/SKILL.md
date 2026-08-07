---
name: android-workflow
description: Entry point for Android developer workflow tasks on this repo (pull from develop, AI PR review, and future commands). Use when the user invokes /android-workflow or asks for a workflow menu.
---

# Android Developer Workflow Assistant

You are the router for this project's Android developer workflow assistant. This file's only job is to present a menu and dispatch to the selected command — it must never contain command-specific logic itself.

## What to do

1. Read `commands/manifest.json` in this skill's directory. It is a JSON array of commands, each with `id`, `title`, `description`, `file`, and `status`.
2. Filter to entries where `status` is `"active"`.
3. Call `AskUserQuestion` with one question asking which workflow action to run, using each active command's `title` as the option label and `description` as the option description.
4. Once the user picks an option, read *only* that command's `file` (resolved relative to `commands/`) and follow its instructions completely. Do not read or load any other command file.
5. If a new command is added later (a new file + manifest entry), this file requires no changes — it always re-reads the manifest fresh.

## Rules

- Do not implement command logic here. If a command file's instructions are incomplete or a stub, follow them as written (e.g. report that the command isn't implemented yet) rather than improvising the missing behavior yourself.
- Do not skip the menu, even if the user's request seems to imply a specific action — always let them pick from the menu unless they've already named the exact action in their invocation (e.g. `/android-workflow pull`), in which case you may route directly to the matching command by `id` or `title`.
