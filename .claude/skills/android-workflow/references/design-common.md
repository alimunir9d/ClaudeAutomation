# Reference: Design Common

Shared behavior for the three Designs commands — `commands/design-xml-constraint.md`, `commands/design-xml-optimal.md`, and `commands/design-compose.md`.

This file is **not** a menu command. It is never dispatched to directly; it is pulled in by a command file that says "run §DN". Each section is addressable by number so the command files reference behavior instead of restating it. If you find yourself writing input handling, project inspection, fidelity, verification, or reporting logic inside a command file, it belongs here instead.

**Sections here are numbered `§D0`–`§D11`, with a `D` prefix.** The two XML commands also read `references/design-xml-common.md`, which owns `§X1`–`§X3`. A command reading two references must never have to guess which file a bare `§2` belongs to, so every citation carries its prefix — the same reason `references/git-common.md` carries `§G`.

**These commands are purely local.** They read and write files in the developer's working project and nothing else. They do not use git, do not read remote branches, do not fetch, and must never ask the developer about remote access or repository state. The remote-first rule in `references/git-common.md` does **not** apply here: it governs commands whose decisions are based on the contents of a *branch*, and these commands implement a design against files on disk. See `references/git-common.md` §G0, which lists these commands as purely local.

The one-sentence version: **implement the supplied design inside the project's own conventions — detect everything, assume nothing, and change nothing the design did not ask you to change.**

## §D0 — Answering questions and skipping

Every `AskUserQuestion` in these workflows is a real gate. If the developer does not select an option, **stop the entire command immediately.**

**What counts as no selection:** they press Skip; the answer comes back as `[No preference]` or empty; or they answer by declining — "none", "neither", "stop", "cancel". These are all the same signal. Skipping is a withdrawal from the decision, not a delegation of it to you.

**What to do:** end the command. Make no further tool calls on its behalf.

**What never to do:**

- Do not fall back to the `(Recommended)` option. That label is a suggestion for someone who is choosing; it carries no authority when nobody chose.
- Do not infer the "obviously applicable" answer from project state, even when only one option could possibly work.
- Do not re-ask the same question in different words, and do not split it into smaller questions to get an answer indirectly.
- Do not continue with a narrowed or "safe subset" version of the workflow — a half-implemented screen is worse than none, because it looks finished.

**What to say:** one short line naming the question that was skipped and stating that the workflow stopped. Do not re-litigate the choice or append an offer to continue.

**Disclose what already happened.** If any file was already created or modified before the skip, list it. "Stopped" must never be read as "nothing changed" when something did.

**Resumption:** the workflow restarts only when the developer invokes it again, from the beginning.

## §D1 — Scope gate: confirm this is an Android project

These commands are Android-specific and must not act anywhere else. A skill installed machine-wide will be invoked in a non-Android project by accident, and a clean stop is the correct outcome.

Before anything, confirm **both**:

- a Gradle project root — `settings.gradle.kts` or `settings.gradle` (or, failing that, a `build.gradle.kts` / `build.gradle`), and
- at least one Android module — a `build.gradle(.kts)` applying an Android plugin (`com.android.application` or `com.android.library`), or a glob over `**/src/*/res/` returning a result.

If either is missing, **stop cleanly**: say what was looked for, what was found instead, and that nothing was modified. Do not offer to create an Android project, and do not fall back to writing UI code somewhere that merely looks plausible.

This gate is about the *project*, not about the mode. Whether the project currently uses XML, Compose, or both is a §D4 question, not a reason to stop here.

## §D2 — Design reference inputs

A Designs command implements a design it was **given**. It never invents one. Up to three reference sources may be supplied, each independently optional:

| Source | What it is | How to take it in |
|---|---|---|
| **Screenshot** | An image of the intended UI — a mockup, an exported frame, a capture of an existing screen, an annotated image. | Attached to the conversation, or a path the developer names. Read it with the image-reading tool. |
| **Figma MCP** | A live design file reachable through a Figma MCP server, if one is connected to this session. | §D2.2 — detect first, never assume. |
| **Context** | What the developer wrote: what the screen is, what it does, what must change, constraints and requirements. | The invocation text and the surrounding conversation. |

