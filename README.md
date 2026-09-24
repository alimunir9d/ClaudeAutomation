# Claude Automation — Android Developer Skills

**v1.0.0** · A set of [Claude Code](https://claude.com/claude-code) skills that turn the repetitive parts of Android development into one-command workflows.

Branch merges, PR reviews, release cuts, translation syncs, screen implementation — the work every Android developer does every week, done the same careful way every time, without doing it by hand.

> Built around the way we work at **9D Technologies** — our `develop`/`main` branching, our release process, our multi-language string files, our design-to-screen handoff. The skills themselves are **project-agnostic**: they detect the project they are run in and never assume anything about the repository they were developed in, so they work in any Android project.

---

## What's in it

**10 commands, reachable 15 ways**, organised into 4 groups.

Every command can be reached two ways:

```
                    ┌─ /android-workflow → pick a group → pick a command   (menu)
one implementation ─┤
                    └─ /<command-name>                                     (direct)
```

### 🔀 Branch Operations

| Command | What it does |
|---|---|
| `/sync-branches` | Safely merge one branch into another, with interactive conflict resolution. Asks before every decision; never guesses on business-logic conflicts. |
| `/pull-from-develop` | Merge the latest `develop` into your current branch. Same safety as Sync Branches, no branch questions. |
| `/ai-pr-review` | Review the diff between a source and target branch as an experienced Android engineer would. **Read-only** — never checks out, merges or commits. Generic or Events focus. |

### 🎨 Designs

Implement a screen from a screenshot, a Figma frame, and/or a written description — into the project's *own* conventions.

| Command | What it does |
|---|---|
| `/design-xml-constraint` | XML layout, strict: one root `ConstraintLayout`, every view a direct child, no other ViewGroups. |
| `/design-xml-optimal` | XML layout with the minimum *reasonable* ViewGroup hierarchy — ConstraintLayout-rooted, other containers only where they genuinely pay for themselves. |
| `/design-compose` | Idiomatic Jetpack Compose, following the project's existing Compose conventions, Material generation and design system. |
| `/designs` | Group entry — asks XML or Compose, then the mode. |
| `/design-xml` | Group entry — asks ConstraintLayout Only or Optimal. |

### 🚀 Release Workflow

| Command | What it does |
|---|---|
| `/dev-to-main-pr` | Prepare the release on the **remote** `develop` and open the `develop` → `main` PR. Never merges it — a human does. Never touches your working tree. |
| `/create-release` | After that PR is merged: tag `main` and publish the GitHub Release. Recovers the already-agreed release notes rather than inventing new ones. |

### 🌍 Sync Strings

Synchronize Android string translations across every supported language, with English as the source of truth.

| Command | What it does |
|---|---|
| `/sync-strings-full` | Complete audit — every English key against every language file. Accuracy over token cost. |
| `/sync-strings-fast` | Token-efficient incremental sync. Scans upward from the bottom of the English file and stops at the synchronization boundary. |

### Group entry points

`/android-workflow` (full menu) · `/branch-operations` · `/designs` · `/design-xml` · `/release-workflow` · `/sync-strings`

---

## Install

Requires [Claude Code](https://claude.com/claude-code). Clone the repo, then copy the skills into your machine-wide Claude directory so they load in **any** Android project:

**Windows (PowerShell):**

```powershell
robocopy ".claude\skills" "$env:USERPROFILE\.claude\skills" /MIR /NFL /NDL /NJH /NJS
```

**macOS / Linux:**

```bash
rsync -a --delete .claude/skills/ ~/.claude/skills/
```

Restart your Claude Code session afterwards so the skills register. Then, from any Android project:

```
/android-workflow
```

`/MIR` and `--delete` mirror the tree, so renames and deletions propagate instead of leaving orphans behind.

---

## Design principles

These are the rules the whole tree is built on. They are worth knowing before you contribute.

**A skipped question stops the workflow.** Every question is a real gate. If you press Skip, the command ends — it does not fall back to the `(Recommended)` option, does not infer the "obviously applicable" answer from repository state, and does not re-ask in different words. A skip is a withdrawal from the decision, not a delegation of it.

**Remote is the source of truth for branch state.** Any command that compares, reviews, merges or releases branch *contents* takes them from `origin/...` refs. A local branch can be days behind, and it fails quietly — a stale baseline moves the merge base backwards and the output still looks like a normal review. If the remote can't be reached, you are told and asked; never silently substituted for.

**Purely-local commands stay local.** Sync Strings and Designs read and write files on disk. They never fetch, never read remote refs, and never ask about remote access. "Remote-first" is a rule about where branch state comes from, not a network dependency every command inherits.

**Detect, never assume.** Project name, module names, source sets, package names, resource layout, version scheme, branch names, git remote — all of it is read from the project at run time. Emitting a name you did not detect is the worst failure mode here, because it is silent and it lands in a commit.

**Guard the entry.** Android commands verify a Gradle root and a resource tree; git-only commands verify a git repository. An unsuitable project means stop cleanly, explain what was expected, and change nothing.

**One implementation, one source of truth.** Command logic lives in exactly one file. The menu lives in exactly one file. Direct entry points contain no logic — they only point. Nothing can drift out of step.

---

## Repository layout

```
.claude/skills/
├── android-workflow/           the whole implementation + the menu
│   ├── SKILL.md                the router — single source of truth for every menu
│   ├── commands/*.md           one file per command — single source of truth for every workflow
│   └── references/*.md         shared logic, pulled in by section number (§G, §D, §X, §N)
└── <command-name>/SKILL.md     15 thin direct entry points — pointers, no logic
```

The `app/` module is a stock Android Studio project kept as a **test bed** — somewhere to run the skills against real layouts, real `strings.xml` files and real branches. It is not the product; the skills are.

See [`.claude/skills/README.md`](.claude/skills/README.md) for the maintainer's guide — how the tree fits together, and how to add a new command.

---

## Contributing

**Contributions are very welcome.** This is v1.0 — a first version, shaped by one team's workflow, and it will get better the more workflows it meets.

The repository is public to read and fork. To propose a change, open a Pull Request — `main` is protected, so every change lands through review. See [CONTRIBUTING.md](CONTRIBUTING.md) for the branch model, the house style, and what makes a change easy to accept.

Good first contributions: a new command for a workflow you repeat by hand, better detection for a project layout the skills mis-read, or a fix for any rule above that a command quietly breaks.

---

## License

[MIT](LICENSE) © Ali Munir
