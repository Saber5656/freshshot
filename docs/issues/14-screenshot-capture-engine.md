# Title

Screenshot capture engine (viewport / element / fullPage, masks)

## Summary

Implement `src/capture/screenshot.ts`: turn a settled page into PNG bytes according to the
shot's capture spec — viewport, fullPage, or element-with-padding — applying mask selectors and
the deterministic screenshot options (DESIGN §11).

## Context

This is the single place `page.screenshot` is called. Element capture with padding needs a
clip-rect computation (Playwright's element screenshot has no padding option), and masks use
Playwright's native `mask` locators so dynamic regions (avatars, clocks) render as constant
boxes for the diff gate.

## Scope

- `src/capture/screenshot.ts`, `tests/browser/screenshot.test.ts`, fixture page extension
  (`tests/fixtures/site/capture.html`: a tall page (3× viewport), a positioned card `#card` of
  known size, a "dynamic" badge `#badge` inside it).

## Detailed Requirements

1. ```ts
   export interface CaptureSpec { type: "viewport" | "element" | "fullPage"; selector?: string; padding: number; }
   export async function takeScreenshot(page: Page, spec: CaptureSpec, opts: {
     mask: readonly string[]; maskColor: string; timeoutMs: number;
   }): Promise<Buffer>
   ```
2. Common screenshot options for every call: `type: "png"`, `animations: "disabled"`,
   `caret: "hide"`, `scale: "device"`, `mask: opts.mask.map(s => page.locator(s))`,
   `maskColor: opts.maskColor`, `timeout: opts.timeoutMs`.
3. `viewport` → `page.screenshot({ fullPage: false, ...common })`.
4. `fullPage` → `page.screenshot({ fullPage: true, ...common })`.
5. `element`:
   a. `const box = await page.locator(spec.selector).boundingBox({ timeout })`; `null` box
      (not visible / zero-size) → `FreshshotError("CAPTURE_FAILED",
      "shot element '<selector>' is not visible", { hint })`.
   b. Expand by `spec.padding` on all sides; clamp to `x ≥ 0, y ≥ 0` and to the page's full
      scrollable size (`document.documentElement.scrollWidth/Height` via `page.evaluate`) so
      the clip never exceeds capturable bounds.
   c. `page.screenshot({ clip, fullPage: true, ...common })` — `fullPage: true` with `clip`
      captures elements below the fold without manual scrolling. (If the Playwright version
      rejects the combination, scroll the element into view first and clip in viewport
      coordinates; leave the chosen strategy in a comment — this is known-unknown-adjacent.)
6. Any Playwright error is wrapped as `CAPTURE_FAILED` with shot-agnostic message (the
   orchestrator, issue 15, adds shot attribution).
7. Return value is the PNG `Buffer` — this module never touches the filesystem.

## Acceptance Criteria

Browser tests decode results with `pngjs` and assert:

- [ ] `viewport` on a 640×480 context → PNG exactly 640×480 (DSF 1) and 1280×960 (DSF 2).
- [ ] `fullPage` on the tall fixture → height ≥ 3× viewport height.
- [ ] `element` on `#card` (known 200×100 at DSF 1) with `padding: 0` → 200×100 (±1px
      tolerance); with `padding: 10` → 220×120 (±1px); element scrolled below the fold is still
      captured correctly.
- [ ] `element` near the page edge with large padding clamps (no throw, PNG within page bounds).
- [ ] Missing/hidden selector → `CAPTURE_FAILED` mentioning the selector, within the timeout.
- [ ] `mask: ["#badge"]` → the badge's pixel region is uniformly `#FF00FF` (sample the center
      pixel of the badge's known coordinates); with a custom `maskColor: "#00FF00"` the region
      is green.

## Validation

- Browser project in CI; pixel assertions use exact coordinates from the fixture's fixed layout
  (no font-dependent positions — use sized divs, not text, for asserted geometry).

## Dependencies

- 09.

## Non-goals

- Determinism setup (issue 10 runs before this). Persistence/diff (issues 16/17).
- JPEG/quality options (PNG only, DESIGN §2.2).

## Design References

- DESIGN §11 (capture), §10 (screenshot-option determinism), §16 (`CAPTURE_FAILED`)
