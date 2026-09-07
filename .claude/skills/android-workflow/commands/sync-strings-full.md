# Command: Sync Strings — Full

Status: **active.**

You are acting as an experienced Android localization engineer performing a complete synchronization and audit of the project's string resources. The English `strings.xml` is the source of truth; the goal is that **every English key has a translation in every supported language**.

This mode is a complete audit, not a quick scan. **Accuracy and safety outrank token consumption here** — read as much context as you need to make each translation decision correctly. If you find yourself wanting to skip a file to save tokens, you are in the wrong mode; that is what `sync-strings-fast.md` is for.

All shared behavior lives in `references/strings-common.md`, referenced below as **§N**. Read that file before starting. Do not restate or re-implement its logic here.

Follow the steps below in order — do not skip or reorder them.

## Step 0 — Locate the string resources

Run **§1 — Locating the string resources**.

Report the inventory before doing anything else:

- The English source file.
- Every locale file found, with the language each qualifier resolves to.
- The module and source set in use, and — if the project had more than one candidate — that the developer chose it.
- Every `values-*` directory **excluded** as non-locale, naming the qualifier type. Stating the exclusions out loud is what stops a UI-mode or density qualifier such as `values-night` from being treated as a language.

If a locale directory exists without a `strings.xml`, note it now — per §5 that is a project decision, not something this command fixes silently.

## Step 1 — Read the English source in full

Read `values/strings.xml` completely and build the full key list per **§2**: `<string>`, plus `<plurals>` and `<string-array>` if present.

Note which entries are marked `translatable="false"` (excluded) and which are product/brand names that will be copied verbatim rather than translated.

## Step 2 — Read every locale file in full

Read each locale `strings.xml` completely and build the key-presence matrix per **§3** — key × language, evaluated independently for each language.

Reading each file in full is deliberate here. It is what lets this mode catch the things a partial scan cannot: duplicate keys, orphan keys, and placeholder mismatches in translations that already exist.

## Step 3 — Report the gap, then confirm

Before editing anything, show the developer:

- Total English keys, and how many are fully synchronized.
- Each missing key and the languages it is missing from.
- Per-language totals of what would be added.

Ask for confirmation (`AskUserQuestion`) before any file is written. If they skip the question, stop per **§0**.

If nothing is missing, say so plainly and skip to Step 7 — there is still an audit worth reporting.

## Step 4 — Resolve context for ambiguous strings

For any key whose translation is not determined by the English text alone, do the call-site lookup described in **§6** — grep `R.string.<key>` and `@string/<key>` and read the surrounding line to learn whether it is a button, a label, a `contentDescription`, or a toast.

This step is why Full mode costs more, and it is where most of that cost earns its keep: a string's UI role usually settles the part-of-speech and register questions that would otherwise force a human-review flag.

## Step 5 — Translate

Translate each missing string per **§4**, into each language that lacks it.

Learn the register and terminology from entries already in the target file before writing new ones. Anything still ambiguous after Step 4 gets no translation — record it for the summary per §6.

## Step 6 — Write the changes

Apply the additions per **§5** — appended before `</resources>`, in English order, matching the file's existing formatting, with the key's absence verified immediately before each append.

Do not touch `values/strings.xml`. Do not modify, reorder, or reformat any existing entry in any file.

## Step 7 — Summary and audit findings

Produce the summary per **§7**: missing keys, languages updated with counts, strings requiring human review with reasons.

Then add the audit findings this mode is uniquely positioned to catch, since it read every file in full:

- **Duplicate keys** within any single file — a build-breaking error.
- **Orphan keys** — present in a locale file but absent from English.
- **Placeholder mismatches in existing translations** — a different placeholder count or different indices than the English original. This is a live crash risk and worth calling out even though this command did not create it.
- Locale directories missing a `strings.xml`, and `values-*` directories excluded as non-locale.

Report these; do **not** fix them. They are pre-existing conditions outside this command's mandate, and a synchronization run silently rewriting existing translations is exactly what §8 forbids.

## Safety Rules (apply throughout)

All of **§8 — Safety rules** applies. The ones this command is most likely to be tempted to break:

- **A skipped question ends the command** (§0). No selection is not permission to proceed with the changes you already computed.
- **Never modify the English source.** It is the reference, not a sync target.
- **Never fix what you only came to audit.** Orphans, duplicates, and pre-existing placeholder mismatches get reported, never rewritten.
- **Never guess an ambiguous translation** (§6) — even in Full mode, where the temptation is strongest because completeness feels like the goal. Coverage is the goal; fabricated coverage is not coverage.
- **Never treat "present in some languages" as done** (§3).
- **Appended lines only** (§5) — no reordering, no reformatting, no deletions.
