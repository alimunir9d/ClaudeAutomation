# Command: Designs — XML, Optimal

Status: **active.**

You are acting as an experienced Android engineer implementing a supplied design as an XML layout with the **minimum reasonable ViewGroup hierarchy** — correctness, performance, readability and maintainability all held together, none of them sacrificed for the others.

This is **not** "ConstraintLayout at all costs". `commands/design-xml-constraint.md` is that mode, and it exists separately for a reason. Here, ConstraintLayout is the natural root and the usual answer, and another container is welcome wherever it genuinely pays for itself.

Equally, it is **not** "minimum XML tags at any cost". Optimal means:

> minimum reasonable ViewGroup hierarchy + appropriate layout choice + clean implementation + good performance + maintainability.

Both failure directions are real. Nesting four containers to arrange three views is the obvious one. Collapsing a simple vertical stack into a web of barriers and hand-tuned margins that no one can safely edit is the other, and it is the one a strict-minded implementation falls into. **Use engineering judgment**, and be able to say why each container is there.

Shared design behavior — inputs, project inspection, fidelity, preserving functionality, verification, reporting — lives in `references/design-common.md`, cited as **§DN**. Shared XML behavior lives in `references/design-xml-common.md`, cited as **§XN**. Read both before starting. Do not restate or re-implement their logic here.

This command is **purely local**: it reads and writes files in the working project, and never uses git or the network (§D preamble).

Follow the steps below in order — do not skip or reorder them.

## Step 0 — Preflight and inputs

1. Confirm this is an Android project per **§D1**. If it is not, stop cleanly and change nothing.
2. Gather the design references per **§D2** — screenshot (§D2.1), Figma MCP if one is actually connected (§D2.2), and context (§D2.3). **At least one is required.** With none, ask for one and stop; do not infer a design.
3. State which sources you have before going further.

## Step 1 — Inspect the project

Run **§D4**. Detect the module and source set, the layout directory, the theme, the design system, the resource and naming conventions, and which layout containers and custom Views the project already uses.

**Pay attention to the project's own hierarchy habits.** If its existing screens are ConstraintLayout-rooted with occasional `LinearLayout` groups, match that. A layout that is structurally unlike every other layout in the project is harder to maintain even when it is objectively flatter — §D4's rule that project convention outranks personal preference applies to hierarchy too.

Read a comparable existing layout before writing one.

## Step 2 — Existing screen or new screen

Run **§D5**. Decide explicitly and say which. Modify the existing layout when one exists; create a new one only when it genuinely is a new screen.

When editing an existing layout, an over-nested hierarchy you did not introduce is **not** automatically yours to flatten. Simplify what the requested design already touches; leave the rest alone (§D8). If the surrounding hierarchy is genuinely bad, say so in the report and let the developer decide — an unrequested structural rewrite is exactly the kind of change §D8 exists to prevent.

## Step 3 — Choose the hierarchy

Plan the structure before writing it. Work outside in.

**Start from the root.**

- Content taller than the screen → a scroll container (`NestedScrollView`) wrapping a content root. Two levels, two reasons.
- A list or grid of repeating items → `RecyclerView`, with the item as its own layout.
- Otherwise → `ConstraintLayout` as the root, usually.
- Use whatever structural container the project's own patterns require — app-bar containers, coordinator behaviors, design-system wrappers. Those are decisions the project already made.

**Then, for each grouping in the design, ask which is genuinely cleaner:**

| Situation | Usually |
|---|---|
| A handful of views in a simple stack, positioned as a unit | A `LinearLayout` — it says "these are a stack" in one line, where a chain says it in three constraints per view |
| Views distributed across the width with spacing rules | A **chain** in the root ConstraintLayout (§X2) |
| Alignment relative to whichever sibling is widest | A **barrier** (§X2) — never a container |
| Overlapping or stacked-on-top-of content | A `FrameLayout`, or simply overlapping constraints in the root |
| Show/hide several views together | A **`Group`** (§X2) — not a wrapper added for visibility |
| Weighted proportional split | Either a `LinearLayout` with weights or percent constraints; prefer whichever the project already uses |
| A design-system component that is a ViewGroup | That component — do not flatten away the project's own component |

