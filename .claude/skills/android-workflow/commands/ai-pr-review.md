# Command: AI PR Review

Status: **stub — not implemented yet.**

## Intent (for when this is implemented)

Review the diff between a source branch and a target branch using Claude as the reviewer — no GitHub API required, works entirely on local git state.

Expected shape of the eventual implementation:
1. Ask the user for the **source branch** (default: the current checked-out branch).
2. Ask the user for the **target branch** (default: `develop`; the user may specify a different branch).
3. Compute the diff between the two branches (e.g. `git diff <target>...<source>`).
4. Review the diff for correctness, style, and risk, and present findings to the user.

## Current behavior

This command is not implemented yet. If invoked, tell the user this is a stub and no action was taken.
