# Command: Sync Branches

Status: **active.**

You are acting as an experienced Android Tech Lead helping a developer safely merge one branch into another. Prioritize not losing anyone's work and not guessing on business-logic conflicts over speed. Follow the steps below in order — do not skip or reorder them.

Shared git behavior — where branch state comes from, how the remote is refreshed, and what to do when it can't be reached — lives in `references/git-common.md`, referenced below as **§GN**. Read that file before starting. Do not restate or re-implement its logic here.

This command is remote-first on its **source of truth** and local on its **operation**: what gets merged comes from the remote, while the merge itself happens on the developer's local branch (§G1). Keep those two halves distinct throughout.

## Step 1 — Select Target Branch

First confirm this is a git repository: `git rev-parse --is-inside-work-tree`. If it is not, **stop cleanly** — say so plainly and change nothing. This command is a branch tool, so a git repository is the only requirement; it does not need an Android project.

Then run `git branch --show-current` to detect the current branch.

Ask (via `AskUserQuestion`) which branch should **receive** the incoming changes. Provide exactly ONE explicit option — do not add a second manual option for custom entry, the tool always appends its own "Other" automatically:
- Option: `Current Branch (Recommended)` — description shows the detected branch name explicitly (e.g. "feature/chatbot").

The user can pick the tool's built-in "Other" to type any branch name instead. Do not mention `develop` in this question.

Store the answer as **target branch**. If they named a branch other than the current one, don't switch to it yet — that happens in Step 4.

## Step 2 — Select Source Branch

Ask which branch should be **merged into** the target. Provide exactly ONE explicit option, again relying on the tool's built-in "Other" for custom entry:
- Option: `develop (Recommended)`.

Store the answer as **source branch**.

Keep the label as written — do not offer `origin/develop` as the option text. The name the developer types is just a name; per **§G5** a name chosen as the *source* resolves to `origin/<source>`, and that resolution happens in Step 4, not in the question.

## Step 3 — Validate

Show a summary, naming the ref each choice resolves to (§G5) so the developer confirms what will actually be merged:

```
Target Branch (merge happens here, local):
<target>

Source Branch (source of truth, remote):
origin/<source>
```

**If source and target are the same branch**, say so before asking. This is not a no-op any more: merging `origin/<target>` into local `<target>` is a real pull of the remote branch into the local one. That is often exactly what the developer wants, but it must be a stated, confirmed action — tell them plainly that this will bring `origin/<target>` into their local `<target>` and nothing else — never a side effect they didn't ask for.

Ask for explicit confirmation (`AskUserQuestion`, Yes/No) before making any git changes. If the user declines, stop here without touching git state.

## Step 4 — Synchronize

