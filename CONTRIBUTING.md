# Contributing

Thanks for wanting to improve this. It's v1.0 — shaped by one team's workflow — so a change that comes from a workflow *you* actually repeat by hand is exactly the kind of contribution that makes it better.

## How changes land

`main` is protected. Nothing is pushed to it directly; every change arrives through a Pull Request that gets reviewed first.

```
task/<your-change>  →  PR into develop  →  reviewed & merged  →  develop → main at release time
```

1. **Fork the repo** (or, if you have write access, branch directly).
2. **Branch off `develop`**, not `main`. Name it `task/<short-description>`.
3. **Open the PR against `develop`.**
4. A maintainer reviews it. Expect questions — the rules below are strict on purpose, and most review comments are about one of them.

Releases go `develop` → `main` as a single reviewed PR, then a tag. Don't open a PR against `main`.

## Before you write anything

**Read [`.claude/skills/README.md`](.claude/skills/README.md).** It is the maintainer's guide to how the tree fits together — what a command file is, what a reference file is, why direct entry points contain no logic. A change that ignores that structure will be sent back regardless of how good the idea is.

## The rules a change must not break

These are the load-bearing ones. Each is in the tree because breaking it fails *quietly*.

**A skipped question stops the workflow.** Not "falls back to the recommended option", not "infers the obvious answer from repository state", not "re-asks in different words". If a new question is added anywhere, it inherits this, and the command's Safety Rules section says so.

**Remote is the source of truth for branch state — and only for branch state.** If your command reads branch *contents* to make a decision (a diff, a review, a merge, a release), it reads `references/git-common.md`, adds itself to that file's §G0 routing table, takes state from `origin/...` refs, and cites exactly one failure gate (§G3 or §G4). If it only reads and writes files in the working project, it must **not** fetch, read remote refs, or ask about remote access — and it says so where its shared logic lives.

**Detect, never assume.** Your command will run in other people's Android projects. Project name, module names, source sets, package names, resource layout, version scheme, branch names and git remote all vary — read them at run time, ask when detection is ambiguous. **Never emit a name you did not detect.** A hardcoded project name written into someone else's repository is the worst failure this tree has, because it is silent and it lands in a commit.

**Guard the entry.** Android commands verify a Gradle root *and* a resource tree. Git-only commands verify a git repository. An unsuitable project means stop cleanly, say what was expected, change nothing.

**One implementation, one source of truth.** Logic lives in the command file. Menus live in `android-workflow/SKILL.md`. References hold shared logic and are never dispatched to directly. Entry points point and nothing else. If you find yourself writing the same rule in two files, one of them is wrong.

**Stay Android-specific.** These are Android tools. Being usable in *any Android project* is the goal; generalizing to other platforms is out of scope.

## Adding a new command

Three files, in this order:

1. **The implementation** — `android-workflow/commands/<name>.md`, following the existing template: `# Command: <Title>`, `Status: **active.**`, a scope paragraph, `## Step N — …` sections, and a closing `## Safety Rules (apply throughout)`.
2. **The menu entry** — one bullet in `android-workflow/SKILL.md`. `AskUserQuestion` allows at most four options per question, so a new top-level command either takes a free slot or joins a grouping entry.
3. **The direct entry point** — `<name>/SKILL.md`, copying the shape of an existing leaf entry. Use **sibling-relative paths** (`../android-workflow/…`), never project-relative ones — a `.claude/skills/…` path resolves against whatever project is open and breaks the moment the tree is installed anywhere else.

If the command has irreversible effects, say so emphatically in its `description`, the way `create-release` and `dev-to-main-pr` do, so a casual mention of "releasing" can't trigger it.

## Testing your change

The `app/` module is a stock Android Studio project kept as a test bed — real layouts, real `strings.xml` files in several locales, real branches. Run your command against it before opening the PR.

Sync the tree to your machine-wide install and restart the session so the change actually loads:

```powershell
robocopy ".claude\skills" "$env:USERPROFILE\.claude\skills" /MIR /NFL /NDL /NJH /NJS
```

```bash
rsync -a --delete .claude/skills/ ~/.claude/skills/
```

**Edit in the repository, never in the global copy.** The global copy is a build artifact — it is overwritten on every re-sync, so changes made there are lost, and while they survive they make the two copies disagree about what the skill does.

## In your PR description

Say what workflow you were doing by hand, what the command does now, and which project you tested it against. If you changed a shared reference file, name every command that reads it — that is the change most likely to break something far away from the file you edited.
