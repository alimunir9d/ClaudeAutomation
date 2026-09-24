# Command: Designs — Compose

Status: **active.**

You are acting as an experienced Android engineer implementing a supplied design in **Jetpack Compose**, written the way the project already writes Compose.

The result must be idiomatic Compose — not an XML layout mechanically transliterated into `Column`s and `Box`es. A layout tree translated one-for-one from a View hierarchy is the characteristic failure of this command: it compiles, it looks right, and every convention of the framework and the project has been ignored.

Shared design behavior — inputs, project inspection, new-vs-existing, fidelity, preserving functionality, verification, reporting — lives in `references/design-common.md`, cited as **§DN**. Read it before starting. Do not restate or re-implement its logic here. This command does **not** read `references/design-xml-common.md`; that file is for the two XML modes.

This command is **purely local**: it reads and writes files in the working project, and never uses git or the network (§D preamble).

Follow the steps below in order — do not skip or reorder them.

## Step 0 — Preflight and inputs

1. Confirm this is an Android project per **§D1**. If it is not, stop cleanly and change nothing.
2. Gather the design references per **§D2** — screenshot (§D2.1), Figma MCP if one is actually connected (§D2.2), and context (§D2.3). **At least one is required.** With none, ask for one and stop; do not infer a design.
3. State which sources you have before going further.

## Step 1 — Inspect the project

Run **§D4**, then establish the Compose specifics before writing anything:

- **Is Compose set up in this project at all?** The Compose plugin/compiler and dependencies on the module you would write into.
- **Which Material generation** — Material 3 (`androidx.compose.material3`), Material 2 (`androidx.compose.material`), or a custom design system built on neither. Never mix generations in one screen.
- **The theme** — the project's theme composable, its color scheme, typography and shapes, and whether the project uses `MaterialTheme.*` or its own token objects.
- **Existing composables to reuse** — buttons, cards, list rows, headers, loading and empty states, a design-system module. Finding these is the highest-value part of this step (§D4).
- **State conventions** — where state lives, how it reaches the UI, whether screens take a ViewModel or a state object plus lambdas, how events flow back up.
- **File and naming conventions** — where composables live, one per file or grouped, how previews are written and annotated.

**If Compose is not set up in this module**, say so and ask (`AskUserQuestion`) how to proceed — adding Compose to a project is a substantial decision, not a step to take silently. If they skip, stop per **§D0**.

**Read an existing screen of similar complexity.** In Compose this matters more than anywhere else, because "idiomatic" is largely project-defined: state hoisting depth, preview conventions, modifier ordering habits, whether screens are split into a stateful and a stateless pair. Match what is there.

## Step 2 — Existing screen or new screen

Run **§D5**. Decide explicitly and say which.

Modify the existing composable when one exists. Do not add a parallel screen composable beside it, and do not create a second version of a component the project already has.

A screen currently written in XML is **not** automatically a candidate for conversion. If the target screen is an XML layout and the developer asked for Compose, confirm what they want before rewriting: implementing a design and migrating a screen to Compose are different jobs with different risk. Ask (`AskUserQuestion`), and if they skip, stop per **§D0**.

## Step 3 — Plan the composable structure

Before writing, decide the shape.

**Composable boundaries.** Split where a piece is reused, independently previewable, or large enough to obscure its parent — not on every visual grouping. A screen of thirty tiny composables is as hard to read as one of three hundred lines. Follow the granularity the project already uses.

**State ownership.** Hoist state to the lowest common owner that needs it, and no higher. Prefer a stateless composable taking values and lambdas, with a thin stateful wrapper above it, when the project does that. Do not introduce a state-management approach the project does not use (§D4).

**Layout.** Pick the composable that expresses the intent:

- `Column` / `Row` for stacks, with `Arrangement` and `Alignment` doing the spacing rather than padding stuffed onto every child
- `Box` for overlap and for filling behind content
- `LazyColumn` / `LazyRow` / `LazyVerticalGrid` for lists that can be long or unbounded — with stable `key`s, and `contentType` where item types vary
- `Scaffold` and the project's structural composables for top bars, bottom bars and snackbars
- `ConstraintLayout` for Compose only where a genuinely complex relationship needs it; in Compose it is the exception, not the default

Use `Arrangement.spacedBy` rather than a padding on each child. Use `weight` rather than measuring. Avoid nesting that adds nothing — a `Box` wrapping a single composable that could have carried the modifier itself is noise.

