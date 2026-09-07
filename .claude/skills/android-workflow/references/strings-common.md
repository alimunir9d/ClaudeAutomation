# Reference: Strings Common

Shared utilities for the two Sync Strings commands — `commands/sync-strings-full.md` and `commands/sync-strings-fast.md`.

This file is **not** a menu command. It is never dispatched to directly; it is pulled in by a command file that says "run §N". Each section is addressable by number so the command files reference behavior instead of restating it. If you find yourself writing comparison, translation, or file-writing logic inside a command file, it belongs here instead.

Both commands are Android-specific and operate on `strings.xml` resources. **English is always the source of truth.**

**Both commands are purely local.** They read and write files in the developer's working project and nothing else. They do not use git, do not read remote branches, do not fetch, and must never ask the developer about remote access or repository state. The remote-first rule in `references/git-common.md` does **not** apply here: it governs commands whose decisions are based on the contents of a branch, and these commands compare resource files on disk. "English is the source of truth" is a statement about which *file* wins, not about which *ref* wins — do not generalize the tree's git rules onto it, and do not add a remote-availability gate to a workflow that never needed the network.

## §0 — Answering questions and skipping

Every `AskUserQuestion` in these workflows is a real gate. If the developer does not select an option, **stop the entire command immediately.**

**What counts as no selection:** they press Skip; the answer comes back as `[No preference]` or empty; or they answer by declining — "none", "neither", "stop", "cancel". These are all the same signal. Skipping is a withdrawal from the decision, not a delegation of it to you.

**What to do:** end the command. Make no further tool calls on its behalf.

**What never to do:**

- Do not fall back to the `(Recommended)` option. That label is a suggestion for someone who is choosing; it carries no authority when nobody chose.
- Do not infer the "obviously applicable" answer from project state, even when only one option could possibly work.
- Do not re-ask the same question in different words, and do not split it into smaller questions to get an answer indirectly.
- Do not continue with a narrowed or "safe subset" version of the workflow.

**What to say:** one short line naming the question that was skipped and stating that the workflow stopped. Do not re-litigate the choice or append an offer to continue.

**Disclose what already happened.** If any file was already written before the skip, list it. "Stopped" must never be read as "nothing changed" when something did.

**Resumption:** the workflow restarts only when the developer invokes it again, from the beginning.

## §1 — Locating the string resources

**Confirm this is an Android project first.** These commands are Android-specific and must not act anywhere else. Before anything, check for both:

- a Gradle project root — `settings.gradle.kts` or `settings.gradle` (or, failing that, a `build.gradle.kts` / `build.gradle`), and
- at least one Android resource tree — a glob over `**/res/values/strings.xml` returning a result.

If either is missing, **stop cleanly**: say what was looked for, what was found instead, and that nothing was modified. Do not fall back to searching for XML elsewhere, and do not offer to create a resource tree. A skill installed machine-wide will be invoked in non-Android projects by accident, and a clean stop is the correct outcome.

**The English source of truth is the unqualified `res/values/strings.xml`.** Always. If a `values-en/` directory exists it is a *locale override* for English, not the source — treat it as one more target language, never as the origin.

Find candidate files with a glob over `**/res/values*/strings.xml`, then classify each directory by its qualifier.

**Write locale globs with their leading `**/`.** A pattern such as `values-*/strings.xml` matches nothing, and it fails *silently* — the empty result is indistinguishable from "this key is translated nowhere." In a sync command that misreading is expensive: it reports every key as missing from every language and appends duplicates of strings that already exist. Whenever a locale lookup comes back with zero matches, verify the glob before believing the answer.

**Locale directories** (targets for translation) match Android's locale qualifier grammar:

- `values-<lang>` — two- or three-letter ISO 639 code: `values-es`, `values-ur`, `values-fil`
- `values-<lang>-r<REGION>` — with region: `values-pt-rBR`, `values-zh-rTW`
- `values-b+<lang>+<script>+<region>` — BCP-47 form: `values-b+sr+Latn`, `values-b+zh+Hans+CN`

**Everything else is not a language** and must be excluded, even if it contains a `strings.xml`. Common non-locale qualifiers: `values-night`, `values-v21` (and any `-v<N>`), `values-sw600dp`, `values-w820dp`, `values-land`, `values-port`, `values-hdpi` / `-xhdpi` / `-xxhdpi`, `values-television`, `values-round`. These look exactly like locale directories to a naive filter: `values-night` is a UI mode, not a language, and a filter of "any `values-*`" would wrongly treat it as one. Classify by the qualifier grammar above, never by the presence of a hyphen.

