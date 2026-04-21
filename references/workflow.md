# MR Review Workflow

## Step 1 — Collect diff information

Run these commands in parallel:

- `git log master..stage --oneline` to list all commits
- `git diff master..stage --name-only` to list all changed files

### Full-Scope Requirement (MANDATORY)

- `mr_review` means a full MR review across the entire changed-file set in the target range, not a high-risk sample
- Do not stop after reviewing only the most suspicious files, even if confirmed bugs are already found
- A staged or partial review may be used only as an internal intermediate step; it must not be presented as the final MR review result
- Before delivering the final review, explicitly ensure every file from `git diff master..stage --name-only` has been covered locally or by a delegated reviewer
- If parallel reviewers are used, the main reviewer must still track coverage of the full changed-file list and merge the results into one final judgment

## Step 2 — Build a change-surface checklist first

Before diving into per-file details, classify the MR by change surface and make sure each applicable surface is reviewed:

- Data layer: schema, migrations, SQL, indexes, backfill scripts, default values, nullability, compatibility with old rows
- API contract: request params, response fields, serialization format, status codes, caller compatibility, fallback behavior
- Runtime config: env vars, config keys, feature flags, cache keys, queue topics, cron jobs, permissions, role checks
- Entry points: routes, controllers, CLI commands, event subscribers, templates, service registration, dependency wiring
- Build and deploy: package changes, lockfiles, CI, Docker image build, artifact paths, deployment config, rollback safety
- Observability: logs, metrics, alerts, error handling, monitoring hooks, whether failures became harder to detect

## Step 3 — Analyze each changed file

For every changed file, run `git diff master..stage -- <file>` and review the full context:

### Preconditions before conclusions

- Before calling something a confirmed bug, verify that the triggering precondition is reachable through the real business entry path, not only through a locally imaginable state
- For user flows such as add-to-cart, checkout, login merge, queue handling, refunds, and document generation, trace the primary entry function first and confirm how the relevant data is actually created, merged, or deduplicated
- Do not treat cleanup code, fallback code, repair code, or historical-data handling as proof that the same state is produced by the mainline path
- If a suspicious state is only reachable through legacy rows, abnormal interruptions, manual data edits, or secondary repair flows, downgrade it to `Risks / Edge Cases` unless the MR itself makes that state newly user-reachable
- When a finding depends on assumptions like duplicate cart rows, missing mappings, repeated submissions, or partially initialized state, explicitly verify whether the upstream guards or dedupe logic already prevent that condition in normal operation
- In short: verify `precondition reachable` before asserting `bug confirmed`

### Variable lifecycle tracking

- Identify all variables involved in the change
- Within the current scope, find the first assignment upward and the last usage downward
- For every reassignment in between, check whether structure or meaning changed
- Verify downstream code does not read fields that no longer exist
- Apply this to any transform pattern, not just a few known helper functions

### Read the full code block

- Read upward to the start of the function or block and downward to its end
- If the changed code is only a wrapper, continue into the callee, template, query, or consumer until behavior is clear

### Silent failure checks

- Missing keys in PHP often return `null` without throwing
- Loose comparisons such as `null == 0` can silently change control flow
- Missing config, missing DOM nodes, stale cache values, and partial rollout states often fail silently before they throw

### Deleted code review

- For every removed function, method, or class, grep the codebase to confirm no live caller still depends on it
- For every removed conditional branch, verify the branch did not guard an edge case that still exists in production data
- For every removed variable assignment, check whether downstream code in the same scope still references that variable
- For removed event listeners, hooks, or callbacks, confirm the corresponding trigger point was also removed or updated
- For removed config or feature flags, confirm all readers and operational docs were updated

### Added code integration review

- For every new function, class, route, template, or handler, verify it is actually wired into a reachable execution path
- Confirm new logic is registered in the right place: route table, service container, event map, template include, build manifest, permissions list, or feature flag config
- Check whether the new path conflicts with an older path, stale cache key, duplicate selector, or shadowed branch that prevents it from running

### Duplicate logic and parallel implementations

- When new functions, helpers, mappers, or handlers are added, search for existing logic that already implements the same business rule
- Verify the MR extends the canonical implementation instead of creating a second divergent path
- Flag duplicated business logic when future fixes would need to be applied in multiple places or when different callers may now use inconsistent implementations

