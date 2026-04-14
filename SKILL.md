---
name: mr-review
description: |
  Review merge requests from `stage` to `master` with full changed-file coverage. Use this skill when the user asks for `mr_review`, `/mr_review`, MR review, PR review, or wants a structured bug-focused review with variable lifecycle tracking and system-level change-surface checks.
---

# MR Review

Use this skill when the user wants a merge request or pull request review.

This skill is for review work, not implementation. The goal is to find bugs, regressions, rollout hazards, and missing verification, then report them in a structured way.

## Trigger

Use this skill when the user says things like:

- `mr_review`
- `/mr_review`
- review this MR
- review this PR
- audit this merge request
- check whether this change is safe

Unless the user specifies a different range, review `master..stage`.

## Progressive loading

Do not load every reference file upfront.

Use this order:

1. read this `SKILL.md`
2. read `references/workflow.md`
3. read `references/report-format.md`
4. read `references/language-checks.md` only when the MR includes PHP or JavaScript

`references/language-checks.md` is a supplemental high-risk checklist, not a limit on review scope.
Use it to probe for language-specific failure modes after building full changed-file coverage, not as a substitute for complete review.

## Core workflow

1. Collect commit list and changed-file list for the target range.
2. Build a change-surface checklist before deep review.
3. Ensure the full changed-file set is covered, locally or through delegated reviewers.
4. Review each changed file with full context, not diff lines alone.
5. Distinguish confirmed bugs from risks and unverified concerns.
6. Produce the final report using the required review format.

## Required review posture

- Prioritize confirmed bugs over style comments
- Treat hidden runtime regressions as more important than cleanliness or taste
- Track variable lifecycle across the full block, not only changed lines
- Check integration, compatibility, rollout order, and verification gaps
- Never claim full MR review unless the entire changed-file set was covered

## Dynamic file mapping checks

When the MR adds or changes any dynamic type-to-file behavior, treat file-mapping completeness as a required review dimension, not an optional follow-up.

This applies to cases such as:

- new `type` or `action` values
- new export, print, document, template, view, page, notification, report, or render modes
- string-built paths
- template or view name mapping tables
- conditional includes, renders, imports, or loaders
- config-driven file selection

For these changes, explicitly check all of the following:

1. Entry is reachable
   Confirm the new type is actually allowed by routing, whitelist checks, enums, config, permissions, and branch conditions.

2. Target files exist
   If code can dispatch to a file by convention, mapping, or string concatenation, verify the corresponding physical file exists.

3. File content matches the new scenario
   Do not stop at file existence. Check whether the file content is actually adapted for the new type instead of being an unchecked copy.

4. Rendered variables are supplied
   For templates/views/documents, trace every newly relevant variable back to the data-preparation layer and verify it is really populated.

5. End-to-end chain is closed
   Review the full path from type declaration to data assembly to file selection to render/output, and flag any broken link.

6. Similar implementations are aligned
   Compare against sibling implementations to detect missing files, missing links, stale titles, stale labels, wrong field usage, or mismatched output structure.

Treat the following as high-priority confirmed bugs when supported by code evidence:

- code opens a new type but the target file is missing
- file exists but still contains another type's title, labels, fields, or assumptions
- template expects variables that are not populated on the active render path
- render path is partially wired, so generation succeeds for some assets but fails for others
- a copied template produces structurally wrong output for the new type even though it does not crash

## Output contract

Follow `references/report-format.md`.

The final output must contain:

- `Confirmed Bugs`
- `Risks / Edge Cases`
- `Verification`
- `Summary`

## Related files

- `references/workflow.md`
- `references/report-format.md`
- `references/language-checks.md`
