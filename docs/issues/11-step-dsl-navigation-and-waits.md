# Title

Step DSL: interpreter core, goto, waits, and the origin policy

## Summary

Implement `src/capture/steps.ts`: the step-dispatch interpreter with uniform error wrapping and
timeouts, plus the navigation/wait steps — `goto` (with the deny-by-default origin policy from
DESIGN §17.4, including post-redirect verification), `waitFor`, `wait`, and `scroll`.
Interaction steps land in issue 12 on top of this core.

## Context

Steps are the recipe language (DESIGN §7). The interpreter must attribute every failure to
`shot id + step index + step type` with the Playwright cause preserved — this is the error UX
users debug against. `goto` is also a security boundary: configs must not be able to
screenshot arbitrary origins (SSRF-style) unless explicitly allow-listed.

## Scope

- `src/capture/steps.ts`, `tests/browser/steps-navigation.test.ts`.

## Detailed Requirements

1. Types:
   ```ts
   export interface StepContext {
     page: Page;
     baseUrl: string;               // normalized (issue 08)
     allowedOrigins: readonly string[];
     defaultTimeoutMs: number;      // effective.stepTimeoutMs
     shotId: string;
     hooks: HooksRegistry | null;   // structural interface owned by this issue; implemented by issue 13
     warn(msg: string): void;       // reporter hook for lint warnings
   }
   export async function runSteps(steps: ResolvedStep[], ctx: StepContext): Promise<void>
   ```
   This issue also declares the structural interfaces in `src/core/types.ts` (issue 13
   implements them in `src/capture/hooks.ts` without changing the shapes):
   ```ts
   export interface HookContext { page: Page; baseUrl: string; shotId: string; log(msg: string): void; }
   export interface HooksRegistry { has(name: string): boolean; invoke(name: string, ctx: HookContext, timeoutMs: number): Promise<void>; }
   ```
2. Dispatch: a `switch` over the step's discriminant covering every DESIGN §7 step type;
   unknown discriminants are unreachable (config guarantees) — assert with `never`.
   This issue implements `goto`, `waitFor`, `wait`, `scroll` and wires stubs that throw
   `STEP_FAILED` "not implemented" for the issue-12/13 steps (replaced there).
3. Error wrapping: run each step inside a helper that
   - applies `timeoutMs` (step override or `defaultTimeoutMs`) to the underlying Playwright call
     via its `timeout` option (never a bare `Promise.race` when Playwright supports timeouts);
   - on failure throws `FreshshotError("STEP_FAILED",
     "shot '<id>': step <index> (<type>) failed: <first line of cause>", { cause, hint })` —
     except `NAV_BLOCKED_ORIGIN` which passes through untouched.
4. `goto` semantics:
   a. Resolve target: `new URL(spec, ctx.baseUrl + "/")` for relative specs; absolute
      `http(s)://` specs parse directly; other schemes → `NAV_BLOCKED_ORIGIN`.
   b. Allowed origins set = `new Set([origin(baseUrl), ...allowedOrigins.map(origin)])`.
      Target origin must be in the set **before** navigating, else `NAV_BLOCKED_ORIGIN`
      naming the origin and the allow-list fix (add to `allowedOrigins`).
   c. Navigate: `page.goto(href, { waitUntil: step.waitUntil ?? "load", timeout })`.
   d. **After** navigation, re-check `new URL(page.url()).origin` is in the set (redirects);
      violation → `NAV_BLOCKED_ORIGIN` including the redirect chain's final URL.
5. `waitFor`: `page.locator(sel).waitFor({ state: step.state ?? "visible", timeout })`.
6. `wait`: `page.waitForTimeout(ms)` and `ctx.warn("shot '<id>': step <i> uses fixed wait; prefer waitFor")` (DESIGN §7).
7. `scroll`: `page.locator(sel).scrollIntoViewIfNeeded({ timeout })`.
8. Steps run strictly sequentially; the first failure aborts the shot (no retries, ADR-005).

## Acceptance Criteria

Browser tests (fixture static site from issue 09 plus ad-hoc `node:http` servers created in
tests for redirect/evil-origin cases):

- [ ] Relative `goto: /` navigates; `waitUntil: "domcontentloaded"` honored (spy on options or
      behavioral proxy).
- [ ] Absolute goto to the server's own origin works; absolute goto to a second live local
      server NOT in `allowedOrigins` → `NAV_BLOCKED_ORIGIN` **without any request hitting it**
      (evil server records requests; assert zero).
- [ ] Same evil origin added to `allowedOrigins` → navigation succeeds.
- [ ] A redirect endpoint on the allowed server 302-ing to the evil origin →
      `NAV_BLOCKED_ORIGIN` mentioning the final URL.
- [ ] `goto: "file:///etc/passwd"` → `NAV_BLOCKED_ORIGIN` (scheme rejection).
- [ ] `waitFor` with `state: hidden` resolves when the element is display:none; missing selector
      times out → `STEP_FAILED` whose message contains shot id, step index, `waitFor`, and whose
      `hint` mentions the selector.
- [ ] `wait: 200` sleeps and emits exactly one warning via `ctx.warn`.
- [ ] `scroll` brings a below-fold element into view (`boundingClientRect.top < innerHeight`).
- [ ] Step timeout override: `waitFor` with `timeoutMs: 300` on a missing selector fails in
      < 1 s.

## Validation

- Browser project in CI; the zero-request assertion for blocked origins is the security
  regression test required by ISSUE_PLAN §6.3.

## Dependencies

- 02, 04 (step types), 09. (13 plugs `hooks` in later; the field ships now as nullable.)

## Non-goals

- `click/hover/fill/press/select` (issue 12); `hook` execution (issue 13).
- Sub-resource request blocking (v2, DESIGN §20).

## Design References

- DESIGN §7 (step table), §17.4 (origin policy), §16 (`STEP_FAILED`/`NAV_BLOCKED_ORIGIN`)
