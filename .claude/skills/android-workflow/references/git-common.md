# Reference: Git Common

Shared git behavior for every Android Workflow command that reads **branch contents** to make a decision — `commands/sync-branches.md`, `commands/pull-from-develop.md`, `commands/ai-pr-review.md`, `commands/dev-to-main-pr.md`, `commands/create-release.md`.

This file is **not** a menu command. It is never dispatched to directly; it is pulled in by a command file that says "run §GN". Each section is addressable by number so the command files reference behavior instead of restating it.

**Sections here are numbered `§G0`–`§G6`, with a `G` prefix.** `references/release-common.md` owns `§0`–`§9` and `references/strings-common.md` owns `§0`–`§8`. A command that reads two references must never have to guess which file a bare `§2` belongs to, so every citation of this file carries the `G`.

The one-sentence version: **when a branch's contents are the basis for a comparison, a review, a merge, or a release decision, that basis comes from the remote — and if the remote can't be reached, the developer is told and asked, never silently substituted for.**

## §G0 — Scope and gate routing

This file governs commands that read branch *contents*. It does **not** govern commands that only read and write files in the developer's working project.

| Command | Remote-first | Gate if the refresh fails |
|---|---|---|
| `sync-branches` | yes | **§G3** (Gate A) |
| `pull-from-develop` | yes | **§G3** (Gate A) |
| `ai-pr-review` | yes | **§G3** (Gate A) |
| `dev-to-main-pr` | yes | **§G4** (Gate B) |
| `create-release` | yes | **§G4** (Gate B) |
| `sync-strings-full` / `sync-strings-fast` | **no — purely local** | none; never asks about remote access |
| `design-xml-constraint` / `design-xml-optimal` / `design-compose` | **no — purely local** | none; never asks about remote access |

Each command cites exactly one gate. Look it up here rather than deciding at runtime which gate applies — the moment a fetch has just failed is the worst moment to be classifying the command you are running.

**Purely local commands are genuinely out of scope.** A command that operates on the working project's own files must not fetch, must not read `origin/...` refs, and must never ask the developer about remote access or repository state. Sync Strings is the standing example: it compares `strings.xml` files on disk, so there is no branch content in its decision and nothing here applies to it. See `references/strings-common.md`. Designs is the same case — it implements a design into layouts and Composables on disk. See `references/design-common.md`.

Growing the menu: a new command that reads branch contents adds a row here and cites one gate. A new purely-local command adds a row saying so. "Remote is the source of truth" is not a blanket rule that every skill inherits — it is a rule about *where branch state comes from*, and it binds only the commands that use branch state.

## §G1 — Remote is the source of truth; local is the working copy

`develop` and `origin/develop` are two different refs. They are never interchangeable, and a local branch is never assumed equivalent to the remote branch of the same name.

| Role | Ref to use |
|---|---|
| Comparison / review baseline | `origin/<branch>` |
| Latest `develop` / `main` state | `origin/develop`, `origin/main` |
| PR source and target state | `origin/<source>`, `origin/<target>` |
| Release, version and tag state | `origin/main` |
| The branch a local merge is *performed on* — `checkout` target, merge destination | **local** branch |
| A temporary merge branch (`sync-branches` Steps 6 and 8) | **local** only — it has no remote counterpart |
| Uncommitted work | the working tree |

A bare branch name used as a **baseline** is a bug. The reason, not just the rule:

- A local branch can be days behind its remote. In this clone local `develop` routinely sits several commits back. A review or release built from it describes code nobody has.
- Worse, it fails *quietly and plausibly*. A stale local target moves the merge base **backwards**, and a three-dot diff measured from a moved merge base attributes commits the source branch received *from* the target in an earlier back-merge to the source branch itself. The output looks like a normal review of a slightly larger change — there is no error, and nothing signals that the baseline was wrong.

The distinction cuts the other way too. A command may legitimately operate on the developer's **local working branch** — that is where a merge actually happens, where a `checkout` lands, where uncommitted work lives. Remote-first does not mean "never touch a local branch"; it means the *state a decision is based on* comes from the remote, while the *operation* may still be local. Both halves matter: erasing the second half is how a branch tool stops being able to merge anything.

## §G2 — Refreshing remote state, and what freshness actually proves

The canonical refresh, everywhere in this tree:

```bash
git fetch origin --prune --tags
```

Run it before any decision that reads branch contents, **even if the session already fetched** — a stale ref produces a wrong diff, a wrong version, or a tag on the wrong commit, and re-fetching costs almost nothing.

**`git rev-parse --verify origin/<branch>` does not prove freshness.** It succeeds on a remote-tracking ref left in `.git` by any previous fetch, however old. It answers "does this ref exist locally", not "is this ref current". An offline machine with cached refs passes it and proceeds against days-old data — which is exactly the silent fallback this file exists to prevent.

So freshness is established one way only: **the fetch's own exit status.**

1. Run the fetch. **Check its exit code.**
   - Non-zero → go to the gate this command is assigned in **§G0**. Do not continue. Do not treat cached refs as a result.
   - Zero → the remote-tracking refs are current; continue.
2. Then verify each ref the command needs resolves: `git rev-parse --verify origin/<branch>`. If one doesn't, name it and stop or re-ask — never guess at a near-match (`dev` for `develop`, `master` for `main`).
3. `git ls-remote --exit-code origin HEAD` is the confirming probe when you need to distinguish "the remote is unreachable" from "the fetch failed for some other reason". Use it to describe the failure accurately, not as a substitute for step 1.

Two things that are **not** remote-availability failures and must not trigger a gate:

