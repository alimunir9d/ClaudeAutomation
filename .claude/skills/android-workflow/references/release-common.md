# Reference: Release Common

Shared utilities for the two Release Workflow commands — `commands/dev-to-main-pr.md` and `commands/create-release.md`.

This file is **not** a menu command. It is never dispatched to directly; it is pulled in by a command file that says "run §N". Each section below is addressable by its number so the command files can reference behavior instead of restating it. If you find yourself writing a `git` or `gh` invocation inside a command file, it belongs here instead.

Throughout, `<O>/<R>` means the resolved owner/repo pair from §2, and `<version>` means the version confirmed with the developer in §3.

## §0 — Answering questions and skipping

Every `AskUserQuestion` in these workflows is a real gate. If the developer does not select an option, **stop the entire command immediately.**

**What counts as no selection:** they press Skip; the answer comes back as `[No preference]` or empty; or they answer by declining — "none", "neither", "stop", "cancel". These are all the same signal. Skipping is a withdrawal from the decision, not a delegation of it to you.

**What to do:** end the command. Make no further tool calls on its behalf.

**What never to do:**

- Do not fall back to the `(Recommended)` option. That label is a suggestion for someone who is choosing; it carries no authority when nobody chose.
- Do not infer the "obviously applicable" answer from repository state, even when only one option could possibly work.
- Do not re-ask the same question in different words, and do not split it into smaller questions to get an answer indirectly.
- Do not continue with a narrowed or "safe subset" version of the workflow.

**What to say:** one short line naming the question that was skipped and stating that the workflow stopped. Do not re-litigate the choice or append an offer to continue.

**Disclose what already happened.** If anything in this run completed before the skip — a remote README commit, a created PR, a tag — list it explicitly with its URL or sha. "Stopped" must never be read as "nothing exists" when something does.

**Resumption:** the workflow restarts only when the developer invokes it again, from the beginning. Do not offer to pick up where it left off.

## §1 — Remote-only contract

Where branch state comes from is the tree-wide rule in `references/git-common.md` **§G1**, and how it is refreshed is **§G2**. Read those; this section does not restate them. What follows is the part specific to the release commands: they not only *read* from the remote, they never write anywhere else either. The developer's local clone is a lens for reading remote refs, never a staging area.

Never run any of these as part of a release command:

- `git checkout` / `git switch` — no branch is ever checked out, including `develop` and `main`.
- `git merge`, `git rebase`, `git cherry-pick`, `git reset`, `git stash`.
- `git commit` — the release-preparation commit is created **remotely** by the GitHub API (§6), never locally.
- `git push` — nothing local exists to push.
- Any write to a file in the working tree. The one file this workflow modifies (`README.md`) is edited through the API on the remote branch. Scratchpad files are fine — they are outside the repo.

Permitted git commands are read-only or ref-only:

- `git fetch origin --prune --tags` — the canonical refresh (`git-common.md` §G2) and the **only** command here that writes anything, and it writes remote-tracking refs and tags, never the working tree or a local branch. This is the same boundary `commands/ai-pr-review.md` Step 0 already establishes.
- `git show origin/<branch>:<path>` — read a file's content as it exists on the remote.
- `git rev-parse origin/<branch>`, `git rev-parse origin/<branch>:<path>` — resolve a commit sha or a blob sha.
- `git log`, `git diff`, `git ls-remote`, `git merge-base` — always against `origin/...` refs.

Never read local `develop` or local `main` (`git-common.md` §G1). In this clone they are routinely stale (local `develop` has sat behind its remote for several commits), and a release built from a stale local ref would ship the wrong code. Always write `origin/develop` and `origin/main` explicitly — a bare `develop` in any command in these workflows is a bug. Unlike the branch commands, there is no local half here: these commands have no local operation to perform, so a local ref has no legitimate role at all.

The developer's current branch and working-tree state are irrelevant to both commands. Say so in preflight so they know it is safe to run mid-task, and never ask them to commit, stash, or switch branches.

## §2 — Preflight and GitHub access gate

Run this before anything else in either command.

0. **Confirm the project is suitable.** These are Android release commands operating on a GitHub repository, and they may be invoked anywhere once installed machine-wide. Check, in order:
   - `git rev-parse --is-inside-work-tree` — if this is not a git repository, **stop cleanly**, say so, and modify nothing.
   - A Gradle project root — `settings.gradle.kts` / `settings.gradle`, or a `build.gradle(.kts)`. If absent, this is not an Android project; stop and say what was looked for.

   Do not attempt to work around either result. A clean stop with a clear explanation is the correct outcome.
