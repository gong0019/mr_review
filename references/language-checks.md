# Language-Specific Review Checks

Read this file only when the MR includes PHP or JavaScript.

This file is a supplemental high-risk checklist, not the full review scope.
Use it to strengthen review coverage for language-specific failure modes that are easy to miss in diffs.

Do not treat this file as a whitelist of what to review.
Absence of these patterns does not reduce the requirement to review the full changed-file set, related call paths, and cross-file impacts.

## PHP-specific checks

- **Membership checks with loose comparison**
  Common trigger: `in_array($val, $arr)` without strict mode. When the needle could be `0`, `false`, or `""`, verify whether `in_array($val, $arr, true)` is required.
- **Membership checks across semantic layers**
  Common trigger: `in_array` where the needle and haystack may represent different domains such as IDs, status values, or labels. Confirm both sides are from the same semantic layer.
- **Config key renames and compatibility drift**
  Common trigger: config array key rename. Search the codebase for the old key and verify all readers, writers, defaults, and migrations were updated consistently.
- **Map-building with non-unique keys**
  Common trigger: `array_column($list, $val, $key)`. Verify the key field is guaranteed unique, or later entries may silently overwrite earlier ones.
- **Repeated logic patterns across sibling files**
  Common trigger: the same business rule exists in multiple controllers, services, templates, or batch jobs. Confirm the change was applied everywhere that shares the pattern.
- **Query result shape and downstream field usage**
  Common trigger: query changes or new downstream field access. Verify the `SELECT` clause includes every field accessed later and that aliases still match consumer expectations.
- **Closure capture and outer-scope dependencies**
  Common trigger: added or modified closures. Verify all referenced outer variables are explicitly captured with `use`, and that captured values still match the intended lifecycle.
- **Truthiness and default-value semantics**
  Common triggers: `empty()`, `isset()`, `?:`, `??`, or branching that mixes `0`, `"0"`, `false`, `null`, and undefined states. Verify the code distinguishes legitimate zero-like business values from missing values.

## JavaScript-specific checks

### 1. Substring matching on composite identifiers

Common triggers: `includes()` or `indexOf()` used against strings that embed IDs or typed values.
When an ID is part of a larger composite string, substring matching can falsely match longer IDs that happen to contain the same digits. Prefer position-anchored parsing such as `startsWith()`, exact token splitting, or explicit field comparison.

### 2. Backend entity expansion without full frontend parity

Common trigger: backend adds a new entity type, status, payment method, or enum value.
JS often hardcodes the same condition in multiple independent locations such as validation, display, calculation, filtering, and form input. Verify every affected frontend branch was updated, not just the first one found.

### 3. Numeric config values passed through the DOM

Common triggers: `parseFloat` or `parseInt` reading from hidden inputs, data attributes, or rendered text.
When PHP renders a config value into the DOM and JS reads it back, a missing or empty source becomes `NaN`. Any arithmetic using `NaN` produces `NaN`, and any comparison with `NaN` returns `false`.

### 4. DOM reads without existence validation

Common triggers: `getAttribute`, dataset access, selector lookups, or reading hidden inputs.
Attributes or elements can be `undefined` or `null` if the node was not rendered or the attribute is absent. Using the raw value directly, for example in `JSON.parse`, can throw at runtime.

### 5. Missing `await` in sequential async flows

Common trigger: a new async call inserted into an existing awaited sequence.
When async logic depends on step-by-step ordering, adding a call without `await` causes downstream code to run before the response arrives. Verify newly inserted async work preserves sequencing expectations.

### 6. Temporary debug output left in production paths

Common triggers: `console.log`, `console.error`, ad hoc prints, or temporary tracing flags.
Flag statements that look like debugging residue rather than intentional operational logging.

### 7. Event listener accumulation and missing cleanup

Common triggers: `addEventListener`, jQuery `on`, framework subscriptions, or repeated initialization paths.
Verify there is a matching removal path such as `removeEventListener`, `off`, destroy, or unmount cleanup. Listeners attached inside loops or repeated entrypoints can accumulate silently.

### 8. DOM timing and dynamic element lifecycle

Common triggers: AJAX rendering, `innerHTML`, modal injection, template cloning, or deferred component mount.
When JS binds events or reads attributes from dynamically inserted elements, verify the binding runs after the element exists, or use delegated binding where appropriate.

### 9. Numeric strings used as numbers without explicit normalization

Common triggers: values rendered from PHP, DOM reads, query params, or API responses used in arithmetic, sorting, or comparisons.
Strings like `"10"` may behave differently in `+`, `===`, relational comparisons, or custom sorting than true numbers. Verify values are normalized before business logic depends on numeric behavior.

## Cross-language contract checks

- **Default-value semantics drift across PHP and JavaScript**
  Common triggers: new fields, fallback logic, empty-string defaults, or optional values passed from PHP to JS. Verify both sides treat missing, null, empty-string, false, and zero-like values consistently so the frontend does not apply a different business meaning from the backend.