**At least one must be available. Any combination of the three is valid** — screenshot only, Figma only, context only, or any pair, or all three. There is no required source and no privileged source.

**If none is available, stop and ask.** Do not proceed from a bare command name. Say plainly that a Designs run needs at least one reference — a screenshot, a Figma frame, or a written description of the screen — and ask for one. Inferring a screen's design from a filename, a class name, or a general sense of what such screens usually look like is the worst failure mode this command has, because the output is confident, complete, and wrong.

Report which sources you actually have before implementing. The developer needs to know whether you saw the image they thought they attached.

### §D2.1 — Reading a screenshot

Inspect it for everything that is genuinely design information:

- overall layout and hierarchy — what contains what, what scrolls, what is pinned
- spacing, padding, margins, and alignment relationships
- typography — relative size, weight, case, line height, truncation
- colors, gradients, opacity
- component identity — is that a button, a chip, a card, a list row, a toolbar, a tab bar
- sizing and proportion, including what is fixed and what fills
- shapes, corner radii, borders, dividers, elevation and shadows
- icons and images, and which are decorative rather than meaningful
- visible state — selected, disabled, empty, loading, error, focused

**Do not reproduce artifacts of the capture rather than the design.** A screenshot carries things that are not UI decisions and must not be implemented:

- device chrome — status bar, notch or cutout, navigation bar, gesture pill, rounded device corners
- editor or tool chrome — rulers, selection outlines, red-line annotations, layer names, grid overlays, cursors
- capture artifacts — compression ringing, scaling blur, a drop shadow the screenshot tool added around the frame
- placeholder content — lorem ipsum, obviously fake names, a stock avatar, sample data. Implement the *slot*, not the sample text.
- accidental cropping — content cut off at the image edge is usually the crop, not a design that clips its own content

When something could be either, say which reading you took and why, rather than silently choosing.

### §D2.2 — Using a Figma MCP, without depending on one

**Figma MCP is an optional additional source. It is never a prerequisite.** If it is not there, continue with whatever else §D2 gave you and say so once in the report.

**Detect it; never assume it.** Check the tools actually available in this session. MCP server names are frequently opaque identifiers rather than the word "figma", so identify the server by its **tool surface** — design-context, screenshot, metadata, variable-definition and code-connect style tools — not by its name. Then call only tools that genuinely exist in the session, with the parameters their schemas declare.

**Never invent a tool call.** Do not guess a tool name, do not guess a parameter, and do not describe a Figma lookup you did not perform. If the tools are present but deferred, load them properly before calling them.

**A Figma source needs a target.** Reading from Figma requires a file and usually a node — normally supplied as a `figma.com` URL, or as the current selection where the server supports that. If a Figma server is available but the developer named no file, node, or selection, do not go hunting through their Figma account: ask for the link, or proceed with the other available sources.

What Figma is *better* at than a screenshot, and therefore what to use it for: exact spacing and dimensions, named design tokens and variables, text styles, component names and variants, and the layer hierarchy. Prefer those values over measuring pixels off an image — **subject to §D7**, which still outranks them when a raw Figma value fights the project's own design system.

**If a Figma call fails** — unreachable, unauthorized, node not found — report the failure in one line and continue from the remaining sources. Do not retry in a loop, and do not stop the command over it: Figma being unavailable is exactly the case this section exists to survive.

### §D2.3 — Using context

Written context is not decoration around the "real" visual input. It routinely carries what no image can:

- what the screen *is*, and where it sits in the app
- what it is supposed to *do* — behavior, navigation, side effects
- what specifically must be created or changed, and what must not
- UX and business requirements, including rules that only surface in states the mockup does not depict
- implementation constraints — a module, a component to reuse, a pattern to follow, an API shape
- states the design does not show: empty, loading, error, offline, long content, permissions

