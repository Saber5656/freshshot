# Title

Chromium session and context factory

## Summary

Implement `src/capture/browser.ts`: launch/close headless Chromium via Playwright, translate a
missing browser install into `BROWSER_NOT_INSTALLED`, and create per-shot browser contexts from
a shot's effective settings (viewport, deviceScaleFactor, colorScheme, reducedMotion, fixed
timezone/locale). Also establish the vitest "browser" test project that CI's `browser-tests`
job runs (replacing the issue-01 placeholder).

## Context

One browser per run, one fresh context per shot (DESIGN §4.3, §11). Context options are the
first half of determinism (DESIGN §10 step 1); the toolkit (issue 10) builds on top. The missing-
Chromium experience must be a copy-pasteable fix, not a Playwright stack trace.

## Scope

- `src/capture/browser.ts`, vitest config update for a `browser` project
  (`tests/browser/**/*.test.ts`), `tests/browser/browser-manager.test.ts`,
  fixture page `tests/fixtures/site/index.html` (created here; extended by later issues).

## Detailed Requirements

1. `export async function launchBrowser(): Promise<Browser>`:
   - `chromium.launch({ headless: true })`.
   - Catch launch errors whose message indicates a missing executable (Playwright's
     "Executable doesn't exist" / "browserType.launch") and rethrow
     `FreshshotError("BROWSER_NOT_INSTALLED", "Chromium for Playwright is not installed",
     { hint: "run: npx playwright install chromium", cause })`. Implement the message test in
     an exported pure function `isMissingBrowserError(err: unknown): boolean` so it is unit-
     testable without breaking a real install.
2. `export async function createShotContext(browser: Browser, s: EffectiveSettings): Promise<BrowserContext>`
   with exactly these options (DESIGN §10 step 1):
   `viewport: s.viewport`, `deviceScaleFactor: s.deviceScaleFactor`,
   `colorScheme: s.colorScheme`, `reducedMotion: s.reducedMotion`,
   `timezoneId: "UTC"`, `locale: "en-US"`, `serviceWorkers: "block"` (determinism: SW caching
   varies across runs — document this addition in a comment referencing DESIGN §10).
3. `export async function closeQuietly(x: Browser | BrowserContext | null | undefined)` —
   close, swallowing errors (used in finally blocks).
4. Vitest: define two projects in `vitest.config.ts` — `unit` (existing pattern) and `browser`
   (`tests/browser/**`); update `.github/workflows/ci.yml` browser-tests job to run
   `vitest run --project browser` after `npx playwright install chromium --with-deps`
   (remove the issue-01 TODO).
5. Fixture page (`tests/fixtures/site/index.html`): static HTML with a heading, a paragraph with
   `system-ui` font stack, one CSS animation (`@keyframes spin` on `#spinner`), an
   `<input id="q">`, a `<button id="btn">` that toggles `#panel` visibility via inline script,
   and a `<select id="pick">` with three options. No external resources. (Later issues reuse it.)

## Acceptance Criteria

- [ ] Browser test: launch → `createShotContext` with `viewport 640×480, deviceScaleFactor 2,
      colorScheme "dark", reducedMotion "reduce"` → a page over the static fixture reports
      `window.innerWidth === 640`, `devicePixelRatio === 2`,
      `matchMedia('(prefers-color-scheme: dark)').matches === true`,
      `matchMedia('(prefers-reduced-motion: reduce)').matches === true`,
      `Intl.DateTimeFormat().resolvedOptions().timeZone === "UTC"`, `navigator.language ===
      "en-US"`.
- [ ] `isMissingBrowserError` unit tests: true for a fabricated Playwright missing-executable
      error message; false for a generic Error.
- [ ] `closeQuietly` never rejects (already-closed context test).
- [ ] CI `browser-tests` job runs the browser project green on ubuntu-latest.

## Validation

- CI green including the new browser job; local run documented in the PR
  (`npx playwright install chromium` prerequisite).

## Dependencies

- 01, 02, 04 (`EffectiveSettings` type).

## Non-goals

- CSS/script injection, fonts, clock, seeding (issue 10).
- Screenshotting (issue 14). Navigation (issue 11).

## Design References

- DESIGN §4.3 (one context per shot), §10 step 1 (context options), §11, §16
  (`BROWSER_NOT_INSTALLED`)
