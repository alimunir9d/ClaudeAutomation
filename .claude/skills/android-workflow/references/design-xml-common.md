# Reference: Design XML Common

Shared XML behavior for the two XML Designs commands — `commands/design-xml-constraint.md` and `commands/design-xml-optimal.md`.

This file is **not** a menu command. It is never dispatched to directly; it is pulled in by a command file that says "run §XN". It covers only what is specific to writing Android layout XML. Everything mode-agnostic — inputs, project inspection, new-vs-existing, fidelity, preserving functionality, verification, reporting, safety — lives in `references/design-common.md` and is cited as **§DN**.

**Sections here are numbered `§X1`–`§X3`, with an `X` prefix**, because both commands that read this file also read `design-common.md`. Every citation carries its prefix so a bare `§2` is never ambiguous. `commands/design-compose.md` does not read this file at all.

## §X1 — XML implementation requirements

These apply to **both** XML modes. The mode decides the *hierarchy*; this section decides everything else, and the mode never overrides it.

**Follow the project's XML conventions, which you detected in §D4.** Attribute ordering, indentation, how ids are named, whether the project uses `style="..."`, `android:theme`, Material components or plain widgets, data binding or view binding or `findViewById`. Read a comparable existing layout and write like it.

**Reuse before you add.**

- Use existing styles and text appearances rather than repeating the same attribute cluster inline.
- Use existing dimension, color and string resources rather than adding near-duplicates. A project with `@dimen/spacing_medium` at 16dp does not want a second 16dp dimen.
- Use the project's own custom Views and `<include>`d layouts where they already express the component in the design.
- Use `<merge>` when including a layout into a parent, so the include does not add a redundant ViewGroup — this matters in both modes and matters most in ConstraintLayout Only.

**Resources, not literals.**

- User-visible text comes from `@string/...` (§D6). A hardcoded string in a layout is both a translation bug and a lint failure in most projects.
- Colors and dimensions follow the project's convention — theme attributes (`?attr/colorPrimary`, `?attr/textAppearanceBodyMedium`) where the project themes, resource references where it does not. Match what is there.
- `tools:text`, `tools:src` and `tools:visibility` are the right way to make a preview readable. They are stripped at build time, so they never leak into the app. Use them instead of shipping placeholder content as real values.

**Ids: add them with a reason.**

Give an id to a view that is referenced — by code, by a constraint, by a binding, by a test, by a navigation graph. Do **not** id every view reflexively. In ConstraintLayout Only nearly everything ends up constrained and therefore ided; in Optimal, a `TextView` inside a `LinearLayout` that nothing addresses does not need one.

Follow the project's id naming convention exactly. If it uses `tvTitle`, do not introduce `title_text_view` beside it.

**Sizing and spacing.**

- `dp` for dimensions, `sp` for text. Never `px`.
- Use `start`/`end`, never `left`/`right` — RTL support is not optional, and mixing the two silently breaks mirroring.
- Prefer `wrap_content` and constrained sizing over fixed heights on anything containing text; a fixed height clips at larger font scales.
- Do not stack redundant spacing — a margin on the child *and* padding on the parent expressing the same gap is a maintenance trap. Pick one and be consistent within the screen.

**Accessibility (§D6) in XML terms.**

- `android:contentDescription` on meaningful `ImageView`s and icon buttons; `android:importantForAccessibility="no"` (or `@null` description) on genuinely decorative ones.
- Keep effective touch targets at 48dp, using padding or `minWidth`/`minHeight` rather than growing the visual glyph.
- `android:labelFor` where a label describes an input.
- Preserve every accessibility attribute an existing layout already has.

**Preserve what is already wired (§D8).** When editing an existing layout, keep ids that code, bindings, tests or navigation reference. Renaming or deleting an id is a functional change, not a design change — if the design genuinely requires it, update the references too and call it out in the report.

## §X2 — Getting ConstraintLayout right

Relevant to both modes — Optimal still uses ConstraintLayout as its usual root — but it is what ConstraintLayout Only lives or dies by.

**Every child needs both a horizontal and a vertical constraint.** An unconstrained view silently collapses to the top-start corner at runtime while looking correct in the editor preview. This is the single most common ConstraintLayout defect and the verification step exists largely to catch it.

**Use the tools the layout actually provides**, rather than reaching for a nested container:

