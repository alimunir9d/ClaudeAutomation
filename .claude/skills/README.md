# Skills layout

Android developer automation with **one implementation per command** and **two ways to reach it**.

The skills are **Android-specific but project-agnostic**. They detect the project they are run in — its modules and source sets, its resource directories, its Gradle configuration, its git remote, its name — and never assume anything about the repository they were originally developed in. Each one also verifies it is somewhere it belongs before doing work, and stops cleanly if not.

```
                                 ┌─ /android-workflow → group → command   (hierarchical menu)
android-workflow/commands/*.md ──┤
                                 └─ /<command-name>                       (direct entry point)
```

## The two kinds of directory here

**`android-workflow/`** is the whole implementation and the hierarchical menu.

- `SKILL.md` — the router. Holds the main menu and the group follow-up menus. **This is the single source of truth for every menu.**
- `commands/*.md` — one file per executable command. **This is the single source of truth for every workflow.** Each file owns its steps, questions, and safety rules.
- `references/*.md` — shared logic that command files pull in by section number (`§N`). Never dispatched to directly. Three of them: `git-common.md` (branch/remote rules for every git-dependent command, cited as `§G0`–`§G6` — the `G` prefix keeps citations unambiguous for a command that reads two references), `release-common.md`, and `strings-common.md`.

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
- **They use sibling-relative paths — `../android-workflow/…` — never project-relative ones.** A path like `.claude/skills/android-workflow/…` resolves against whichever project is currently open, so it breaks the moment the tree is installed anywhere but that project. Sibling-relative paths resolve against the skill's own base directory and therefore work identically from a repo checkout and from the global install. **Never rewrite them as `.claude/skills/…`.**
- **They state the base directory.** A command's relative references — `references/release-common.md`, `references/strings-common.md` — resolve against `../android-workflow/`, not against the entry point's own directory. This is the one way a wrapper can genuinely break, so every entry says it explicitly.
- **They skip menu navigation and nothing else.** Every question, preflight check, confirmation and step inside a command runs identically whichever way it was reached. Being launched directly is never a reason to behave differently.
- **The thing pointed at always wins** in any disagreement.

## Where these live: canonical source vs. installed copy

The tree exists in **two places, with one direction of flow.**

| | Location | Role |
|---|---|---|
| **Canonical source** | `ClaudeAutomation/.claude/skills/` | Where the skills are developed, tested, reviewed and committed. Git-tracked; this is the history. **All edits happen here.** |
| **Installed copy** | `C:\Users\Ali\.claude\skills\` | A machine-wide install so the skills load in *any* Android project. A build artifact, not a source. |

**Never edit the global copy directly.** It is overwritten on every re-sync, so changes made there are lost, and while they survive they make the two copies disagree about what the skill does.

The workflow is one-way:

```
edit in this repo  →  test here  →  commit & push  →  re-sync to the global install
```

### Install / re-sync

Run after any change to the skills, from the repository root:

```bash
robocopy ".claude\skills" "$env:USERPROFILE\.claude\skills" /MIR /NFL /NDL /NJH /NJS
```

`/MIR` mirrors the tree, so deletions and renames propagate rather than leaving orphans behind. Restart the Claude Code session afterwards for the reloaded skills to register.

### Duplicate names while working in this repo

With the tree present both here and globally, every skill name is defined twice while this project is open. The project-level copy is expected to take precedence — verify this rather than assuming it. If duplicates ever behave ambiguously, the fix is to stop shipping the global copy of the affected name, **not** to delete anything from this repository: this repo is the source.

## Adding a new command

Three files, in this order:

1. **The implementation** — a new `android-workflow/commands/<name>.md`, following the existing template: `# Command: <Title>`, `Status: **active.**`, a scope paragraph, `## Step N — …` sections, and a closing `## Safety Rules (apply throughout)`.
2. **The menu entry** — one new bullet in `android-workflow/SKILL.md`. Each `AskUserQuestion` allows at most four options, so a new top-level command either takes a free slot or joins a grouping entry the way Release Workflow and Sync Strings do.
3. **The direct entry point** — a new `<name>/SKILL.md` here, copying the shape of any existing leaf entry. Its `description` should scope invocation to explicit requests; commands with irreversible effects (`create-release`, `dev-to-main-pr`) say so emphatically, so a casual mention of "releasing" cannot trigger them.

Adding a new group means one new follow-up section in `SKILL.md` plus one group entry point here.

Then **re-sync the global install** (above), or the new command exists only in this repo.

### Keeping new commands project-agnostic

Anything added here runs in other people's Android projects, so it must not encode facts about this one. In particular:

- **Detect, never assume** — the project name, module names, source sets, package names, resource layout, version scheme, branch names, and git remote all vary. Read them from the project at run time; ask when detection is ambiguous.
- **Never emit a name you did not detect.** A hardcoded project name written into another repository's files is the worst failure mode this tree has, because it is silent and it lands in a commit.
- **Guard the entry.** Android commands verify a Gradle root and a resource tree; git-only commands verify a git repository. Either way, an unsuitable project means stop cleanly, explain what was expected, and change nothing.
- **Decide whether the command reads branch contents, and be consistent about it.** If it does — a diff, a review, a merge, a release decision — it reads `android-workflow/references/git-common.md`, adds itself to that file's §G0 routing table, takes its branch state from `origin/...` refs, and cites exactly one failure gate (§G3 or §G4). If it doesn't — a command that only reads and writes files in the working project, as Sync Strings does — it must **not** fetch, read remote refs, or ask about remote access, and it says so where its own shared logic lives. Remote-first is a rule about where branch state comes from, not a network dependency to hand every new command.
- **Stay Android-specific.** These are Android tools. Generalizing them to other platforms is out of scope; being usable in *any Android project* is the goal.
