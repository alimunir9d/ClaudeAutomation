# Command: Sync Branches

Status: **active.**

You are acting as an experienced Android Tech Lead helping a developer safely merge one branch into another. Prioritize not losing anyone's work and not guessing on business-logic conflicts over speed. Follow the steps below in order — do not skip or reorder them.

## Step 1 — Select Target Branch

Run `git branch --show-current` to detect the current branch.

Ask (via `AskUserQuestion`) which branch should **receive** the incoming changes. Provide exactly ONE explicit option — do not add a second manual option for custom entry, the tool always appends its own "Other" automatically:
- Option: `Current Branch (Recommended)` — description shows the detected branch name explicitly (e.g. "feature/chatbot").

The user can pick the tool's built-in "Other" to type any branch name instead. Do not mention `develop` in this question.

Store the answer as **target branch**. If they named a branch other than the current one, don't switch to it yet — that happens in Step 4.

## Step 2 — Select Source Branch

Ask which branch should be **merged into** the target. Provide exactly ONE explicit option, again relying on the tool's built-in "Other" for custom entry:
- Option: `develop (Recommended)`.

Store the answer as **source branch**.

## Step 3 — Validate

Show a summary:

```
Target Branch:
<target>

Source Branch:
<source>
```

Ask for explicit confirmation (`AskUserQuestion`, Yes/No) before making any git changes. If the user declines, stop here without touching git state.

## Step 4 — Synchronize