When a directory carries both a locale and a non-locale qualifier (`values-es-night`), it is a locale *variant*, not the base translation for that language. Do not sync into it unless it already contains the key being synced; report it in the summary instead.

Resolve each qualifier to a language name before translating (`es` → Spanish, `ur` → Urdu, `pt-rBR` → Brazilian Portuguese). Never translate from the folder name alone if the mapping is unclear — flag it per §6.

**Detect the module and source-set layout — never assume it.** Android projects range from a single `app` module to dozens of feature modules, and these commands must adapt to whichever they are run in:

1. Group the globbed files by the module and source set they belong to — the path segments before `/src/<sourceSet>/res/`.
2. If exactly one module–source-set pair contains `res/values/strings.xml`, use it and say so in the inventory.
3. If more than one does, **ask** (`AskUserQuestion`) which to synchronize, listing each with its resource counts. Do not guess, do not pick the largest, and do not pick the one named `app`.

Then stay within **that one module and source set**. Do not mix `src/main/res` with `src/debug/res`, or one module's resources with another's; a key in a different source set is a different resource, and cross-syncing them is a bug.

Report the inventory before doing any work: the module and source set in use, the English file, every locale file found, and every `values-*` directory excluded as non-locale.

## §2 — What counts as a translatable string

**`<string>` elements** are the core of both commands.

**`<plurals>` and `<string-array>`** are synchronized too when present. Match the element by its `name`, then synchronize its contents: every `<item quantity="…">` for plurals, every `<item>` in order for arrays. A plurals block is *not* complete just because the element exists — a target language may need quantity classes English does not have (`one`/`other` in English vs. `one`/`few`/`many`/`other` in others), and missing quantity classes count as missing translation work.

**Skip anything marked `translatable="false"`.** That attribute is the project telling you the string is not for translation, and adding it to a locale file would be an error, not a fix.

**Product and brand names are copied verbatim, never transliterated.** `app_name` is the usual case — whatever the app is called stays spelled that way in every locale rather than being translated or transliterated into each script. Note such entries in the summary so the developer knows they were deliberate copies rather than untranslated oversights.

Strings whose entire value is a resource reference (`@string/other_key`) are aliases — copy the reference as-is rather than translating the text it points at.

## §3 — Key comparison

**Match the `name` attribute exactly, and case-sensitively.** `login_title` and `login_Title` are different keys. Never treat two strings as equivalent because their text looks similar, because one looks like a typo of the other, or because they appear at the same position in their files. Text similarity is not identity; only the key is.

**Presence is evaluated independently for every language.** Build the comparison as a key × language matrix, not as a single "is this translated?" flag. A key present in seven locale files and missing from the eighth is a miss for that eighth language and must be translated for it.

This is the rule most likely to be broken under token pressure, so it is stated plainly: **a key existing in *some* languages is never grounds to skip it.** "Mostly translated" is not translated.

## §4 — Translation rules

Translate naturally and idiomatically for the target language, preserving the original meaning and intent. A literal word-for-word rendering that reads wrong in the target language is a defect, not a safe choice.

**Follow the file's existing conventions.** Before translating into a language, look at entries already in that file to pick up its terminology and register — formal vs. informal address (Spanish *tú* vs. *usted*), how the project already renders recurring product terms, sentence case vs. title case. Consistency with what is already there outranks your own preference.

**Placeholders are the sharp edge. Preserve them exactly.**

- Keep every `%s`, `%d`, `%f`, `%1$s`, `%1$d`, `%%` intact, with the **same count** and the **same indices** as English.
- If the target language needs a different word order, you must switch to the **positional** form (`%1$s`, `%2$d`) rather than reordering bare `%s`/`%d`. Reordering non-positional placeholders silently changes which argument lands where at runtime — the string still compiles, still passes review, and prints the wrong values.
- Never add a placeholder English doesn't have, and never drop one it does.

**Preserve XML escaping and Android string syntax:**

- Escapes: `\'`, `\"`, `\n`, `\t`, `\\`, and `\u` sequences.
- Entities: `&amp;`, `&lt;`, `&gt;`, `&#160;`.
- Inline markup: `<b>`, `<i>`, `<u>`, `<font>` — keep the tags and their structure.
- `<xliff:g>` blocks mark do-not-translate content: keep the tag, its attributes, and its inner text unchanged, translating only the text around it.
- `<![CDATA[…]]>` sections keep their wrapper.
- Leading/trailing whitespace that is significant is protected by surrounding double quotes (`"  text "`) — preserve that quoting.

