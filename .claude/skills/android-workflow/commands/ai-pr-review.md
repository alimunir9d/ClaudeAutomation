# Command: AI PR Review

Status: **active.**

You are acting as an experienced Android engineer reviewing a real production PR. The PR itself may not be reachable — assume there is no GitHub/GitLab API and no CI integration. The entire review is built from a git diff between two branches, computed in the local clone but measured between the clone's **remote-tracking refs**, because those are what the PR would actually be built from.

Shared git behavior — where branch state comes from, how the remote is refreshed, and what to do when it can't be reached — lives in `references/git-common.md`, referenced below as **§GN**. Read that file before starting. Do not restate or re-implement its logic here.

Review **only** the changes the source branch would introduce into the target branch. Do not review unrelated existing code unless you need it to understand the changed code or to spot a regression the change causes.

Prioritize correctness and meaningful risk over the number of comments. Follow the steps below in order — do not skip or reorder them.

## Step 0 — Preflight

This command is **read-only**. It never checks out, merges, commits, stashes, or pushes anything. Say so up front, so the developer knows it is safe to run mid-task.

0. `git rev-parse --is-inside-work-tree` — confirm this is a git repository. If it is not, **stop cleanly**, say so, and review nothing. This command is a diff tool, so a git repository is the only requirement; it does not need an Android project.
1. Refresh remote state per **§G2** — `git fetch origin --prune --tags`, so the review baseline isn't stale. This touches refs only, never the working tree. **Check the fetch's exit status:** non-zero → **§G3 (Gate A)**, which reports the error verbatim and offers exactly `Continue using local branches` / `Cancel`. A resolvable `origin/develop` is not evidence the state is current (§G2), so never skip the exit-status check because the refs look fine.
2. `git branch --show-current` — note the current branch for the Step 2 default.
3. `git status --short` — if the working tree is dirty, tell the developer plainly that **uncommitted changes are not part of this review** (the diff is branch-to-branch, so anything unstaged or uncommitted is invisible to it). Ask whether to continue anyway or stop and commit first. Never offer to stash or commit on their behalf.

   The same limitation extends one step further now that the review is measured between remote refs: **committed but unpushed commits are also invisible to it.** Step 2 detects that case explicitly and asks; don't pre-empt it here, but don't imply that committing alone is enough to be included either.

## Step 1 — Select Target Branch

Ask (via `AskUserQuestion`) which branch the changes would be merged **into** — the PR's base:

1. `develop (Recommended)`
2. `Other Branch`

If they pick `Other Branch`, ask a free-text follow-up for the name. Store as **target branch**.

## Step 2 — Select Source Branch

Ask which branch holds the changes under review — the PR's head:

1. `Current Branch (Recommended)` — show the detected branch name explicitly in the description (e.g. "task/conflict2").
2. `Other Branch`

If they pick `Other Branch`, ask a free-text follow-up for the name. Store as **source branch**.

Keep both question labels as written — a developer types a bare branch name, and the resolution to a ref happens here, after the answer (§G5).

### Resolve each branch to the ref the review is measured against

A PR is a comparison between two states on the remote, so that is what gets reviewed. Resolve per **§G5**:

**Target** (the PR's base) — always **`origin/<target>`**. Verify it with `git rev-parse --verify origin/<target>`. If it doesn't resolve, say which ref failed and re-ask the Step 1 question. **Never fall back to the local `<target>` branch** — a stale local base is the single most damaging input to this command: it moves the merge base backwards and makes the review blame the source branch for commits it received *from* the target in an earlier back-merge (§G1). That failure has no symptom, which is why there is no silent fallback for it.

**Source** (the PR's head) — prefer **`origin/<source>`**. Two cases where it isn't available even though the remote is perfectly reachable. Neither is a Gate A situation, and neither may be resolved silently:

- **`origin/<source>` does not resolve** — the branch was never pushed, the common case for a developer reviewing their own work before opening the PR. Say so plainly, then ask (`AskUserQuestion`): review the local `<source>` branch instead, or stop and push first. Reviewing local here is legitimate — there is no remote state for a branch that has none — it just has to be labelled all the way through to the output.
- **`origin/<source>` resolves but local `<source>` has commits it doesn't** (§G6) — show both counts and `git log --oneline origin/<source>..<source>`, then ask which state to review. Recommend `origin/<source>`, because a PR reviews what was pushed; if they choose local, say that those commits are not on the remote and would not be in the PR as it stands.

Call the two resolved refs **base-ref** and **head-ref**. If they are the same ref, there is nothing to review; say so and stop.

## Step 3 — Select Review Type

Ask which review to run:

1. `Generic PR Review` — full code review of the change.
2. `Events PR Review` — the PR's primary purpose is analytics/events.

## Step 4 — Optional Context

**If Generic:** ask "Do you want to provide additional PR context? This is optional." Let them either supply context or skip.

**If Events:** ask whether they want to point you at the relevant event-implementation file(s) or any event-naming reference, or skip. Read whatever they name.

Either way: context tells you what the change is *intended* to do. It is **never** evidence that the implementation does it. Verify intent against the actual code, and if the code contradicts the stated intent, that's a finding.

## Step 5 — Determine the Effective Diff

Use the **base-ref** and **head-ref** resolved in Step 2 — normally `origin/<target>` and `origin/<source>`. All four commands use the same pair; mixing a local ref into one of them produces a report whose commit list and diff describe different change sets.

Run, in order:

1. `git merge-base <base-ref> <head-ref>` — the fork point.
2. `git diff --stat <base-ref>...<head-ref>` — scope overview.
3. `git log --oneline <base-ref>..<head-ref>` — the commits in the would-be PR.
4. `git diff <base-ref>...<head-ref>` — the review material.

Use **three dots** for the diffs. `<base-ref>...<head-ref>` is the source branch's own work measured from the fork point; it excludes commits the target gained independently. Two dots would blame the source branch for changes it never made. (Command 3 is deliberately two-dot: it lists the commits, which is a different question from what the diff measures.)

Measuring both endpoints on the remote is what makes the fork point match the one GitHub computes for the PR. A local base ref that has fallen behind yields an *older* merge base, which silently widens the diff with commits that arrived from the target itself.

Two failure modes to handle rather than run past:

- **`git merge-base` fails** — the two refs have unrelated histories, and the three-dot diff will error too. Report it and stop; do not fall back to a two-dot diff to force output.
- **More than one merge base** (criss-cross history — check with `git merge-base --all` if command 1 looks ambiguous) — `...` silently picks one. Say so, name the bases, and note that the diff boundary is approximate rather than presenting it as exact.

If the diff is empty, report that the source branch introduces no changes to the target and stop. Do not manufacture findings.

Then scope what's worth reading. Deprioritize machine-generated and IDE churn — `.idea/`, `build/`, `*.iml`, lockfiles, generated resources, binary assets. Account for them in one line ("N IDE/config files changed, not reviewed") rather than reviewing them line by line. If such a file looks genuinely wrong to have in the PR, that's a single finding, not a category.

## Step 6 — Understand Before Judging

Before writing a single finding, work out:

- What the change is trying to achieve.
- Which files and components it affects.
- How the changed code is reached at runtime — who calls it, from which lifecycle stage or user action.
- What existing behavior it interacts with.
- Whether it introduces side effects nobody asked for.

Read changed files in full whenever the hunk alone is ambiguous. A diff hunk hides its own context; reviewing one without opening the file is how false findings get written. Reading untouched code is fine **only** to understand the change or to catch a regression it causes.

## Step 7 — Review

### If Generic PR Review

Work through these, reporting only what the diff actually supports:

- **Code quality** — readability, maintainability, unnecessary complexity, duplication, naming, and consistency with conventions **already present in this project**.
- **Android architecture** — separation of concerns, responsibility placement, lifecycle awareness. Judge against the architecture the project already uses; do not impose Clean Architecture or MVVM on a codebase that isn't built that way.
- **Kotlin / concurrency** — coroutine usage and scoping, threading, lifecycle-tied work, race conditions, unsafe shared mutable state.
- **Logic & state** — incorrect state transitions, wrong business logic, unhandled edge cases, null safety, unstated assumptions, crash-prone paths.
- **Performance** — repeated or unnecessary work, inefficient processing, avoidable allocations, leaked references (Context, listeners, observers), expensive work on the main thread.
- **Resources** — newly introduced user-visible values that belong in `strings.xml`, dimens, colors, or drawables instead of inline literals. Ignore hardcoded strings used only for logging or debugging unless they cause an actual problem (e.g. leaking sensitive data into logs).
- **Constants** — magic numbers, hardcoded comparison values, constants duplicated across files.
- **Security / privacy** — sensitive data exposure, unsafe handling of user data, insecure patterns.
- **Regression risk** — whether the change could disturb existing behavior that wasn't part of its goal.
- **Testing** — meaningful missing scenarios. Do **not** report a finding merely because a test is absent; report it when the changed behavior carries real risk that a test should reasonably cover.

### If Events PR Review

Review the diff with analytics correctness as the focus:

- **Event triggering** — fires from the correct user action; not fired prematurely; not fired after a failed operation; not silently skipped on some paths.
- **Duplicate events** — can the same event fire more than once via repeated callbacks, lifecycle changes, recomposition, re-registered observers, configuration changes, navigation, retries, or multiple listeners?
- **Event names** — consistent with the conventions already used in this project and with whatever SDK the diff reveals (infer the SDK — Firebase, GA4, Amplitude, something in-house — from the code and any files supplied in Step 4; do not assume one).
- **Parameters** — required params present, names consistent, values correct and of the expected type, consistent with sibling events.
- **Existing analytics** — whether any already-working event was altered as a side effect.

#### Critical events rule

The purpose of this PR is analytics. Therefore verify explicitly that adding analytics has **not** changed how the app behaves. Check for unintended changes to:

UI behavior · navigation · business logic · state management · execution flow · timing or ordering of operations · existing functionality · side effects

If an event implementation also changes application logic, **report it even if it looks trivial** — a reordered call, an added early return, a moved null check, a changed nullability. Do not assume such a change was intentional; ask. Analytics should be additive and isolated, and a behavioral change riding along inside an analytics PR is exactly what a reviewer is there to catch.

## Step 8 — Filter the Findings

Before reporting, cut anything that doesn't survive these rules:

- Only findings the visible code supports. Do not invent surrounding behavior to make a finding work.
- Do not report subjective style preferences as defects unless they break an existing project convention or create a real maintainability problem.
- Prioritize real issues over minor suggestions. Do not flood the developer with low-value findings.
- When a finding depends on business intent the code cannot reveal, ask a question instead of declaring the implementation wrong.

## Step 9 — Output

### What was reviewed

Open with one line naming the two refs the review was built from — `origin/develop...origin/task/chatbot`, for instance. It is the difference between a review of the PR and a review of something adjacent to it, and the reader can't tell from the findings themselves.

If either side was **local** — the developer accepted Gate A in Step 0, or chose the local branch in Step 2 — that line becomes an explicit disclosure: which ref was local, why (not pushed / remote unreachable), and that the review may therefore not match what the PR would show. Per §G3, when it was Gate A, include the ref's sha and vintage. **Carry this disclosure into the ready-to-post comment block below as well** — that block gets pasted into a real PR thread, where a review of unpushed code presented as a review of the PR is actively misleading.

### Findings

For each meaningful finding:

- **Severity** — Critical / High / Medium / Low
- **File** — as a clickable `path:line` link
- **Line** — approximate is fine, when identifiable; use the post-change (new-side) line number so the link lands in the right place
- **Description** — what the code does
- **Why it matters** — the concrete consequence
- **Suggested fix**

Order by severity, highest first.

### Ready-to-post comment

Then produce **one consolidated PR review comment** in a fenced block so it can be copied straight into the PR thread, plus a short standalone snippet for each Critical/High finding for inline threads.

The comment must be conversational, professional, and collaborative; grounded only in visible code; phrased as a question or discussion point; free of absolute claims. Explain the concern, then ask how the scenario is handled.

Preferred shape:

> "Looking at this implementation, it seems this flow could also execute when ____. How is this scenario being handled? I may be missing some surrounding context, but from the current changes it looks like this could potentially lead to ____."

For events:

> "From the current implementation, it looks like this event could also be triggered when ____. Is that intentional, or is this path handled elsewhere?"

> "I noticed this change also affects the existing execution flow in addition to adding analytics. Was this behavioural change intentional, or should the analytics implementation remain isolated?"

Avoid: "This is wrong." · "Please fix this." · "This will crash."

### If nothing significant was found

Say so explicitly rather than going quiet. For Generic, state that no major code quality, architecture, logic, performance, security, or regression concerns were found. For Events, state each of these plainly:

- No duplicate or missing event issues were identified.
- Event naming and parameters appear consistent.
- No unintended behavioural changes were observed.
- No major code quality or architecture concerns were found.

## Safety Rules (apply throughout)

- **A skipped question ends the command.** If the developer presses Skip on any of Steps 1–3, or the answer comes back as `[No preference]` or empty, stop and say so — never fall back to `develop (Recommended)` or `Current Branch (Recommended)`, never guess the review type, and never produce a review nobody scoped. Nothing needs cleanup, since the command is read-only. **One exception:** Step 4's optional-context question explicitly offers skipping as a valid answer meaning "no additional context" — that one continues the review as normal.
- **Read-only, always.** No `checkout`, `merge`, `rebase`, `commit`, `stash`, `reset`, or `push`. `git fetch origin --prune --tags` is the only command that writes anything, and it writes only remote-tracking refs and tags — never the working tree or a local branch. The developer's branch and working tree must be exactly as they were when the command started.
- **Remote refs are the baseline** (§G1). Never substitute a local branch for a remote one without an explicit developer choice — not for the target under any circumstances, and for the source only via Step 2's questions or Gate A. `<source>` and `origin/<source>` are different refs and are never treated as interchangeable.
- Never review uncommitted work silently — a branch diff cannot see it, so say so in Step 0 instead of letting the developer assume it was covered. The same goes for committed-but-unpushed work when the review is measured on remote refs: Step 2 asks about it rather than quietly excluding it.
- Never treat developer-supplied context as proof of correctness.
- Never state as fact something the diff doesn't show. If evidence is missing, ask.
- Do not fix anything. This command reports; the developer decides. If they ask for fixes afterwards, that's a separate, explicit request.