1. **Check `gh`.** `gh --version`, then `gh auth status`. Every remote write in these workflows goes through the GitHub API, so if `gh` is missing or unauthenticated, **stop the normal path here** and go to §8. Do not continue and do not improvise a local-git substitute.
   - Not installed → tell the developer plainly and give the install command:

     ```bash
     winget install --id GitHub.cli
     ```

   - Installed but not authenticated → give:

     ```bash
     gh auth login
     ```

   - After either, they must restart the shell/session before `gh` is visible.
2. **Resolve the repository.** `git remote get-url origin`, then parse to `<owner>/<repo>`:
   - SSH form `git@github.com:<owner>/<repo>.git`
   - HTTPS form `https://github.com/<owner>/<repo>.git`

   Strip any trailing `.git`. If the remote is not a GitHub URL, stop — these commands are GitHub-specific.
3. **Refresh remote state.** `git fetch origin --prune --tags`, per `git-common.md` **§G2**. Do this even if the session already fetched: both commands make decisions from remote refs, and a stale ref produces a wrong version, wrong notes, or a tag on the wrong commit.

   **Check the fetch's exit status.** Non-zero → **`git-common.md` §G4 (Gate B)**: report the error verbatim and offer exactly `Retry` / `Cancel`. Do not continue on cached refs, and do not offer local branches — for these commands there is no acceptable local substitute (§1), and §8 is not a valid destination either, because it needs a successful fetch of its own.
4. **Verify the branches exist on the remote.** `git rev-parse --verify origin/develop` and `git rev-parse --verify origin/main`. If either fails, say which one and stop — never guess at a near-match name like `dev` or `master`.

   This is an existence check, not a freshness check: per §G2 it succeeds on cached refs from any earlier fetch, however old. It is step 3's exit status that establishes the state is current, so passing this step is never a reason to skip that one.
5. **Report the developer's local state as untouched.** `git branch --show-current` — mention it once, explicitly framed as "this branch will not be touched." Do not run `git status` gates; uncommitted local work is irrelevant here because nothing local is read or written.

## §3 — Version detection ladder

Work down this ladder against the branch the calling command names (`origin/develop` for release prep, `origin/main` for release creation) and stop at the first confident hit.

1. **README release line.** `git show origin/<branch>:README.md` and look for a line matching `Release: <version>`. This is the convention these commands themselves write, so once one release has been prepared this is the authoritative source.
2. **Newest remote tag.** `git ls-remote --tags origin`, strip `refs/tags/` and any `^{}` suffix, and sort with version ordering (`git tag --sort=-v:refname` semantics) to find the highest.
3. **Gradle version name.** Locate the Android application module's build file on the remote — glob for `**/build.gradle.kts` / `**/build.gradle` and take the one declaring `applicationId` (the module name is *not* always `app`) — then `git show origin/<branch>:<that path>` and read `versionName`. Treat this as weak evidence: it is a build identifier and many projects never bump it in step with releases.
4. **Ask.** If nothing above yields a version, or the candidates disagree, ask the developer for the release version via `AskUserQuestion` with a free-text answer. If they skip it, stop the command per §0 — a skipped version question does not mean "use your best guess," and there is no safe default for a number that will end up on a tag.

Rules that apply regardless of which rung answered:

- **A detected version is a proposal, never a decision.** Always show the developer the value found, which rung it came from, and — when incrementing — the suggested next version, then confirm before using it. Silently adopting a detected version is how the wrong number ends up on a tag.
- **Follow the project's existing shape.** If a previous release used a four-part version like `3.6.0.2`, propose a four-part successor; do not renormalize it to semver, and do not invent a new scheme.
- **Disagreement means ask.** If the README says one version and the newest tag says another, show both and let the developer choose.
- **The confirmed version is used verbatim everywhere** — README release line, PR title, PR body, tag name, release title. Never reformat it between uses (no stripping a leading `v`, no zero-padding).

Expect rung 4 on the first ever run in a repo with no README, no tags, and an untouched `versionName`. That is the designed path, not a failure — ask cleanly rather than manufacturing a number.

## §4 — Release-notes generation

Build notes **only** from what actually changed between the remote branches.

Source material, in order:

1. `git log --oneline origin/main..origin/develop` — the commits the release would deliver.
2. `git diff --stat origin/main...origin/develop` — scope and file footprint.
3. `git diff origin/main...origin/develop` on specific files, or reading a changed file in full, whenever a commit subject is too vague to describe honestly.

