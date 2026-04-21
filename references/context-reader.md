# Context Reader Agent

## Purpose

The context reader agent traces cross-file and cross-layer dependencies for identifiers extracted from the diff.

It answers questions the main reviewer cannot answer from diff lines alone:

- Where does this value actually come from?
- Who else reads or depends on this identifier?
- Does the runtime state match what the changed code assumes?
- Was anything that still has live dependents silently removed?

This agent does not review the MR as a whole. It focuses only on its assigned target list and returns a dependency map for the main reviewer to reason about.

## When to spawn

Spawn one context reader agent when the diff contains any of the following:

- JS that reads DOM elements, data attributes, or hidden fields rendered by PHP
- PHP variables rendered into HTML that are later consumed by JS show/hide or calculation logic
- Removed code that may still have live dependents elsewhere in the codebase
- Changed variable structure or type that downstream consumers may silently misread
- Conditional display logic (show/hide, enable/disable, class toggle) driven by server-rendered values
- A new identifier (DOM ID, hidden field name, CSS class, function name) that crosses a file boundary

## How the main reviewer prepares the target list

Before spawning, extract from the diff:

| Category | Examples |
|---|---|
| DOM IDs and class names referenced in JS | `#CryptoCharge`, `.payment_row`, `#paySubmit` |
| Hidden field `name` attributes read by JS | `PaymentId`, `CryptoPaymentMethod`, `TotalPrice` |
| PHP variables rendered into HTML | `$totalCryptoPrice`, `$cryptoSymbol`, `$payment_row['Method']` |
| JS functions or init calls that cross files | `cart_obj.offline_payment_init()`, `cart_obj.success_init()` |
| CSS selectors used as jQuery targets | `form[name=pay_edit_form]`, `.payment_list .payment_row` |
| Removed functions, variables, or event listeners | any deleted block that had a name |

Pass the target list explicitly in the agent prompt. Do not ask the agent to derive it from scratch.

## Agent task (per target)

For each target, perform these steps in order:

### 1. Find the write side

Grep for where the value is assigned, rendered, or registered.

- For PHP variables: find the assignment and the DB query or config source behind it
- For DOM elements: find the PHP template that renders them and the condition under which they appear
- For JS functions: find where they are defined and what they call

### 2. Find the read side

Grep for every place the target is consumed.

- For DOM IDs: find every JS `$('#id')`, `getElementById`, or delegated selector
- For hidden field names: find every `$('input[name=X]').val()` or `FormData` read
- For PHP variables passed to JS: find every place the rendered value is parsed or used
- For removed functions: grep the full codebase for callers

### 3. Check cross-layer assumptions

Identify assumptions the consuming code makes about the producing code:

- Does JS assume the PHP always renders an element that may be conditionally absent?
- Does JS assume a rendered value always has a specific type, range, or non-empty state?
- Does JS assume the symbol or currency matches the currently selected payment method?
- Does one layer assume an initialization order that the other layer may not guarantee?

### 4. Check for stale dependents

Confirm that every consumer of the target was updated in the MR.

- If the MR changed how a value is produced, grep for all consumers and verify each was updated
- If the MR removed a function or element, confirm no remaining code still references it
- If the MR added a new conditional path, confirm all consumers handle the new state

## Output format

Return one block per target. Use this structure:

```
[Target: <name>]
Write side: <file>:<line> — <what it does and under what condition>
Read side:  <file>:<line> — <what the consumer expects and assumes>
Cross-layer assumption: <the implicit contract between producer and consumer>
Stale dependent: <file>:<line> — <code not updated in this MR that still depends on the old behavior>
Risk: <one-sentence description of what breaks and when>
```

Omit any field for which there is no concrete evidence. Do not speculate.

If a target has no findings, write `[Target: <name>] — no cross-layer dependency risk found`.

## Scope rules

- Review only the assigned target list, not the full MR
- Do not read files the main reviewer explicitly said are already covered
- Do not report style issues, naming preferences, or problems unrelated to the assigned targets
- Do not report pre-existing issues unless the MR makes them newly reachable or newly worse
- Distinguish findings introduced by the MR from findings that existed before it