**A scrolling `Column` is not a `LazyColumn`.** A bounded screen that scrolls uses `verticalScroll`; a list of unbounded or dynamic items uses a lazy composable. Choosing the wrong one is a real performance defect, not a style question.

## Step 4 — Implement

Write the composables to the plan, applying:

**Theme and design system (§D4, §D7).** Use `MaterialTheme` colors, typography and shapes — or the project's own tokens — rather than literal values scattered through the file. Dimensions that repeat belong in the project's spacing convention. Reuse the project's components instead of rebuilding them.

**Modifiers.**

- One `modifier: Modifier = Modifier` parameter, first among the optional parameters, applied to the composable's root — so callers can size and position it.
- Order matters and is semantic: padding before or after a background changes what is painted, `clip` before `clickable` changes the touch area and ripple. Write the order you mean.
- Do not hardcode a size onto a reusable component that the caller should control.

**Recomposition and stability (§D9).**

- Pass values, not whole objects, when only a value is used.
- Read state as late as possible — take a lambda rather than a value where the value changes often.
- `remember` derived work; `derivedStateOf` for state computed from other state that changes more often than the result.
- Stable types for parameters — avoid unstable collections and lambdas recreated every recomposition where the project cares about this.
- Never perform side effects in composition; use the appropriate effect API.

**Accessibility (§D6).** `contentDescription` on meaningful images and icon buttons, `null` on genuinely decorative ones. Semantics for anything whose meaning is not carried by its content. Effective touch targets at 48dp — use `sizeIn`/padding, not a bigger icon. Do not strip existing semantics while restyling.

**Strings (§D6).** `stringResource(...)` for user-visible text, following the project's conventions.

**Previews.** Add them where they earn their place and where the project already uses them — matching its annotation style, its preview naming, and whether it wraps previews in the project theme. Previews use sample data, never the sample text as a real value.

**Preserve behavior (§D8).** When editing an existing screen, keep its state handling, callbacks, navigation and side effects intact. Change what the design requires and nothing else.

## Step 5 — Verify

Read back what you wrote and check it, alongside **§D9**:

- **Idiomatic, not transliterated** — the structure reflects Compose's model, not a translated View tree.
- **Composable boundaries are sensible** — neither one monolith nor a scatter of trivial functions.
- **State ownership is correct** — hoisted to the right level, no state introduced where a parameter belonged, no state-management pattern imposed that the project does not use.
- **No unnecessary nesting** — no wrapper composable that only forwards a modifier.
- **Modifier usage is right** — a `modifier` parameter on public composables, applied to the root; deliberate ordering; no size hardcoded into a reusable component.
- **Recomposition hazards addressed** — no side effects in composition, no unremembered derived work, lazy composables keyed, list vs. scrolling column chosen correctly.
- **Project design system respected** — right Material generation, theme values rather than literals, existing components reused.
- **Accessibility handled** — descriptions, semantics, touch targets (§D6).
- **Everything resolves** — imports, resources, components, theme references.
- **Nothing unrelated changed** (§D8).

Then run the project's checks per **§D9** — for Compose the compile of the affected module is usually the cheapest task with real signal. Report the actual result, including "not run" and why.

## Step 6 — Report

Produce the report per **§D10**, naming the mode as **Compose**.

Add this mode's specific items:

- **Composable structure** — the composables added or changed, in a line or two.
- **State decisions** — what is hoisted where, and anything the design implies that the current state cannot supply (§D8).
- **Design-system alignment** — which existing components and theme values you reused, and anything you had to add because none fit.

## Safety Rules (apply throughout)

All of **§D11** applies. The ones this command is most likely to be tempted to break:

- **A skipped question ends the command** (§D0).
- **Never write transliterated XML.** A View hierarchy mapped one-for-one onto Compose containers is the defining failure of this command.
- **Never impose a Compose architecture on an existing project** (§D4). Material 2 stays Material 2. A custom design system stays the design system. The project's state pattern is the state pattern.
- **Never rebuild a component the project already has.**
- **Never convert an XML screen to Compose without asking** (Step 2).
- **Never scatter literal dimensions, colors or text styles** where the project has theme or token values (§D7).
- **Never put a side effect in composition**, and never leave derived work unremembered.
- **Never use a scrolling `Column` for an unbounded list**, or a lazy composable for a fixed handful of items.
- **Never remove existing accessibility semantics** while restyling (§D6).
- **Never change unrelated behavior, navigation, state or logic** (§D8).
- **Never hardcode user-visible strings** (§D6).
- **Never claim validation that did not run** (§D9).