### Contract and compatibility review

- When fields are renamed or removed, verify all producers and consumers were updated together
- For every changed output, payload, query result, mapped object, or derived field, verify downstream consumers still read the same shape and meaning
- Check field presence, naming, type, default-value semantics, units, enum meanings, and collection shape
- Flag cases where data is produced in one format but consumed as another, even if the code does not immediately throw
- When defaults change, verify old stored data, old requests, and partially rolled-out nodes still behave safely
- When a change depends on migration order, deployment order, or cache invalidation, confirm the rollout sequence is safe and reversible

### Variable declaration and naming

- For every variable used in the diff, verify it has an actual declaration somewhere in the reachable scope
- Watch for used-before-assignment cases where a variable is only assigned inside one branch but used after the branch
- Verify case matches exactly in PHP and JS, because variable names are case-sensitive

### Error handling and observability

- Check whether failures are now swallowed, downgraded, or only logged without surfacing to callers
- Check whether success and failure paths still produce useful logs or metrics for operational debugging
- Flag changes where exceptions, return codes, or retries were removed without an equivalent safeguard

### Tests and verification hooks

- Review whether the MR updated, added, or removed tests in proportion to the behavior change
- If a bug risk can only be confirmed by runtime behavior, mark it as requiring verification instead of pretending static review is enough
- Flag missing regression coverage when the MR touches fragile branching, data transforms, or multi-step flows

## Step 4 — Parallel review strategy

Use multiple review agents in parallel when the MR is large, cross-domain, or likely to exceed one reviewer's effective context.

### Trigger

- The MR touches several subsystems at once
- The same concept was renamed or expanded and may need consistency checks in many places
- The change includes both backend and frontend handling of the same business rule
- The reviewer cannot confidently track the full variable lifecycle across the whole MR alone

### Split rule

- First read the diff summary and identify the risky concepts
- Split by business domain or semantic chain, not by raw file count
- Each agent gets one coherent review theme only

### Main reviewer rule

- Read the diff summary first and form a review strategy
- Decide which parts must still be checked locally by the main reviewer
- Assign the remaining parts to parallel review agents
- Keep ownership of the final judgment and final report
- Track file coverage so the final output still represents a whole-MR review

### Agent output rule

- Review only the assigned theme
- Report only confirmed bugs introduced by the MR
- Trace full variable lifecycle, not diff lines only
- Check whether the same logic pattern was updated everywhere it exists
- Distinguish newly introduced regressions from historical issues
- Output findings with file and line, root cause, impact, and fix direction

### Merge rule

- Never forward raw agent output directly
- Deduplicate overlapping findings
- Remove issues that are pre-existing on the base branch
- Re-check any finding that depends on business assumptions before marking it confirmed
- The final MR review must be the merged result of local review plus parallel review

## Step 4-B — Context reader agent

Run one context reader agent in parallel with the main per-file review whenever the diff crosses a layer boundary.

See `references/context-reader.md` for the full agent instructions.

### Trigger

Spawn the context reader agent when the diff contains any of the following:

- JS that reads DOM elements, data attributes, or hidden fields rendered by PHP
- PHP variables rendered into HTML consumed by JS show/hide or calculation logic
- Removed functions, event listeners, or variables that may still have live dependents
- Conditional display logic driven by server-rendered values
- A new identifier (DOM ID, hidden field, CSS class, function name) that crosses a file boundary

### Main reviewer preparation

Before spawning, extract a target list from the diff:

- DOM IDs and class names referenced in JS
- Hidden field `name` attributes read by JS
- PHP variables rendered into HTML
- JS init functions or cross-file calls
- Removed named blocks (functions, listeners, variables)

Pass the target list explicitly in the agent prompt alongside the commit range.

### Parallelism

Spawn the context reader agent at the same time as other parallel domain reviewers.
Do not wait for context reader output before starting per-file review.

### Merge rule

- Treat context reader findings as additional evidence, not as a separate review section
- Promote a context reader finding to `Confirmed Bug` only after verifying the cross-layer assumption holds in the actual runtime path
- Downgrade to `Risks / Edge Cases` if the scenario requires an unusual user action or partial rollout state
- Do not forward context reader output verbatim; synthesize it into the final report