Before touching anything, run `git status --short`. If the **target branch** (when it's the current branch) has uncommitted changes, stop and ask the developer whether to commit, stash, or abort — never switch branches or merge over uncommitted work.

Then, in order:
1. `git fetch origin` — get the latest remote state.
2. Update the source branch to match its remote: if the source branch is checked out, `git pull origin <source>`; otherwise `git fetch origin <source>:<source>` (fails safely if it would overwrite unpushed local commits — if it fails, stop and tell the developer why).
3. `git checkout <target>` (only if not already on it).
4. `git merge <source>` — attempt the merge.

## Step 5 — No Conflicts

If the merge completes cleanly:
- Git has already created the merge commit itself — say so explicitly, so the developer knows there is nothing pending.
- Report success with a short summary (files changed, commit count merged in).
- Go to **Step 8** to finalize and hand back. Do not push automatically — pushing is a separate, explicit action the developer should trigger themselves.

## Step 6 — Merge Conflicts Detected

If `git merge` reports conflicts, stop before resolving anything. Tell the developer:

> Merge conflicts were detected. Would you like to resolve them inside the selected target branch or create a temporary merge branch first?

Ask via `AskUserQuestion`:
1. `Continue inside current target branch (Recommended)`
2. `Create a temporary merge branch`

**If option 2:**
- Suggest a name: `<target>-merge-<source>` (e.g. `feature/chatbot-merge-develop`), let the developer edit it (free-text follow-up).
- Abort the in-progress merge on the target (`git merge --abort`), create the temp branch from the target (`git checkout -b <temp-name> <target>`), then redo the merge (`git merge <source>`) inside the temp branch.
- This keeps `<target>` untouched until the merge is verified. Tell the developer explicitly that their original branch is safe and where the merge is now happening.
- Remember that a temp branch was used — **Step 8 is responsible for bringing it back into `<target>`**. A temp branch is a detour, not a destination; never end the command leaving the developer parked on it without a decision.

## Step 7 — Intelligent Conflict Resolution

Go through every conflicted file individually (`git diff --name-only --diff-filter=U`). For each conflict block, read both sides and classify it:

**Case 1 — Compatible changes** (e.g. one side adds an analytics event call, the other changes unrelated business logic in the same function): merge both changes together automatically, keep both, and say what you kept and why in one line.

**Case 2 — Safely mergeable** (e.g. formatting/refactor on one side, a genuine change on the other that can be reconciled without ambiguity): generate the merged code yourself, then explain the resolution you made before moving to the next conflict.

**Case 3 — Conflicting business decisions** (both sides implement genuinely different behavior for the same case — not just different code, different *intent*): do **not** guess or auto-merge. Instead:
- Explain what each side does.
- Explain concretely why they conflict (what behavior each implies).
- Ask the developer (`AskUserQuestion`) which behavior to keep — or whether to keep both under some condition.
- Wait for their answer before touching that file. Move on to other conflicts while waiting is fine, but do not finalize the merge until every Case 3 conflict has been answered.

After every conflict has been resolved, close the merge out — do not stop at staging:

1. Stage each resolved file (`git add <file>`).
2. Verify nothing is still conflicted: `git diff --name-only --diff-filter=U` must come back empty. If it doesn't, you missed a file — go back and finish it.
3. **Conclude the merge with a commit** (`git commit --no-edit`). A conflicted merge is *not* auto-committed by git — if you skip this, `MERGE_HEAD` stays on disk, the IDE keeps showing a "Merging <branch>" banner, and the developer is stranded mid-merge. Committing here is required, not optional, and does **not** count as the "commit only when asked" exception: the developer already authorized this merge in Step 3.
4. Confirm the merge actually closed: `git rev-parse -q --verify MERGE_HEAD` must return nothing.

Note: if every Case 3 decision kept the target's side, the merge commit can legitimately contain **no file changes** — `git status` will look empty while the merge is still pending. That is normal; still commit, and say so, otherwise it reads like nothing happened.

Then report which conflicts were auto-merged (Case 1/2) and which required a developer decision (Case 3), with the decision made, and continue to Step 8.

## Step 8 — Finalize and Hand Back

Never end this command with the repository in an in-between state. Before reporting, `MERGE_HEAD` must be gone and the developer must know exactly which branch they are standing on.

**If the merge happened directly in `<target>`** (no temp branch): report the final summary — files changed, commits merged in, branch name — and tell them it's ready to review and push. Done.

**If a temp branch was used** (Step 6, option 2): the merge is committed there, but `<target>` still doesn't have it. Ask (`AskUserQuestion`) how to bring it back:

1. `Merge into <target> now (Recommended)` — `git checkout <target>` then `git merge --ff-only <temp-name>`. This is guaranteed to fast-forward because the temp branch was created from `<target>` and `<target>` hasn't moved. If `--ff-only` fails, `<target>` moved underneath you — stop, do not force it, and tell the developer what happened.
2. `Keep <temp-name> for review / open a PR` — leave `<target>` untouched and stay on the temp branch. Say plainly that `<target>` does **not** yet contain the merge and name the exact command they'll need later (`git checkout <target> && git merge --ff-only <temp-name>`).

If option 1 was taken, also ask whether to delete the now-redundant temp branch (`git branch -d <temp-name>` — the safe `-d`, never `-D`). Default to keeping it if they don't care.

Finish with a final state report: current branch, whether `<target>` contains the merge, whether the temp branch still exists, and the fact that nothing was pushed.

## Safety Rules (apply throughout)

- **A skipped question ends the command.** If the developer presses Skip, or the answer comes back as `[No preference]` or empty, stop — never substitute the `(Recommended)` option, never infer the branch or resolution they "probably" wanted, never re-ask the same question in different words. What "stop" requires depends on where you are, because this command can be mid-operation:
  - **Steps 1–3** (branch selection, confirmation) — nothing has been touched. Stop outright and say so.
  - **Step 6** (conflict handling) or **Step 7 Case 3** (business-logic conflict) — a merge is in progress, so stopping is an action, not inaction. Do not guess a resolution and do not leave `MERGE_HEAD` dangling: run `git merge --abort` to return to the pre-merge state, verify with `git rev-parse -q --verify MERGE_HEAD` that it came back empty, then report that the merge was abandoned and nothing was resolved. If the abort itself fails, say so plainly and give the developer the exact command to run.
  - **Step 8** (bringing a temp branch back) — the merge is already committed on the temp branch, so there is nothing to abort. Stop, and report plainly that `<target>` does **not** contain the merge, naming the temp branch and the exact command to finish later (`git checkout <target> && git merge --ff-only <temp-name>`).
- Never leave the repository mid-merge. Every path out of this command ends with either a completed merge commit or an explicit `git merge --abort` — never a dangling `MERGE_HEAD`.
- Never leave the developer on a temp branch without telling them `<target>` doesn't have the merge yet and how to get it there.
- Never overwrite business logic automatically — only Case 1/2 conflicts get auto-resolved, and both must be genuinely unambiguous.
- Never discard developer code automatically — if in doubt whether something is dead code or intentional, treat it as intentional and ask.
- Always explain non-trivial decisions as you make them, not just at the end.
- Only ask the developer questions when a decision requires human/business judgment — don't ask about things you can determine yourself from the diff (e.g. simple import ordering).
- Never push to the remote as part of this command.
