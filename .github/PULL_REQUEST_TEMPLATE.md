<!--
Target this PR at `develop`, not `main`. See CONTRIBUTING.md.
-->

## What this changes

<!-- One or two sentences. If it adds a command, name the workflow you were doing by hand. -->

## Type of change

- [ ] New command
- [ ] Change to an existing command
- [ ] Change to a shared reference file (`references/*.md`)
- [ ] Documentation only
- [ ] Other:

## Files touched

<!--
If you changed a shared reference file, list EVERY command that reads it.
That is the change most likely to break something far from the file you edited.
-->

## Checklist

- [ ] Branched off `develop` and targeting `develop`
- [ ] Logic lives in one place — no rule restated in two files
- [ ] Any new question stops the workflow when skipped (no fallback to `(Recommended)`)
- [ ] Nothing is hardcoded that should be detected — project name, modules, branches, resource paths
- [ ] The command stops cleanly when run in an unsuitable project
- [ ] If it reads branch contents: added to `git-common.md` §G0 and cites exactly one gate (§G3 or §G4)
- [ ] If it is purely local: does not fetch, read remote refs, or ask about remote access
- [ ] New command? Added all three files — `commands/<name>.md`, the `SKILL.md` menu entry, and the direct entry point
- [ ] Entry points use sibling-relative paths (`../android-workflow/…`)

## Tested against

<!-- Which project, and what you ran. The `app/` test bed counts — say what you exercised. -->