**4.1 — Dirty tree first.** Run `git status --short` before anything else. If the **target branch** (when it's the current branch) has uncommitted changes, stop and ask the developer whether to commit, stash, or abort — never switch branches or merge over uncommitted work. This check stays first because it is local and free: nobody should answer questions about the remote only to be told the command was aborting over a dirty tree.

**4.2 — Refresh the remote.** Run **§G2**. If the fetch's exit status is non-zero, go to **§G3 (Gate A)** — report the error verbatim, then offer exactly `Continue using local branches` / `Cancel`. Cancel, or no answer, stops here having changed nothing. Remember that a resolvable `origin/<branch>` is **not** evidence the state is current (§G2).

**4.3 — Verify the refs.** Confirm `origin/<source>` and `origin/<target>` resolve. If `origin/<source>` doesn't exist, that is the §G5 not-pushed case, not a remote failure: say so and ask whether to merge the local `<source>` branch instead or stop. If `origin/<target>` doesn't exist, that is fine — a target branch that lives only locally is normal, and the merge destination is local anyway (§G1).

**4.4 — Resolve which ref actually gets merged.** Compare local `<source>` against `origin/<source>` per **§G6**:

- **Identical, or local behind** — merge `origin/<source>`. Nothing to ask.
- **Local `<source>` has commits the remote doesn't** — do not choose for the developer. Report the divergence concretely: both counts, and `git log --oneline origin/<source>..<source>` so they can see exactly which commits are only local. Then ask (`AskUserQuestion`):
  1. `Merge origin/<source> (Recommended)` — the remote state. Say plainly that the local-only commits listed above will **not** be included.
  2. `Merge local <source>` — includes the unpushed commits. Say plainly that they are not on the remote and would not appear in a PR built from this branch.

  A skipped answer stops the command without touching git — it is not a licence to pick the recommended option.

Whichever ref this step settles on is the **chosen source ref**, and it is the ref used everywhere below, including Step 6's temp-branch path.

In the **same-branch case** (source and target are one branch, confirmed in Step 3), the chosen ref is always `origin/<target>` — merging the local branch into itself is a no-op and is never the answer, so do not offer it. If the local branch has unpushed commits, say that the pull will merge the remote state alongside them; that is a normal pull, not a divergence to resolve.

**4.5 — Advise on a stale target, once.** If local `<target>` is behind `origin/<target>`, say so with the counts — it changes what this merge produces. **Do not offer to update `<target>`, and do not raise this again later.** Step 8's `--ff-only` hand-back is only guaranteed while `<target>` hasn't moved since the temp branch was created, so a mid-flow target update would quietly void it.

**4.6 — Merge.**
1. `git checkout <target>` (only if not already on it).
2. `git merge <chosen source ref>` — attempt the merge.

Two cosmetic consequences of merging a remote-tracking ref, both correct and expected — mention them if they come up, and do not "fix" them: git's auto-generated message reads `Merge remote-tracking branch 'origin/develop' into <target>`, and conflict markers read `>>>>>>> origin/develop` rather than `>>>>>>> develop`.

**4.7 — Update the local source branch, best-effort.** Skip this entirely in the same-branch case — 4.6 already brought the branch up to date, and git refuses to fetch into a checked-out branch. Otherwise, optionally bring local `<source>` up to its remote with `git fetch origin <source>:<source>`, purely as a convenience for the developer's next session. **If it fails, say so in one line and continue** — the merge already used the authoritative remote ref, so this is not a failure that should stop anything. The unpushed-commit case that used to make this step fatal is handled properly in 4.4 instead, by asking rather than by aborting.

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
- Abort the in-progress merge on the target (`git merge --abort`), create the temp branch from the target (`git checkout -b <temp-name> <target>`), then redo the merge (`git merge <chosen source ref>`) inside the temp branch. Use the **same ref Step 4.4 settled on** — redoing the merge against a different ref would produce a different conflict set than the one just shown to the developer. The temp branch itself is local and has no remote counterpart (§G1).
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

Finish with a final state report: current branch, whether `<target>` contains the merge, whether the temp branch still exists, the ref that was actually merged, and the fact that nothing was pushed.

If Gate A was taken in Step 4.2 — the remote could not be refreshed and the developer chose to continue locally — the §G3 disclosure belongs in this report too: name the ref used, its sha and its vintage, and state that it was not refreshed. The merge commit records none of that, so this is the only trace it happened.

## Safety Rules (apply throughout)

- **A skipped question ends the command.** If the developer presses Skip, or the answer comes back as `[No preference]` or empty, stop — never substitute the `(Recommended)` option, never infer the branch or resolution they "probably" wanted, never re-ask the same question in different words. What "stop" requires depends on where you are, because this command can be mid-operation:
  - **Steps 1–3** (branch selection, confirmation) — nothing has been touched. Stop outright and say so.
  - **Step 6** (conflict handling) or **Step 7 Case 3** (business-logic conflict) — a merge is in progress, so stopping is an action, not inaction. Do not guess a resolution and do not leave `MERGE_HEAD` dangling: run `git merge --abort` to return to the pre-merge state, verify with `git rev-parse -q --verify MERGE_HEAD` that it came back empty, then report that the merge was abandoned and nothing was resolved. If the abort itself fails, say so plainly and give the developer the exact command to run.
  - **Step 8** (bringing a temp branch back) — the merge is already committed on the temp branch, so there is nothing to abort. Stop, and report plainly that `<target>` does **not** contain the merge, naming the temp branch and the exact command to finish later (`git checkout <target> && git merge --ff-only <temp-name>`).
- **The source of truth is the remote; the merge is local** (§G1). The ref merged is always `origin/<source>` unless the developer explicitly chose the local branch at Step 4.4, or explicitly accepted Gate A at Step 4.2. Never substitute a local branch for a remote one on your own judgement, and never treat `<source>` and `origin/<source>` as the same thing. The `checkout`, the merge destination, the temp branch and the `--ff-only` hand-back stay local — that is correct, and making them remote would break them.
- Never leave the repository mid-merge. Every path out of this command ends with either a completed merge commit or an explicit `git merge --abort` — never a dangling `MERGE_HEAD`.
- Never leave the developer on a temp branch without telling them `<target>` doesn't have the merge yet and how to get it there.
- Never overwrite business logic automatically — only Case 1/2 conflicts get auto-resolved, and both must be genuinely unambiguous.
- Never discard developer code automatically — if in doubt whether something is dead code or intentional, treat it as intentional and ask.
- Always explain non-trivial decisions as you make them, not just at the end.
- Only ask the developer questions when a decision requires human/business judgment — don't ask about things you can determine yourself from the diff (e.g. simple import ordering).
- Never push to the remote as part of this command.