**The test for every container: name what it does that the parent could not.** "It groups these visually" is not an answer — ConstraintLayout groups things visually without a container. "These three views are a fixed vertical stack laid out and positioned as a unit, and expressing it as a chain would be three times the XML for the same result" is an answer.

**Then check the hierarchy you planned.**

- Any container with a single child → remove it; constrain the child directly.
- Any container whose only child is another container → collapse them.
- Anything resembling `ConstraintLayout → LinearLayout → FrameLayout → LinearLayout` → start over; that is the failure this mode is named against.
- Any grouping that exists only to hold a background, a padding, or a visibility toggle → almost always a `Group`, a background on an existing view, or margins instead.
- Conversely: any place where removing a container has produced a tangle of interdependent barriers and margins to express something a `LinearLayout` says plainly → put the container back.

Depth is not the metric. **Justified depth** is. A three-level hierarchy where each level has a reason beats a two-level one held together with twenty margins.

## Step 4 — Implement

Write the layout to the plan.

Apply **§X1** throughout — project conventions, reused styles and resources, string resources for user-visible text, `tools:` attributes for preview data, ids only where something references them, `start`/`end` not `left`/`right`, accessibility attributes on meaningful elements, `<merge>` on included layouts so an include does not add a redundant ViewGroup.

Apply **§X2** wherever a ConstraintLayout is in play — complete constraints on both axes, chains and barriers and guidelines rather than simulated structure, `0dp` for match-constraints, `goneMargin` where content is conditional.

Apply **§D7** for fidelity and **§D8** for leaving unrelated things alone.

## Step 5 — Verify

Run **§X3**, including its Optimal section — in particular:

- **every ViewGroup has a stated reason**; walk them one at a time and name it
- no single-child containers, no wrapper holding only another wrapper, no redundant depth
- ConstraintLayout used properly where used — full constraints, no circular references
- the result is still readable and maintainable, not flattened into something fragile
- everything in §X3's "both modes" list: resources resolve, ids the project relies on survive, no hardcoded strings, no `px`, no `left`/`right`, overflow and long content handled, accessibility present

Then run **§D9**: re-read the file, confirm nothing unrelated changed, and run the cheapest project check that genuinely validates the change. Report the real result, including "not run" and why.

## Step 6 — Report

Produce the report per **§D10**, naming the mode as **Optimal**.

Add this mode's specific item:

- **The hierarchy you chose, and why** — the containers used and the one-line reason for each. Keep it to a few lines. This is the record that the structure was decided rather than accumulated, and it is the part a reviewer actually reads.

If you deliberately kept a container that a stricter reading would have removed, say so and why. If you left surrounding over-nesting alone per Step 2, note it as something the developer may want to address separately.

## Safety Rules (apply throughout)

All of **§D11** and all of **§X1** apply. The ones this command is most likely to be tempted to break:

- **A skipped question ends the command** (§D0).
- **Optimal is not minimum tags.** Do not sacrifice readability, maintainability or correctness to remove a container. A layout nobody can safely edit is not optimal.
- **Optimal is not ConstraintLayout-only.** Do not refuse a `LinearLayout` that is genuinely the cleaner expression. The strict mode is a separate command, and the developer chose this one.
- **Every ViewGroup must have a reason you can state.** If you cannot say what it does that its parent could not, it should not be there.
- **Never nest for its own sake** — no single-child containers, no wrapper-around-wrapper, no four-deep stacks.
- **Never leave a child unconstrained on an axis** inside a ConstraintLayout (§X2).
- **Never restructure surrounding hierarchy that the request did not touch** (§D8, Step 2).
- **Never change unrelated behavior, navigation, logic or ids** (§D8, §X1).
- **Never hardcode user-visible strings** (§D6, §X1).
- **Never claim validation that did not run** (§D9).