- `--prune` deleting `origin/<branch>` because the branch was removed from the remote. The fetch succeeded. This routes into §G5's "not on the remote" case.
- A branch that was never pushed, so `origin/<branch>` never existed. Also §G5.

## §G3 — Gate A: refresh failed, local comparison permitted

**Cited by `sync-branches`, `pull-from-develop`, `ai-pr-review`** — commands whose output is a comparison or a local merge, where a clearly-labelled local result still has value.

This is the sequence that is **forbidden**:

```
remote unavailable  ->  quietly use the local branch  ->  continue as if everything is correct
```

This is the sequence that is **required**:

```
remote unavailable  ->  tell the developer  ->  ask  ->  use local only if they chose it
```

Concretely:

1. **Report the failure verbatim.** Show git's actual error text. Do not paraphrase it, summarize it, or replace it with "couldn't reach the remote" — the real message is what tells the developer whether this is DNS, auth, a VPN, or a typo'd remote.
2. **Name what is affected.** Which comparison, which refs, and plainly: the local branches may be stale, so the result may differ from the current remote state.
3. **Ask** via `AskUserQuestion`, with exactly these two options:
   - `Continue using local branches`
   - `Cancel`
4. **Cancel stops the command and changes nothing.** So does no answer at all — Skip, `[No preference]`, empty, or a declining answer ("no", "stop", "cancel"). **A skipped gate is a Cancel, never a Continue.** This is the same rule the commands already apply to every other question.
5. **On Continue**, proceed with local refs — and carry a one-line disclosure into every subsequent report, including the final summary and any output meant to be copied elsewhere.

**A failed fetch leaves three possible states, not two:** no cached `origin/<branch>` at all, a cached-but-stale `origin/<branch>`, or (had it succeeded) a current one. Because of the middle case, "continue with local branches" is ambiguous on its own. The disclosure must therefore name **the exact ref used, its sha, and its vintage**:

```bash
git log -1 --format="%h %cr" <ref>
```

— and state that it was not refreshed. Nothing else records this: a merge commit carries no note that its source was stale, and a review has no metadata. The disclosure is the only trace, so it is not optional and it does not get trimmed for brevity.

## §G4 — Gate B: refresh failed, no local substitute exists

**Cited by `dev-to-main-pr` and `create-release`.** These commands write to the remote — a release-preparation commit, a PR, a tag, a GitHub Release. A local ref is not a degraded input here, it is the wrong input: a release built from a stale local ref tags the wrong commit and ships code nobody reviewed. This restates the contract already at `references/release-common.md` §1 and §9; it is not a new rule.

1. **Report the failure verbatim**, as in §G3.
2. **Ask** via `AskUserQuestion`, with exactly these two options:
   - `Retry`
   - `Cancel`
3. Retry re-runs §G2 from the top. Cancel — or no answer — stops the command with nothing created.

**Never offer local branches here.** Not as a third option, not as a suggestion afterwards, and not by improvising a local-git path that reaches the same outcome.

**Gate B does not route to `release-common.md` §8.** That fallback exists for a *different* failure: `gh` is missing or unauthenticated while the fetch itself worked. §8 runs version detection and release-notes generation on the way through, and both read remote refs (`git show origin/develop:README.md`, `git log origin/main..origin/develop`). If the fetch is what failed, §8 would hand the developer a paste-ready README and a set of release notes built from stale data — worse than stopping, because it looks finished. Say plainly: **without a successful fetch there is no manual handover either.**

## §G5 — Resolving a developer-supplied branch name

Developers answer branch questions with bare names — `develop`, `feature/chatbot`. Question labels stay that way on purpose; they are what a developer recognizes, and inviting `origin/` into a free-text answer produces `origin/origin/develop`. The name becomes a ref **here**, after the answer, and which ref depends on the **role** it was chosen for:

| Chosen as | Resolves to |
|---|---|
| Source branch, comparison baseline, review target/base | `origin/<name>` |
| Merge destination, `checkout` target, temp branch name | local `<name>` |

State the resolution to the developer once, so they can see which ref each name became. Do not restate the bare name later as though it were the ref that was used.

Two cases where `origin/<name>` is unavailable **even though the remote is perfectly reachable**. Neither is a remote-availability failure and neither triggers §G3 or §G4:

- **`origin/<name>` does not exist.** The branch was never pushed — the common case for a feature branch the developer is reviewing before opening a PR — or `--prune` just removed it because it was deleted on the remote. Say which of the two it is if you can tell. Then let the calling command ask what to do. Working from a local-only branch is legitimate here; it simply has to be labelled, because there is no remote state to be the source of truth for a branch that has none.
- **`origin/<name>` exists but local `<name>` has commits it doesn't.** See §G6.

## §G6 — Divergence between a local branch and its remote

The remote is reachable and both refs resolve; they just disagree. **Never resolve this silently in either direction.** Measure it:

```bash
git rev-list --left-right --count origin/<branch>...<branch>
```

The left count is commits only on the remote (local is behind); the right count is commits only local (unpushed work).

- **Left non-zero, right zero** — local is simply behind. The remote ref is the source of truth and there is nothing to ask.
- **Right non-zero** — the developer has unpushed commits that **will not** be part of a remote-based comparison or merge. Say so explicitly, list them with `git log --oneline origin/<branch>..<branch>`, and let the calling command gate on it. Do not quietly include them and do not quietly drop them: excluding a developer's own work without telling them is the same class of silent substitution as using a stale ref.

Report both counts as concrete numbers, not as "the branches differ".
