# Command: Create Release

Status: **active.**

You are acting as an experienced Android release manager cutting a production release. This is the second half of the Release Workflow, and it runs **after** a human has reviewed and merged the Dev → Main PR created by `commands/dev-to-main-pr.md`.

Two properties define this command:

1. **The release is always cut from `main`.** The tag and the GitHub Release point at a commit on `origin/main` — never `develop`, under any circumstance, including when `develop` looks identical.
2. **It never invents release notes.** The notes belong to the release that was already prepared and confirmed; this command recovers them, it does not author them.

This command is independent of Dev → Main PR: it does not invoke it, and it typically runs in a fresh session with none of its state in memory. Everything it needs is recovered from the remote.

All shared behavior lives in `references/release-common.md`, referenced below as **§N**. Read that file before starting. Do not restate or re-implement its logic here.

Follow the steps below in order — do not skip or reorder them.

## Step 0 — Preflight

Run **§2 — Preflight and GitHub access gate**.

Say up front that this command will not touch the developer's current branch or working tree. If `gh` is missing or unauthenticated, §2 sends you to **§8 — Manual fallback**; take it rather than improvising.

## Step 1 — Verify main

Confirm the release actually landed in `main` before doing anything irreversible.

1. `git fetch origin --prune --tags` (already done in §2 — do not skip it if you arrived here another way).
2. `git log --oneline origin/main -5` — look for the merge of the release PR and the `chore(release): prepare release <version>` commit.
3. `git show origin/main:README.md` — the release line should be present on `main`.
4. `git log --oneline origin/main..origin/develop` — expected to be empty, or to contain only work committed *after* the release merged.

If the release preparation is **not** present in `main`, stop and tell the developer the Dev → Main PR has not been merged yet, naming what you looked for and didn't find. Do not tag, do not release, and **do not fall back to `develop`** — a release built from an unmerged branch is exactly the failure this check exists to prevent.

## Step 2 — Determine Version

Run **§3 — Version detection ladder** against **`origin/main`** — not `develop`. The version being released is whatever `main` now carries.

Show what was detected and where it came from, and confirm with the developer before continuing. Store as the **release version**.

## Step 3 — Existing tag check

Check both the tag and the release per **§7**:

- `git ls-remote --tags origin "refs/tags/<version>"`
- `gh release view <version> --repo <O>/<R>`

If the tag already exists, do **not** create another tag or release. Tell the developer, using this wording:

> Tag `<version>` already exists. A release may already have been created for this version.

Include the existing release URL if `gh release view` returned one, and the commit the tag points at. Then stop.

If the tag does not exist but a release somehow does, report that mismatch and stop rather than resolving it automatically.

## Step 4 — Recover the release notes

The notes for this release were already written and confirmed during release preparation. Recover them; do not generate new ones. Work down this ladder and stop at the first hit:

1. **The merged PR body** — the preferred source, because it is the exact text confirmed in Dev → Main PR:

   ```bash
   gh pr list --repo <O>/<R> --base main --head develop --state merged --limit 10 --json number,title,body,mergedAt,mergeCommit
   ```

   Pick the PR whose title is `Release <version>`. If several match, take the most recently merged. Extract the release-notes section of the body.
2. **The README on `main`** — the `## Release` section of `git show origin/main:README.md`, if it carries notes.
3. **Ask the developer.** If neither source yields notes, ask them to supply the notes for this release via `AskUserQuestion` (free text).

Do **not** generate notes from the diff at this stage. Generation belongs to release preparation, where the developer reviewed and confirmed them; regenerating here would silently publish text nobody approved, and would likely disagree with the PR.

Show the recovered notes and name the source they came from, then confirm before use per **§5**. If the developer skips that confirmation — or skips rung 3's request for notes — stop per **§0**. Do not fall through to generating notes from the diff; that would break "never invent release notes" on top of ignoring the skip.

## Step 5 — Create the tag

1. Resolve the commit: `git rev-parse origin/main`. This is the final merged production code, and it is the only acceptable target.
2. Show the developer exactly what is about to be created — tag name, the commit sha, and the commit subject — and ask for explicit confirmation via `AskUserQuestion`. This is the first irreversible action in the command. If they skip it, stop per **§0** and create nothing — no tag, no release. Nothing has been written at this point, so the report is simply that the release was not created.
3. On confirmation, create the tag per **§7**:

   ```bash
   gh api --method POST repos/<O>/<R>/git/refs -f ref="refs/tags/<version>" -f sha="<origin/main sha>"
   ```

If the API reports the reference already exists, someone created the tag between Step 3 and now — stop and report it per Step 3's wording. Never force or overwrite it.

## Step 6 — Create the GitHub Release

Write the confirmed notes to a scratchpad file and create the release per **§7**:

```bash
gh release create <version> --repo <O>/<R> --title "Release <version>" --notes-file <scratchpad>
```

The tag created in Step 5 already pins the commit, so do not pass `--target`. Do not pass `--generate-notes` — that would replace the confirmed notes with GitHub's own. Do not mark the release as draft or prerelease unless the developer explicitly asks.

If the call fails, report the error verbatim and stop. Note clearly that the tag from Step 5 now exists without a release, and that a retry should not attempt to recreate the tag.

## Step 7 — Report

Report, plainly:

- **Release created** — URL
- **Tag:** `<version>`
- **Commit on `main`:** sha and subject
- **Release notes used:** the confirmed notes, in full, and the source they were recovered from

## Safety Rules (apply throughout)

All of **§9 — Shared safety rules** applies. The ones this command is most likely to be tempted to break:

- **A skipped question ends the command** (§0). Skipping the tag confirmation means no release, not "proceed with the detected version." If the tag was created before the skip, say so — it exists without a release.
- **Never create a release from `develop`.** If `main` doesn't have the release, the answer is "the PR isn't merged yet," never "use develop instead."
- **Never overwrite an existing tag** and never create a duplicate release. An existing tag ends this command.
- **Never invent release notes.** Recover them from the merged PR, the README, or the developer. Thin but true beats rich but fabricated.
- **Never checkout, merge, or commit locally**, and never modify any project file. This command writes nothing to the repository contents — only a tag and a release (§1).
- **Always confirm before the tag and the release.** Both are outward-facing and effectively irreversible.
- **Report failures verbatim** and say exactly what was and wasn't created, so a retry starts from a known state.
