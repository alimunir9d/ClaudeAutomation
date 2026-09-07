# Command: Designs — XML, ConstraintLayout Only

Status: **active.**

You are acting as an experienced Android engineer implementing a supplied design as an XML layout using **one root `ConstraintLayout` and nothing else**. Every UI element is a direct child of that root.

This mode is intentionally strict. Its value is a genuinely flat hierarchy, and that value is lost the moment a container is added "just for this one part". It is not a stylistic preference to be traded away when the layout gets awkward — the awkwardness is usually a signal that a chain, a barrier, or a guideline is the right tool (§X2), not that a `LinearLayout` is.

Shared design behavior — inputs, project inspection, fidelity, preserving functionality, verification, reporting — lives in `references/design-common.md`, cited as **§DN**. Shared XML behavior lives in `references/design-xml-common.md`, cited as **§XN**. Read both before starting. Do not restate or re-implement their logic here.

This command is **purely local**: it reads and writes files in the working project, and never uses git or the network (§D preamble).

Follow the steps below in order — do not skip or reorder them.

## Step 0 — Preflight and inputs

1. Confirm this is an Android project per **§D1**. If it is not, stop cleanly and change nothing.
2. Gather the design references per **§D2** — screenshot (§D2.1), Figma MCP if one is actually connected (§D2.2), and context (§D2.3). **At least one is required.** With none, ask for one and stop; do not infer a design.
3. State which sources you have before going further, so a missing attachment surfaces now rather than after a screen has been written.

## Step 1 — Inspect the project

Run **§D4**. Detect the module and source set, the layout directory, the theme, the design system, the resource and naming conventions, and — critically for this mode — whether ConstraintLayout is already a dependency of the module you are about to write into.

**If ConstraintLayout is not a dependency**, say so and ask (`AskUserQuestion`) whether to add it, showing which module's build file would change. Adding a dependency is a project decision, not a layout detail. If they skip, stop per **§D0**.

Read a comparable existing layout in this project before writing one. It teaches the conventions faster than any rule here.

## Step 2 — Existing screen or new screen

Run **§D5**. Decide explicitly, and say which. Modify the existing layout when one exists; create a new one only when it genuinely is a new screen.

**Editing an existing layout that is not currently a single ConstraintLayout is normal** and is what this mode is usually asked for. Restructure it to the mode's shape, keeping every id, binding and behavior the project depends on (§X1, §D8).

## Step 3 — Plan the flat hierarchy

Before writing XML, work out how each visual grouping in the design maps onto ConstraintLayout's own tools rather than onto a container. Use **§X2**:

- a row or column of views → a **chain**, with the chain style that matches the spacing in the design
- a shared alignment edge → a **guideline**
- "position this after whichever of those is widest" → a **barrier**
- show/hide several views together → a **group**
- a wrapping set of chips or buttons → **`Flow`**
- proportional or aspect-ratio sizing → **percent** constraints and **`dimensionRatio`**

Virtual helpers are not containers. `Guideline`, `Barrier`, `Group`, `Flow` and `Space` add no hierarchy depth and are fully permitted — reaching for them is the mode working as intended, not a workaround.

## Step 4 — Decide whether the mode actually fits

Do this **before** implementing, while changing course is still free.

A single ConstraintLayout genuinely cannot express some designs. The real cases, as opposed to "this is fiddly":

- **The screen scrolls.** Scrolling requires a scroll container, and a `ConstraintLayout` is not one. The standard shape is `NestedScrollView` → `ConstraintLayout`, which means the root is not a ConstraintLayout.
- **A list or grid of repeating items**, which needs a `RecyclerView` — that is a ViewGroup, and its items are separate layouts. (A `RecyclerView` as a *leaf child* whose items live in their own files is fine; hand-writing repeated rows into the layout is not.)
- **A component the project's design system provides as a ViewGroup** — a card, a bottom sheet container, a text-input layout, a swipe-refresh wrapper. Substituting a flat approximation to satisfy the mode would abandon the project's own components, which §D4 ranks higher.
- **A structural Android requirement** — a `Toolbar` inside an app-bar container, a drawer, a coordinator-driven scroll behavior.