Use it **together with** the visual references, not underneath them. When context describes behavior and the image describes appearance, both are authoritative in their own domain and there is no conflict to resolve.

## §D3 — Conflicting inputs

Sources disagree in normal use — a screenshot exported before a Figma revision, context describing an intent the mockup predates.

Work through it in this order:

1. **Is the conflict material?** A four-pixel difference in a gap the project expresses as a spacing token is not a conflict; it is measurement noise, and §D7 already decides it. A different number of buttons, a different navigation target, a different component, a field present in one source and absent from another — that is material.
2. **Can it be resolved from the available evidence?** Context usually wins on *behavior and intent* — it is the most recent statement of what the developer wants, and the only source that can express intent at all. Figma usually wins over a screenshot on *exact values*, being the live design rather than a capture of one. Project conventions (§D4) resolve conflicts about *how* to express something.
3. **If it still cannot be resolved safely, ask** — one focused `AskUserQuestion`, naming the two readings concretely and what each would produce. Do the unaffected work first where it is genuinely independent, so the question arrives with everything else already done.

**Never resolve a material conflict silently.** Picking one source and moving on produces a screen that satisfies nobody and, worse, gives no signal that a decision was made at all. Equally, do not escalate every minor discrepancy into a question — that is how a design command becomes unusable. The line is whether the choice changes the implementation in a way the developer would have wanted to make themselves.

If you resolve a conflict without asking, say so in the report (§D10) with the reason.

## §D4 — Inspect the project before implementing

This runs in someone else's Android project. **Detect; do not assume.** In particular, do not assume a single `app` module, a single `main` source set, a particular package, a particular architecture, a particular theme, or a particular design system.

Inspect enough to write code that looks like it belongs. Not all of this applies to every project — gather what is there and move on.

**Structure**

- Modules and their types — application, library, feature. Read `settings.gradle(.kts)` and each module's plugins.
- Source sets — `src/main`, and any flavor or build-type source sets that carry UI.
- The package root, from the namespace/applicationId and the actual directory tree.
- Where UI lives now: `res/layout/`, and/or Composable source files.

**UI technology**

- XML, Compose, or both. Many projects are mid-migration; find out which side the target screen is on before choosing where code goes.
- For Compose: the Compose setup, and whether Material 2 (`androidx.compose.material`), Material 3 (`androidx.compose.material3`), or a custom system is in use.
- For XML: the theme parent (`Theme.MaterialComponents.*` vs `Theme.Material3.*` vs `Theme.AppCompat.*`), and whether ConstraintLayout is already a dependency.

**Design system and resources**

- The theme and its attributes; whether the project styles via theme attributes or via direct resource references.
- Colors — `colors.xml`, a Compose color file, a token layer, semantic vs. literal naming.
- Typography — text appearances and styles, a Compose typography definition, custom fonts.
- Dimensions — a `dimens.xml`, a Compose spacing object, or values written inline throughout.
- Shapes, corner radii, elevation conventions.
- Reusable components — custom Views, custom Composables, a design-system module. **This is the highest-value thing to find.** A project with its own primary-button component does not want another one.
- Resource naming conventions — how existing layouts, ids, drawables, strings, colors and dimens are named. Match the convention that is actually there, including when it is not the one you would have chosen.
- Icons and drawables already present, before adding new ones.

**Architecture and patterns**

- How screens are structured — Activity/Fragment, single-Activity with a navigation graph, Compose navigation.
- How state reaches the UI — ViewModel, state holder, whatever the project does.
- How an existing screen of similar complexity is written. **Read one.** It is the fastest way to learn the project's actual conventions, and it outranks every general best practice in this file.

**Then follow what you found.**

