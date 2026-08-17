# Skills layout

This project's developer automation has **one implementation per command** and **two ways to reach it**.

```
                                 ┌─ /android-workflow → group → command   (hierarchical menu)
android-workflow/commands/*.md ──┤
                                 └─ /<command-name>                       (direct entry point)
```

## The two kinds of directory here

**`android-workflow/`** is the whole implementation and the hierarchical menu.

- `SKILL.md` — the router. Holds the main menu and the group follow-up menus. **This is the single source of truth for every menu.**
- `commands/*.md` — one file per executable command. **This is the single source of truth for every workflow.** Each file owns its steps, questions, and safety rules.
- `references/*.md` — shared logic that command files pull in by section number (`§N`). Never dispatched to directly.

**Every other directory here is a direct entry point** — a thin pointer that exists only so a command can be run without walking the menu. There are ten:

| Entry | Points at |
|---|---|
| `sync-branches` | `commands/sync-branches.md` |
| `pull-from-develop` | `commands/pull-from-develop.md` |
| `ai-pr-review` | `commands/ai-pr-review.md` |
| `dev-to-main-pr` | `commands/dev-to-main-pr.md` |
| `create-release` | `commands/create-release.md` |
| `sync-strings-full` | `commands/sync-strings-full.md` |
| `sync-strings-fast` | `commands/sync-strings-fast.md` |
| `branch-operations` | `SKILL.md` → *Branch Operations follow-up* |
| `release-workflow` | `SKILL.md` → *Release Workflow follow-up* |
| `sync-strings` | `SKILL.md` → *Sync Strings follow-up* |

## Rules for entry points

- **They contain no workflow logic and no menu definitions.** A leaf entry points at a command file; a group entry points at a follow-up section in `SKILL.md`. Neither restates what it points at, so nothing can drift out of step.
- **They state the base directory.** A command's relative references — `references/release-common.md`, `references/strings-common.md` — resolve against `android-workflow/`, not against the entry point's own directory. This is the one way a wrapper can genuinely break, so every entry says it explicitly.
- **They skip menu navigation and nothing else.** Every question, preflight check, confirmation and step inside a command runs identically whichever way it was reached. Being launched directly is never a reason to behave differently.
- **The thing pointed at always wins** in any disagreement.

## Adding a new command

Three files, in this order:

1. **The implementation** — a new `android-workflow/commands/<name>.md`, following the existing template: `# Command: <Title>`, `Status: **active.**`, a scope paragraph, `## Step N — …` sections, and a closing `## Safety Rules (apply throughout)`.
2. **The menu entry** — one new bullet in `android-workflow/SKILL.md`. Each `AskUserQuestion` allows at most four options, so a new top-level command either takes a free slot or joins a grouping entry the way Release Workflow and Sync Strings do.
3. **The direct entry point** — a new `<name>/SKILL.md` here, copying the shape of any existing leaf entry. Its `description` should scope invocation to explicit requests; commands with irreversible effects (`create-release`, `dev-to-main-pr`) say so emphatically, so a casual mention of "releasing" cannot trigger them.

Adding a new group means one new follow-up section in `SKILL.md` plus one group entry point here.
