# Command: Sync Strings — Fast

Status: **active.**

You are acting as an experienced Android localization engineer performing an incremental, token-efficient synchronization of string resources. This mode solves the same problem as `sync-strings-full.md` but deliberately avoids scanning the entire resource set.

**It depends on a workflow convention.** Before running this mode, the developer places all newly added, untranslated English strings at the **bottom** of `values/strings.xml`. Treat that as an intentional convention the developer is responsible for upholding — this mode trusts it rather than auditing the file to verify it.

The purpose is to minimize token consumption, file reading, and processing time. What it must **not** minimize is correctness: every candidate key is still checked independently against every supported language.

All shared behavior lives in `references/strings-common.md`, referenced below as **§N**. Read that file before starting. Do not restate or re-implement its logic here.

Follow the steps below in order — do not skip or reorder them.

## Step 0 — Locate the string resources

Run **§1 — Locating the string resources**, but by directory listing only — glob the paths, do **not** read any file's contents yet.

Report the English file, the locale files found, and any `values-*` directory excluded as non-locale.

## Step 1 — Read only the tail of the English file

Read the **end** of `values/strings.xml` only — a **chunk** of roughly the last 12 lines, using an offset rather than a full read. Never read the whole file up front; that alone would defeat the mode.

Twelve lines is about ten candidates, which matches the convention this mode relies on: a handful of new strings at the bottom. Keep the chunk small on purpose. Per the cost model in the Safety Rules, an oversized window sweeps *already-translated* keys into the Step 3 query, and those are the only keys that cost anything.

If the chunk yields no synchronization boundary (Step 4), extend upward by one more chunk of similar size and repeat. Read more only when the boundary has not yet been found — never speculatively, and never "while you're here."

## Step 2 — Extract the candidate keys

From the **current chunk only**, extract the string keys in **bottom-most-first** order per **§2**. These are the candidates: the strings the convention says are newly added.

Skip anything marked `translatable="false"`.

The list is scoped to this chunk, never to the whole file. Naming a key here asserts nothing about whether it is translated; that question is asked once, for the whole chunk, in Step 3.

## Step 3 — Presence-check the chunk in one query

Do **not** read the locale files to find out which keys they contain. Query them instead.

Run a **single `Grep` covering this chunk's candidates**, all of them at once:

- Pattern: `name="(key_a|key_b|key_c)"` — an alternation over the chunk's keys
- Glob: `**/res/values-*/strings.xml`
- `output_mode: "content"` with line numbers on.

One call returns which locale files contain which of those keys, and where. Restrict the results to genuine locale directories from Step 0; a match inside a non-locale `values-*` directory does not count as a translation.

**The batch never extends beyond the current chunk.** Do not widen the pattern toward the whole file, do not pre-scan candidates from chunks you have not read, and do not re-query keys already resolved. The chunk is the unit precisely because it is small: a tight window cannot reach far above the boundary, so batching inside it is cheap *and* bounded, whereas batching across the file re-creates the bug where already-translated keys are swept in and the boundary becomes decorative.

**Use the glob exactly as written, including the leading `**/`.** A glob such as `values-*/strings.xml` silently matches nothing, which looks identical to "these keys are translated nowhere" and would cause every candidate to be reported as missing from every language. If a query returns no matches at all, verify the glob with a key you know exists before believing the result.

## Step 4 — Walk upward and find the boundary

Walk this chunk's results **bottom-up**:

- **Missing from one or more languages** → it needs work. Record exactly which languages lack it and move to the next key up.
- **Present in every supported language** → **stop.** This is the synchronization boundary.

If the chunk contains no fully-translated key, extend by one more chunk (Step 1) and query again (Step 3). Otherwise stop — do not read the rest of the English file.

A key above the boundary is not "skipped" — as far as this command is concerned it was never looked at. Appearing incidentally in a chunk's query results is **not** licence to act on it, translate it, or mention it in the summary.

Two things about this boundary, because they are what keeps the optimization honest:

**Token efficiency never overrides correctness.** A key is evaluated against *every* language independently (§3). If it exists in seven languages and is missing from one, it is missing — translate it for that one. Never skip a key because it exists in some of the languages, and never infer a key's status from its neighbors'.

**The degenerate case is reported, not swallowed.** If the very bottom-most key is already present in every language, the scan stops immediately having synchronized nothing. Under this mode's boundary rule that is a legitimate result — but it is also precisely where the heuristic is blind, since nothing above was examined. Say so explicitly: name the boundary key, state that no keys above it were checked, and recommend `Sync Strings — Full` if the developer wants certainty. Do not report it as a clean "everything is in sync."

If the boundary cannot be established safely — the window is exhausted without ever finding a fully-translated key, or the file's structure doesn't match the convention — say so and recommend Full mode rather than expanding this scan into a de-facto full audit.

## Step 5 — Sample the target files' style cheaply

Before translating, read only the **tail** of each target locale file — roughly the last 30 lines — to pick up the terminology and register §4 requires you to follow. Do not read these files in full.

This is the deliberate tradeoff of the mode: enough context to match the file's existing voice, without paying for the whole file.

## Step 6 — Translate and write

Translate per **§4** — placeholders preserved with the same count and indices, escaping and inline markup intact, positional forms when word order changes.

Write per **§5** — appended before `</resources>`, in English order, matching existing formatting, with each key's absence verified immediately before it is appended.

**One edit per locale file.** Append all of a file's missing strings in a single edit rather than editing once per string; ten strings across eight languages is 8 edits, not 80.

Confirm with the developer before writing (`AskUserQuestion`), showing which keys go into which files. If they skip that question, stop per **§0**.

Anything ambiguous gets no translation — record it per **§6**. The cheap call-site grep is still worth doing for the handful of strings actually being translated here; it is bounded work and usually resolves the question.

## Step 7 — Summary

Produce the summary per **§7**: strings synchronized, languages updated with counts, and anything requiring human review.

Then add this mode's **mandatory boundary disclosure**, which is not optional and not abbreviated:

- The key and line number in `values/strings.xml` where the scan stopped.
- An explicit statement that keys **above** that point were not examined.

A Fast run that reports what it synchronized without reporting where it stopped is misleading, because it reads as full coverage.

## Safety Rules (apply throughout)

All of **§8 — Safety rules** applies. The ones this command is most likely to be tempted to break:

- **A skipped question ends the command** (§0).
- **Batch within the chunk, never beyond it.** One query per window chunk is correct and cheap. Widening the pattern toward the whole file re-creates the bug where already-translated keys get swept in and the boundary becomes decorative — controlling what is written while everything is examined anyway. Chunks stay small so the batch cannot reach far above the boundary.
- **Optimize the right variable.** The cost model is not intuitive, so it is written down: a `Grep` costs only its *matches*, which makes a key that is absent everywhere effectively free to include in a pattern, while an already-translated key costs one line per locale. A `Read` costs every line it returns, translated or not. Therefore: keep read windows tight, and do not spend round trips splitting a query to avoid keys that cost nothing.
- **Token optimization never sacrifices translation correctness.** Every candidate key is verified independently in every language. "It's probably fine, it exists in most locales" is the failure this rule exists to prevent.
- **Never expand into a full audit** to feel thorough. If the optimized scan cannot safely determine where to stop, say so and recommend Full mode — that is the escape hatch, not silently reading everything.
- **Always disclose the stop point.** Coverage that isn't stated is coverage that will be assumed.
- **Never modify the English source**, never delete strings, never create duplicate keys, never reformat unrelated content (§5, §8).
- **Never guess an ambiguous translation** (§6).
