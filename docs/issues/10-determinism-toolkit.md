# Title

Determinism toolkit

## Summary

Implement `src/capture/determinism.ts`: the context-level setup (frozen clock, seeded
`Math.random`) and the pre-capture page-level setup (font readiness, animation-freezing CSS,
scrollbar hiding, settle delay) exactly as ordered in DESIGN §10.

## Context

The diff gate (ADR-005) only works if identical UI renders identically. This module is the
product's answer to font pop-in, animation frames, blinking carets, OS scrollbars, `Math.random`
and `Date.now` jitter. The product-wide invariant ("second consecutive update is 100 %
unchanged") depends on it.

## Scope

- `src/capture/determinism.ts`, `tests/browser/determinism.test.ts`, fixture page additions
  (`tests/fixtures/site/dynamic.html` with animation, random and clock displays).

## Detailed Requirements

1. `export async function applyContextDeterminism(context: BrowserContext, s: EffectiveSettings): Promise<void>`
   — call **before any page exists** (DESIGN §10 order):
   - `s.freezeTime` non-null → `await context.clock.install({ time: new Date(s.freezeTime) })`.
   - `s.seedRandom` → `context.addInitScript` replacing `Math.random` with mulberry32, seed
     fixed to `1`. Embed the standard mulberry32 implementation; keep it in an exported
     constant `SEED_RANDOM_SCRIPT` for testability.
2. `export async function applyPageDeterminism(page: Page, s: EffectiveSettings): Promise<void>`
   — call after the last step, immediately before capture (DESIGN §10 step 4), in this order:
   a. `s.waitForFonts` → `await page.evaluate(() => (document as any).fonts?.ready)` guarded
      with a 5 s timeout (`Promise.race`) so a broken font never hangs a shot — on timeout,
      continue silently (capture proceeds; determinism best-effort).
   b. `s.disableAnimations` → `page.addStyleTag` with exported constant `FREEZE_CSS`:
      `*,*::before,*::after{animation:none!important;transition:none!important;caret-color:transparent!important;scroll-behavior:auto!important}`.
   c. `s.hideScrollbars` → `page.addStyleTag` with exported constant `HIDE_SCROLLBAR_CSS`:
      `::-webkit-scrollbar{display:none!important} html{scrollbar-width:none!important}`.
   d. `s.settleMs > 0` → `page.waitForTimeout(s.settleMs)`.
3. Both functions are no-ops for disabled flags; both are idempotent (calling twice must not
   throw or duplicate effects beyond harmless extra style tags).
4. No screenshot-option handling here (that's issue 14: `animations: "disabled"`,
   `caret: "hide"`, `scale: "device"`).

## Acceptance Criteria

Browser tests against the fixture pages:

- [ ] With `seedRandom: true`, two separate contexts loading `dynamic.html` (which renders
      `Math.random()` into the DOM on load) show the **identical** value; with `false`, values
      differ.
- [ ] With `freezeTime: "2026-01-02T03:04:05.000Z"`, `Date.now()` in the page equals
      1767323045000 and does not advance across a 300 ms wait.
- [ ] After `applyPageDeterminism` with `disableAnimations: true`, the fixture's animated
      element reports `getComputedStyle(el).animationName === "none"`.
- [ ] `waitForFonts` path completes < 5.5 s on a page with no custom fonts (no hang).
- [ ] With `hideScrollbars: true` on a page forced to overflow,
      `window.innerWidth - document.documentElement.clientWidth === 0` (no classic scrollbar
      gutter).
- [ ] All flags false → both functions resolve without modifying the page (style tag count
      unchanged).

## Validation

- Browser project tests in CI. (The full second-run-unchanged invariant lands with issue 25;
  this issue proves the individual mechanisms.)

## Dependencies

- 09.

## Non-goals

- Locale/timezone (context options, issue 09). Masks (issue 14). Retry/stabilization loops
  (v1 non-goal).

## Design References

- DESIGN §10 (ordering and mechanisms), ADR-005, §21 unknown #1 (clock vs schedulers — if the
  fixture reveals problems, file a new issue rather than expanding this one)