**For RTL languages such as Urdu, Arabic, Hebrew, Farsi:** write the translation in normal logical order. Do not insert directional control characters (RLM/LRM/RLE/PDF), and do not reorder placeholders or punctuation to *look* right in a left-to-right editor. Android handles bidirectional rendering at runtime; manual reordering produces text that displays correctly in your editor and incorrectly on device.

## §5 — Writing changes

**Append missing entries immediately before the closing `</resources>` tag**, in the same relative order they appear in the English file. Do not insert them elsewhere in the file and do not reorder what is already there.

**Match the file's existing formatting** — indentation width, attribute quote style, and blank-line conventions. Read the file's existing entries to determine this rather than assuming; this project uses four-space indentation.

**Before appending any key, verify it is genuinely absent from that file.** This is what makes duplicate keys impossible rather than merely unlikely. A duplicate `name` in one `strings.xml` is a build-breaking error in Android.

**Never** modify an existing entry, reorder existing lines, reformat unrelated content, or delete anything. The diff a run produces should be appended lines and nothing else.

**Preserve file encoding and structure:** UTF-8 without BOM, existing line endings, and the trailing newline.

**A missing locale file is not a missing key.** If a language directory exists without a `strings.xml`, or the developer expects a language that has no directory at all, do not create it silently — adding a supported language is a project decision with manifest, QA, and release implications. Report it and ask (`AskUserQuestion`) before creating any new locale file.

## §6 — Ambiguity and human review

A string is ambiguous when the English text alone does not determine the translation. Common cases:

- **Part of speech is unclear** — "Open", "Close", "Record", "Filter" can each be a verb on a button or a noun/adjective on a label, and most languages render those differently.
- **Referent is unknown** — a short string like "None" or "All" whose grammatical gender or number depends on what it describes.
- **Agreement can't be determined** — languages requiring gendered adjectives or verb forms need to know who or what the subject is.
- **Placeholder semantics are opaque** — `%1$s` could be a name, a file, or a date, and word choice around it changes accordingly.
- **The language qualifier itself is unclear** — an unfamiliar or malformed locale folder.

**Before flagging, try the cheap disambiguation first:** grep for the key's usage — `R.string.<key>` in Kotlin/Java and `@string/<key>` in layout XML — and read the surrounding line. A button binding or a `contentDescription` usually settles the question immediately.

**If it is still ambiguous, write nothing for that key.** Do not guess, do not pick the most likely reading, and do not insert a placeholder translation. Record it in the summary with the key, the target language(s), and the specific reason it could not be resolved, so a human adds it deliberately.

## §7 — Summary format

End every run with a concise summary containing:

- **Missing keys found** — each key and which languages lacked it.
- **Languages updated** — each language with the count of strings added.
- **Requiring human review** — each skipped key, its target language(s), and the reason (§6).
- **Other issues discovered** — reported, never silently fixed:
  - Duplicate `name` values within a single file (a build error).
  - Orphan keys — present in a locale file but absent from English, usually a leftover from a removed feature.
  - Placeholder mismatches in *existing* translations — a different count or different indices than the English original, which is a live runtime crash risk.
  - Locale directories with no `strings.xml`, and `values-*` directories excluded as non-locale (§1).

Keep it scannable. If nothing was missing, say so plainly rather than going quiet.

Fast mode adds one required item to this summary — see its own file for the boundary disclosure.

## §8 — Safety rules

Apply to both Sync Strings commands, at every step:

- **A skipped question ends the command** (§0).
- **Never modify the English source strings.** `values/strings.xml` is read-only in both modes.
- **Never delete or remove a string** in any file, including orphans — report them instead.
- **Never create a duplicate key** (§5).
- **Never reformat or reorder unrelated parts of a file.** Appended lines only.
- **Never guess an ambiguous translation** (§6). Skipping is the correct answer; a plausible-looking wrong translation is worse than a missing one, because it ships silently.
- **Never treat "present in some languages" as done** (§3).
- **Never mangle a placeholder.** Same count, same indices; positional forms when word order changes (§4).
- **Confirm before writing.** Show what will be added, and to which files, before editing anything.