**If the mode does not fit, do not force it and do not quietly switch strategies.**

Stop and say so:

- state plainly that the design cannot be appropriately implemented as a single ConstraintLayout
- name the specific reason — which element, and why it needs a ViewGroup
- describe what the layout would look like with the minimum additional containers

Then **ask** (`AskUserQuestion`) how to proceed: implement it with the minimum necessary containers described, or stop and let them choose a different mode. If they skip, stop per **§D0**.

Two things this exception is not. It is not a licence to add a container because a chain would take some thought — §X2 exists precisely for those cases. And it is not a reason to silently produce an Optimal-mode layout under this command's name: the developer picked the strict mode, and switching without telling them defeats the point of having two modes.

## Step 5 — Implement

Write the layout: one root `<androidx.constraintlayout.widget.ConstraintLayout>`, with every UI element as a direct child.

```xml
<androidx.constraintlayout.widget.ConstraintLayout ...>
    <!-- every view constrained, both axes, no intermediate ViewGroups -->
</androidx.constraintlayout.widget.ConstraintLayout>
```

Apply **§X1** throughout — project conventions, reused styles and resources, string resources for user-visible text, `tools:` attributes for preview data, ids only where something references them, `start`/`end` not `left`/`right`, accessibility attributes on meaningful elements.

Apply **§D7** for fidelity: match the design, but snap values to the project's scale and keep the layout correct across font scales, widths and RTL.

Apply **§D8**: when editing an existing screen, change only what the design requires. Keep the ids, bindings, listeners and navigation the project already relies on.

## Step 6 — Verify

Run **§X3**, including its ConstraintLayout Only section — in particular:

- exactly one ViewGroup in the file, the root ConstraintLayout, with any exception being one Step 4 explicitly agreed
- every child constrained on both axes, checked one at a time rather than sampled
- no circular constraints, no reference to a missing id, no missing resource
- no simulated nesting — chains and barriers, not stacks of hand-tuned margins

Then run **§D9**: re-read the file, confirm nothing unrelated changed, and run the cheapest project check that genuinely validates the change. Report the real result, including "not run" and why.

## Step 7 — Report

Produce the report per **§D10**, naming the mode as **ConstraintLayout Only**.

Add this mode's specific items:

- **Which ConstraintLayout tools carried the structure** — chains, barriers, guidelines, groups, `Flow` — in one line. This is what tells the developer the flatness is real rather than margin-simulated.
- **Any container that survived**, with the Step 4 reason it did and the developer's agreement to it.

If Step 4 ended the command instead, the report is the explanation and the question — nothing was implemented, and say so.

## Safety Rules (apply throughout)

All of **§D11** and all of **§X1** apply. The ones this command is most likely to be tempted to break:

- **A skipped question ends the command** (§D0).
- **One root ConstraintLayout, no other ViewGroups** — unless Step 4 identified a genuine technical limitation *and* the developer agreed to the exception.
- **Virtual helpers are not violations.** Do not avoid `Barrier`, `Group` or `Flow` out of a vague sense that they count as nesting. They do not.
- **Never silently switch layout strategy.** If the mode does not fit, say so and ask (Step 4). Producing an Optimal layout under this command's name is the failure this mode is defined to prevent.
- **Never force a bad implementation to satisfy the label.** A flat layout that is unreadable, fragile across screen sizes, or dependent on twenty hand-tuned margins is worse than an honest explanation of why the mode does not fit.
- **Never leave a child unconstrained on an axis.** It looks right in the preview and collapses to the corner at runtime.
- **Never change unrelated behavior, navigation, logic or ids** (§D8, §X1).
- **Never hardcode user-visible strings** (§D6, §X1).
- **Never claim validation that did not run** (§D9).