- **Chains** — `layout_constraintHorizontal_chainStyle` / `Vertical_chainStyle` with `spread`, `spread_inside`, or `packed` — express a row or column of views, including their distribution. This is what replaces a `LinearLayout` in ConstraintLayout Only, and it does more: `packed` plus a bias gives grouped-and-offset arrangements a `LinearLayout` cannot.
- **`layout_constraintWidth_percent` / `Height_percent`** with `0dp` for proportional sizing; **`layout_constraintDimensionRatio`** for aspect-ratio boxes such as images and video frames.
- **Guidelines** (`Guideline` with `percent` or `begin`/`end`) for a shared alignment edge several views share.
- **Barriers** (`Barrier` with `constraint_referenced_ids`) when a view must sit beyond whichever of several siblings is widest or tallest. This is the correct answer to "these labels have different lengths", and it is what people wrongly reach for a nested layout to solve.
- **Groups** (`Group` with `constraint_referenced_ids`) to toggle the visibility of several views at once without wrapping them in a container.
- **`Flow`** (from `constraintlayout` `2.x`) for a wrapping or evenly-distributed set of views — chips, tags, a button row that must wrap.
- **`layout_goneMargin*`** so spacing stays correct when a neighbour becomes `gone`. Layouts with conditional content break here constantly.
- **`Space`** when a gap genuinely needs to be a constrainable anchor.

`Guideline`, `Barrier`, `Group` and `Flow` are **virtual helpers**: they are `View` subclasses with no drawing and no layout pass of their own. They are not ViewGroups, they do not add hierarchy depth, and using them does **not** violate ConstraintLayout Only.

**`0dp` means "match constraints", not "zero".** It is how a view fills the space between two anchors, and it is the correct width for most text that should truncate or wrap rather than push its neighbours off screen.

**Do not build a grid out of margins.** If positions are expressed as a long series of hand-tuned margins from the parent, the layout will not survive a text-size change or a different screen width. Constrain views to each other.

**Bias is for asymmetric positioning**, not a substitute for constraints. A view constrained to both parent edges with a bias is centered-and-offset; a view with a bias and one constraint is unconstrained (see above).

## §X3 — XML verification

Run after implementing, alongside §D9. Read the layout back and check it — do not verify from memory.

**Both modes:**

- **Every view has complete constraints or an unambiguous parent contract.** Inside a ConstraintLayout, that means both axes constrained. Walk the children one at a time; do not sample.
- **No circular constraints**, and no constraint referencing an id that does not exist in the file.
- **Every referenced resource exists** — strings, dimens, colors, drawables, styles, theme attributes, custom View class names.
- **Every id the project's code, bindings, tests or navigation graph relied on is still present** (§D8, §X1).
- **No hardcoded user-visible strings**, no `px`, no `left`/`right` where `start`/`end` belongs.
- **Content that can overflow is handled** — text that can be long either wraps or truncates deliberately, and a screen taller than the viewport scrolls.
- **`tools:` attributes only carry preview data**, never anything the app needs at runtime.
- **Accessibility attributes are present** on meaningful images and controls (§X1).
- The layout renders sensibly at a different width and a larger font scale — reason it through even when you cannot run it.

**ConstraintLayout Only additionally:**

- **Exactly one ViewGroup in the file: the root `ConstraintLayout`.** Search the file for every other ViewGroup — `LinearLayout`, `FrameLayout`, `RelativeLayout`, `ScrollView`, `NestedScrollView`, `CardView`/`MaterialCardView`, `RecyclerView` as a container of hand-written children, another `ConstraintLayout`. Any hit is either a violation or a documented exception the command file explicitly permits.
- **Virtual helpers are not violations** — `Guideline`, `Barrier`, `Group`, `Flow`, `Space` and `<merge>` are all fine and are usually the sign the mode was done properly.
- **The hierarchy is genuinely flat**: every UI element is a direct child of the root.
- **No simulated nesting** — a chain or a barrier, not a stack of margins pretending to be a container.

**Optimal additionally:**

- **Every ViewGroup earns its place.** Name the reason for each one. "It groups things visually" is not a reason; "these three views are a fixed vertical stack that a chain would express less clearly, and the group is positioned as a unit" is.
- **No container with a single child** that the parent could have positioned directly.
- **No redundant depth** — no wrapper whose only job is to hold one other wrapper.
- **Depth is justified where it exists.** A scrolling screen legitimately needs its scroll container plus a content root; that is two levels with two reasons, not nesting for its own sake.