- **Prefer existing components and conventions** over introducing a parallel implementation of the same thing.
- **Do not impose an architecture or design system** because it is more modern or because you prefer it. A Material 2 project gets Material 2. A project with a custom design system gets that design system.
- If the project's convention is genuinely inadequate for the design — a token that does not exist, a component that cannot express the requirement — extend it in the project's own idiom, and say so in the report.
- If detection is ambiguous in a way that changes where code lands — several candidate modules, several source sets, no obvious home for a new screen — **ask** (`AskUserQuestion`) rather than guessing. Do not pick the largest module, and do not pick the one named `app` because it is named `app`.

## §D5 — New screen or existing screen

Decide this explicitly, before writing anything, and state the conclusion.

**Search for an existing implementation first.** Use the names the reference sources give you — the screen's name, its purpose, distinctive text visible in the screenshot, the Figma frame name — and search layouts, Composables, Fragments/Activities, and navigation graphs. Distinctive on-screen text is often the fastest way in: a literal from the mockup usually leads straight to a string resource and from there to the screen that uses it.

**If the request corresponds to an existing screen:** modify that implementation. Find the specific layout file or Composable that renders it and change that. Do not create a second screen alongside the first, do not add a differently-named copy next to it, and do not leave the original orphaned.

**If it is clearly a new screen:** create it in the structure and conventions §D4 found — the right module, the right source set, the right package, the naming the project already uses.

**If you cannot tell**, ask. Two possibilities and no evidence to choose between them is exactly what one focused question is for. The cost of guessing wrong is asymmetric: a duplicate screen is a silent, lasting mess that the build will not catch.

**Never create a duplicate file.** If a near-match exists but is not quite the same screen, say what you found and how it differs, rather than quietly adding a parallel one.

## §D6 — Resources, strings, and accessibility

Applies to every mode.

**Strings.** User-visible text belongs in string resources, following the project's conventions and naming. Do not hardcode user-visible strings into layouts or Composables when the project keeps them in resources. Text that is genuinely not user-facing — a preview sample, a test tag — is not a string resource.

If the project carries translations, note in the report that newly added English strings will need translating; **do not translate them here.** That is a different command with its own rules.

**Other resources.** Follow the project's existing approach for colors, dimensions, styles, and drawables — theme attributes where the project uses theme attributes, resource references where it uses those. Add new resources only when no existing one fits, name them the way the project names things, and put them where the project puts them.

**Accessibility is part of the design, not a polish pass.**

- Every meaningful non-text element gets a description; purely decorative ones are explicitly marked as decorative rather than left ambiguous.
- Touch targets stay at least 48dp effective, even when the mockup draws a smaller glyph.
- Text scales — avoid fixed heights on text containers, and use scalable text units.
- Preserve any accessibility behavior an existing screen already has. Dropping a content description while restyling a screen is a regression nothing will flag.
- Do not fight the system's contrast, focus order, or screen-reader semantics to match a mockup exactly.

## §D7 — Visual fidelity

Aim high. The implementation should be recognizably the supplied design — hierarchy, spacing, padding and margins, alignment, dimensions, typography, colors, shapes, borders, elevation, icons, images, scrolling behavior, responsive behavior, and how it holds up with content lengths other than the sample.

**But fidelity is to the design, not to the pixels.** Two rules, in this order:

1. **The project's design system outranks a raw measured value.** If the mockup's 15dp gap sits among a project spacing scale of 8/16/24, it is 16dp. If a swatch is one shade off a named theme color, use the theme color. Reproducing off-scale values scattered through a screen is how a design system dies — quietly, one screen at a time.
2. **Android platform correctness outranks a literal copy.** Density-independent units, not pixels. Insets and system bars handled properly. Text that reflows and scales. RTL that mirrors — use start/end rather than left/right. A design that appears to demand otherwise is being read too literally.

**When the reference is ambiguous, prefer the project's established convention** and note the choice. Ask only when the ambiguity materially changes the result (§D3).

Do not invent detail the reference does not contain — a hover state, an animation, a gradient that is compression noise. Implement what is there.

## §D8 — Do not change unrelated functionality

When modifying an existing screen, change **only** what the requested design requires.

