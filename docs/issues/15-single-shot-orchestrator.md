# Title

Single-shot orchestrator

## Summary

Implement `src/capture/shot.ts`: `captureShot()` composes issues 09–14 for one shot — fresh
context, context determinism, page, steps, page determinism, screenshot — with guaranteed
context cleanup and uniform error attribution. The runner (issue 18) iterates this over shots.

## Context

DESIGN §4.2 defines the per-shot sequence and §4.3 mandates isolation (one fresh context per
shot, closed always). Getting the finally-blocks and error taxonomy right here means the runner
stays trivial and failures always name their shot.

## Scope

- `src/capture/shot.ts`, `tests/browser/shot-orchestrator.test.ts`.

## Detailed Requirements

1. ```ts
   export interface ShotDeps { browser: Browser; baseUrl: string; allowedOrigins: readonly string[]; hooks: HooksRegistry | null; warn(msg: string): void; debug(msg: string): void; }
   export interface ShotCapture { bytes: Buffer; durationMs: number; }
   export async function captureShot(shot: ResolvedShot, deps: ShotDeps): Promise<ShotCapture>
   ```
2. Sequence (exact order, DESIGN §4.2 / §10):
   a. `createShotContext(browser, shot.effective)` (issue 09)
   b. `applyContextDeterminism(context, shot.effective)` (issue 10)
   c. `page = await context.newPage()`
   d. `runSteps(shot.steps, stepCtx)` (issues 11–13) with `defaultTimeoutMs =
      shot.effective.stepTimeoutMs`
   e. `applyPageDeterminism(page, shot.effective)` (issue 10)
   f. `takeScreenshot(page, shot.captureSpec, { mask: shot.mask, maskColor: shot.maskColor,
      timeoutMs: shot.effective.stepTimeoutMs })` (issue 14)
   g. finally: `closeQuietly(context)` — runs on success and every failure path.
3. Error policy: `FreshshotError`s from steps/hooks/capture pass through unchanged (they carry
   attribution already); any **other** exception is wrapped as
   `FreshshotError("CAPTURE_FAILED", "shot '<id>': unexpected failure: <first line>", { cause })`.
4. `durationMs` measured with `performance.now()` from context creation to screenshot bytes.
5. `debug` receives one line per phase (`shot home: context created`, `… steps done`, …) for
   `--verbose` runs.
6. This module does not read config, touch the filesystem, or know about baselines.

## Acceptance Criteria

Browser tests against the fixture site (static server from issue 06 or Playwright's file URL —
use the static server to exercise realistic http):

- [ ] Happy path: a shot with `goto` + `click` + `waitFor` returns a decodable PNG of the
      expected dimensions; `durationMs > 0`.
- [ ] After success AND after a failing step, `browser.contexts()` length returns to its
      pre-call value (no context leak) — the failing case asserts the error is the original
      `STEP_FAILED` with the shot id in its message.
- [ ] A hook-using shot works end-to-end (hooks registry from issue 13 threaded through).
- [ ] An unexpected error injected via a hook that closes the page mid-run surfaces as a
      `FreshshotError` (either the step's own code or wrapped `CAPTURE_FAILED`), never a bare
      Playwright error.
- [ ] Two sequential `captureShot` calls with the same shot produce byte-identical PNGs on the
      static fixture (local determinism smoke; the product-wide invariant is issue 25).

## Validation

- Browser project in CI.

## Dependencies

- 10, 11, 12, 13, 14.

## Non-goals

- Baseline comparison, writing, statuses (issues 16–18). Parallelism (v1 is sequential).

## Design References

- DESIGN §4.2 (data flow), §4.3 (isolation), §10 (ordering), §16 (error pass-through)
