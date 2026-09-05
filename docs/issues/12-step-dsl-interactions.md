# Title

Step DSL: interaction steps

## Summary

Implement the interaction steps in `src/capture/steps.ts` on top of the issue-11 interpreter:
`click`, `hover`, `fill`, `press`, `select` — replacing their stubs, with the same timeout and
error-attribution behavior.

## Context

These five steps cover the vast majority of "drive the app into the documented state" needs
(open a menu, fill a search box, pick an option). Semantics map 1:1 to Playwright locator APIs
(DESIGN §7); anything beyond them is a hook (ADR-003).

## Scope

- `src/capture/steps.ts` (extend), `tests/browser/steps-interactions.test.ts`, fixture page
  extension (`tests/fixtures/site/form.html`: button with click counter, right-click handler,
  hover-reveal tooltip, text input, keyboard-shortcut handler, select with three options,
  focusable input for selector-scoped press).

## Detailed Requirements

1. `click`: string form → `page.locator(sel).click({ timeout })`; object form adds
   `button` (`left|right|middle`, default left) and `clickCount` (1–3, default 1) mapped
   directly to Playwright options.
2. `hover`: `page.locator(sel).hover({ timeout })`.
3. `fill`: `page.locator(step.selector).fill(step.value, { timeout })` — value is the literal
   string from config (no interpolation; DESIGN §17.5).
4. `press`: with `selector` → `page.locator(selector).press(key, { timeout })`; without →
   `page.keyboard.press(key)` (no timeout option; wrap so a hang still respects the step
   timeout via `Promise.race` — the documented exception to the "prefer Playwright timeouts"
   rule, comment it).
5. `select`: `page.locator(selector).selectOption(value | { label } | { index }, { timeout })`
   according to which of the three the config carries (exactly one, enforced by issue 04).
6. All five wrap failures as `STEP_FAILED` via the issue-11 helper (message: shot id, step
   index, type; cause first line; hint carries the selector when the step form has one —
   unscoped `press` uses a key-focused hint instead).

## Acceptance Criteria

Browser tests on `form.html`:

- [ ] `click` increments the counter; `{ button: "right" }` triggers the contextmenu handler;
      `{ clickCount: 2 }` triggers the dblclick handler.
- [ ] `hover` reveals the tooltip (`#tip` becomes visible).
- [ ] `fill` sets the input's value and clears any previous value (fill twice, assert last).
- [ ] `press: "Enter"` on the focused form input submits (page reflects submit); scoped form
      `{ press: "Control+K", selector: "#q" }` triggers the shortcut handler.
- [ ] `select` works by `value`, by `label`, and by `index`, each reflected in the page.
- [ ] A non-matching selector on `click`, `hover`, `fill`, scoped `press`, and `select` →
      `STEP_FAILED` naming the step type and index; completes within the step timeout (use
      `timeoutMs: 300` overrides to keep tests fast). Unscoped `press` has no selector and
      remains valid — malformed step shapes are issue-04 `CONFIG_INVALID`, not runtime failures.
- [ ] The issue-11 navigation tests still pass unchanged (no interpreter regressions).

## Validation

- Browser project in CI green.

## Dependencies

- 11.

## Non-goals

- Drag-and-drop, file upload, frames, multi-tab — hooks territory (ADR-003), possibly v2 DSL.

## Design References

- DESIGN §7 (step table, forms and defaults), §17.5 (literal fill values), §16 (`STEP_FAILED`)