Use **three dots** for the diff. `origin/main...origin/develop` measures develop's own work from the fork point; two dots would attribute to develop changes it never made. (Same reasoning as `commands/ai-pr-review.md` Step 5.)

If `origin/main..origin/develop` is empty, `develop` has nothing to release — say so and stop. Do not manufacture notes for a no-op release.

Writing the notes:

- Group into short sections — **Added** / **Changed** / **Fixed** — and omit any section with no entries.
- One plain-language line per entry, written for someone reading a release page, not a commit log. Prefer "Branch sync now resolves conflicts interactively" over "merge fix".
- Collapse noise into a single line rather than enumerating it: `.idea/`, `build/`, `*.iml`, lockfiles, generated resources, binary assets ("N IDE/config files updated").
- Merge commits and revert pairs describe process, not product — fold them into the change they delivered, or drop them.

The hard rule: **every line must be traceable to something in the diff.** Do not describe a feature because the version number implies it, because a branch name suggests it, or because it would round out the notes. If the diff does not support a claim, the claim does not go in.

## §5 — Release-notes confirmation

Notes are never used unconfirmed, whether generated (§4) or supplied by the developer.

1. Show the notes back in full, in a fenced block, exactly as they would appear.
2. Ask via `AskUserQuestion`:
   - `Approve (Recommended)` — use as shown.
   - `Edit` — take a free-text revision and show the result again for approval.
   - `Regenerate` — only offered for generated notes; rebuild from §4 with any steer the developer gives.
3. Loop until approved.

If the developer skips this question, stop the command per §0. Silence is not approval — unconfirmed notes must never reach a PR body or a release page.

Once approved, the text is frozen. The README release entry, the release-preparation commit, the PR body, and the eventual GitHub Release body all use the **same confirmed text, byte for byte**. Do not re-summarize it, re-wrap it, or tighten it for one of those surfaces — a PR body and a release page that disagree are how release notes stop being trustworthy.

## §6 — Remote README write

This writes `README.md` on remote `develop` through the GitHub Contents API. It creates a commit on the remote branch directly — there is no local edit, no local commit, and no push.

1. **Read the current file.**
   - Content: `git show origin/develop:README.md`
   - Blob sha: `git rev-parse origin/develop:README.md`

   If both fail, the file does not exist on `develop` — that is the create case below.
2. **Build the new content.**
   - **README exists:** change only the release line. If a `Release: <old>` line is present, replace that one line. If there is no release section, append a `## Release` section at the end. Preserve everything else exactly — no reflowing, no heading renumbering, no formatting cleanup, no unrelated edits. Match the file's existing conventions (heading depth, blank-line style, trailing newline).
   - **README absent:** create a minimal one — the project's own name as an H1 and a `## Release` section, nothing more:

     ```markdown
     # <detected project name>

     ## Release

     Release: <version>
     ```

     **Detect the name from the project being released; never hardcode one.** In order: `rootProject.name` from `git show origin/develop:settings.gradle.kts` (or `settings.gradle`); failing that, the repository name parsed from `git remote get-url origin`. If neither resolves, ask the developer rather than inventing a name.

     Do not add a description line. You do not know what the project is, and guessing puts a sentence into someone's README that they never wrote.

3. **Show the developer the exact before/after of the release line** before writing anything. One line each — this is the confirmation that the right version is about to land.
4. **Write it.** Save the new content to a scratchpad file, base64-encode it, and PUT it:

   ```bash
   gh api --method PUT repos/<O>/<R>/contents/README.md -f message="chore(release): prepare release <version>" -f content="<base64>" -f branch=develop -f sha="<blob-sha>"
   ```

   In PowerShell, encode with `[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes((Get-Content <file> -Raw)))`.

   Omit `-f sha=...` **only** in the create case. When the file exists, `sha` is mandatory: it makes the call fail rather than clobber if `develop` moved after the fetch. If the call returns a 409/422 conflict, `develop` advanced underneath the workflow — re-fetch, re-read, rebuild, and show the developer what changed before retrying. Never retry by dropping `sha`.
5. **Confirm it landed.** The response contains the new commit sha — report it. Then `git fetch origin --prune --tags` (§G2) and verify `origin/develop` advanced to it.

## §7 — Remote GitHub operations

All of these run against `<O>/<R>` resolved in §2. Bodies are always passed via `--body-file` / `--notes-file` pointing at a scratchpad file, never inline — inline bodies mangle newlines and markdown.

**Check for an existing open release PR:**

```bash
gh pr list --repo <O>/<R> --base main --head develop --state open --json number,title,url
```