Preserve: existing behavior, navigation, state and business logic, event handling, and every feature the screen already has. Keep click listeners wired, keep bindings valid, keep ids that other code references, keep test tags that tests depend on.

**Do not refactor unrelated code for style.** Not the naming, not the formatting, not the architecture, not "while I was in there". A design change whose diff touches logic is a design change nobody can review.

**If the design genuinely requires a functional change** — a control the design removes, a new action with no handler, a field the current state cannot supply — **identify it explicitly** rather than implementing it silently. Say what the design implies, what you did, and what you deliberately left for the developer. A dead button that looks right is a bug; a button you flagged as needing a handler is a handoff.

Before finishing, check what your own diff touched. Anything outside the screen under work needs a reason.

## §D9 — Verification and project tooling

After implementing, **read back what you wrote** and check it. The mode-specific checklists live in the command files and in `design-xml-common.md` §X3; this section covers what applies to all of them.

- Re-read every file you created or modified, in full. Do not verify from memory of what you intended to write.
- References resolve: every resource, style, color, dimension, drawable, string, component and import actually exists.
- Nothing unrelated changed (§D8).
- No leftovers — no placeholder you meant to fill, no unused imports, no orphaned resource added for an approach you abandoned.

**Then run the project's own checks, if that is practical.**

- Detect the tooling rather than assuming it: a Gradle wrapper (`gradlew` / `gradlew.bat`), and the module you actually changed.
- Prefer the **cheapest task that genuinely validates the change** over a full build — resource processing or compilation of the affected module usually catches what matters, and a full assemble on a large project can take many minutes for no extra signal.
- Announce the command before running it, and report the actual result.

**Report honestly.**

- If checks pass, say which ran.
- If they fail, show the relevant output and fix what your change caused. Do not fix pre-existing failures unrelated to your change (§D8) — report them instead.
- If you did not or could not run them — no wrapper, no network for dependency resolution, a build that would be unreasonably long — **say that plainly.** "Implemented but not compiled" is a useful, honest status. Implying validation that did not happen is not.

## §D10 — Reporting

End every run with a short report. Match its length to the work: a straightforward screen gets a few lines, not an essay, and never a generic explanation of what ConstraintLayout or Compose is.

Cover:

- **Mode used** — ConstraintLayout Only, Optimal, or Compose.
- **Input sources** — which of screenshot / Figma MCP / context were available, and which you actually used. Name any that were expected but absent.
- **Files created or modified** — each path, and whether it was new or edited.
- **Design and implementation decisions worth knowing** — an existing component reused instead of a new one, a value snapped to the project's scale, a conflict resolved and why (§D3), a convention followed that differs from the reference.
- **Limitations and ambiguities** — anything the reference did not determine, anything left for the developer (§D8), any new strings needing translation (§D6).
- **Validation** — which checks ran and their result, or that none ran and why (§D9).

If a mode's own constraint could not be met, that is a headline, not a footnote — see the command file for what to do.

## §D11 — Safety rules

Apply to all three Designs commands, at every step:

- **A skipped question ends the command** (§D0).
- **Never implement a design you were not given** (§D2). No reference means ask, not improvise.
- **Never invent an MCP tool call, and never claim a Figma lookup you did not perform** (§D2.2).
- **Never assume project structure** — module, source set, package, architecture, theme, or design system (§D4).
- **Never create a duplicate screen** when an existing one should be modified (§D5).
- **Never impose a new architecture or design system** on an existing project (§D4).
- **Never resolve a material conflict silently** (§D3).
- **Never change unrelated behavior, navigation, state, or logic** (§D8).
- **Never remove existing accessibility behavior** while restyling (§D6).
- **Never hardcode user-visible strings** against the project's conventions (§D6).
- **Never claim validation that did not run** (§D9).
- **Never force a mode's constraint into a bad implementation.** If the requested mode cannot express the design well, say so and explain why — do not silently switch strategies, and do not ship something misleading to satisfy a label.