**Create the PR:**

```bash
gh pr create --repo <O>/<R> --base main --head develop --title "Release <version>" --body-file <scratchpad>
```

`gh pr create` never merges. Do not pass `--fill`, and never call `gh pr merge` in either command.

**Check for an existing tag or release:**

```bash
git ls-remote --tags origin "refs/tags/<version>"
gh release view <version> --repo <O>/<R>
```

**Create the tag on a specific commit:**

```bash
gh api --method POST repos/<O>/<R>/git/refs -f ref="refs/tags/<version>" -f sha="<commit-sha>"
```

The API rejects a ref that already exists, so this cannot overwrite a tag — that is why tags are created this way rather than with `git tag`/`git push --tags`. If it returns "Reference already exists", treat that as the existing-tag case and stop; never force it.

**Create the release:**

```bash
gh release create <version> --repo <O>/<R> --title "Release <version>" --notes-file <scratchpad>
```

The tag already exists at this point, so do not pass `--target` — it would be ignored at best and misleading at worst. Do not pass `--generate-notes` (that would invent notes, violating §4/§5). Do not pass `--prerelease` or `--draft` unless the developer explicitly asks.

If any `gh` call fails, report the actual error text and stop. Do not retry with different flags, and do not fall back to a local git equivalent.

## §8 — Manual fallback when `gh` is unavailable

Reached from §2 when `gh` is missing or unauthenticated, and the developer would rather proceed now than install it.

**Precondition: the fetch in §2 step 3 succeeded.** This fallback exists for a missing *tool*, not for a missing *remote*. Everything below reads remote refs, so if the fetch is what failed, this section is unreachable — §G4 (Gate B) offers `Retry` / `Cancel` and stops there. A paste-ready README and a set of release notes built from stale cached refs would be worse than no fallback at all, because they look finished.

Do **not** substitute local git operations. The remote-only contract (§1) still holds — a fallback that checks out `develop` and commits locally is worse than no fallback, because it silently changes what the developer asked for.

Instead, do all the work that does not require the API, and hand over everything pre-built:

1. Version detection (§3) and release notes (§4, §5) run normally — they are read-only and need only the successful fetch this section already requires.
2. Print the **complete new README content** in a fenced block, ready to paste into GitHub's web editor on the `develop` branch, along with the commit message `chore(release): prepare release <version>`.
3. Print the compare URL to open the PR from:

   `https://github.com/<O>/<R>/compare/main...develop?expand=1`

   with the PR title (`Release <version>`) and the full PR body in a fenced block.
4. For Create Release, print the release URL:

   `https://github.com/<O>/<R>/releases/new?tag=<version>&target=main`

   with the title and the confirmed notes in a fenced block.
5. State explicitly, as the last line: **nothing was created** — no commit, no PR, no tag, no release. List what the developer still has to do, in order.

## §9 — Shared safety rules

Apply to both release commands, at every step:

- **A skipped question ends the command** (§0). Never substitute the `(Recommended)` option, never infer the answer from repo state, never re-ask it another way.
- **Never merge the Dev → Main PR.** No `gh pr merge`, no offer to merge, no "want me to merge it now?" The human developer merges it. This is the single most important rule in the workflow.
- **Never create a release from `develop`.** The tag and the GitHub Release always point at a commit on `origin/main`.
- **Never overwrite an existing tag** and never create a duplicate release. If either already exists, report and stop.
- **Never invent release notes.** Every line traces to the diff (§4), or comes from the developer, or comes from the already-confirmed notes of this release (§5). "Do not invent" outranks "produce complete-looking notes" — thin but true beats rich but fabricated.
- **Never modify any file other than `README.md`.** No version bump in any module's `build.gradle(.kts)`, no `gradle.properties`, no CHANGELOG unless the developer explicitly asks for it as a separate action.
- **Never touch local branches or the working tree** (§1). The developer must end both commands standing exactly where they started, with the same uncommitted work they had.
- **Never assume local refs are current** (`git-common.md` §G1). Read `origin/...` refs only, after a fetch that actually succeeded — a ref resolving is not evidence it is fresh (§G2). If the fetch fails, §G4 (Gate B) stops the command; never continue on cached refs and never offer a local substitute.
- **Confirm before every irreversible remote action** — the README commit, the PR creation, the tag creation, the release creation. Show what is about to happen with the concrete values, and wait for a yes.
- **Report failures verbatim.** If a `gh` call fails, show the error and stop. Do not paper over it, do not retry with weakened flags, and never report a step as done when it errored.
