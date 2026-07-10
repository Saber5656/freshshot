# Title

Hooks module loader and executor

## Summary

Implement `src/capture/hooks.ts`: load the single optional repo-local ESM hooks module declared
in config, validate its exports, and execute named hooks from the `hook` step with timeout and
error attribution — the DSL's explicit escape hatch (DESIGN §8, ADR-003). Replace the issue-11
`hook` step stub.

## Context

Hooks are arbitrary trusted code (same trust as npm scripts). The engineering requirements are
therefore not sandboxing but: explicit opt-in by path, path confinement, clear failures
(`HOOK_FAILED` naming the hook), timeouts so a hung hook cannot hang CI, and a stable, minimal
context object so hook code survives freshshot upgrades.

## Scope

- `src/capture/hooks.ts`, `hook` step wiring in `src/capture/steps.ts`,
  `tests/browser/hooks.test.ts`, fixtures under `tests/fixtures/hooks/`
  (`good.mjs`, `throws.mjs`, `slow.mjs`, `not-a-function.mjs`).

## Detailed Requirements

1. Types:
   ```ts
   export interface HookContext { page: Page; baseUrl: string; shotId: string; log(msg: string): void; }
   export interface HooksRegistry { has(name: string): boolean; invoke(name: string, ctx: HookContext, timeoutMs: number): Promise<void>; }
   export async function loadHooks(hooksFileAbs: string | null): Promise<HooksRegistry | null>
   ```
2. `loadHooks`:
   - `null` input → `null` (no hooks configured).
   - Extension must be `.mjs` or `.js` (DESIGN §8); violation → `CONFIG_INVALID` (belt-and-
     suspenders; issue 04 already validates path confinement).
   - `await import(pathToFileURL(hooksFileAbs).href)`; import failure (syntax error, missing
     file) → `FreshshotError("HOOK_FAILED", "failed to load hooks module …", { cause, hint })`.
   - Loaded once per run (the runner caches the registry; the module itself does no caching).
3. `invoke(name, ctx, timeoutMs)`:
   - Missing export or non-function export → `HOOK_FAILED`
     (`hook '<name>' is not an exported function of <file>`).
   - Run `await fn(ctx)` raced against `timeoutMs` (there is no Playwright-managed timeout
     here; a raw race is correct — comment it). Timeout → `HOOK_FAILED`
     (`hook '<name>' timed out after <ms>ms`). Throw → `HOOK_FAILED` with `cause`.
   - `ctx.log` lines go to the reporter as debug output prefixed `hook:<name>:`.
4. `hook` step in `steps.ts`: `ctx.hooks === null` → `HOOK_FAILED` with hint
   `set 'hooks: ./freshshot.hooks.mjs' in config`; otherwise
   `hooks.invoke(step.hook, hookCtx, timeoutMs)` wrapped with the standard step attribution
   (shot id + step index) from issue 11.
5. Document the trust model in the module header comment (link ADR-003): hooks run with the
   user's privileges; freshshot never generates or fetches hook code.

## Acceptance Criteria

- [ ] `good.mjs` (`export async function openMenu(ctx)` clicking a fixture button via
      `ctx.page`) runs via a `hook: openMenu` step and the page reflects the click.
- [ ] `ctx` received by the hook contains exactly `page`, `baseUrl`, `shotId`, `log`
      (fixture asserts key set and that `shotId` matches).
- [ ] `throws.mjs` → `HOOK_FAILED`, message contains the hook name and the cause's first line.
- [ ] `slow.mjs` (sleeps 60 s) with `timeoutMs: 500` → `HOOK_FAILED` timeout in < 1 s.
- [ ] `not-a-function.mjs` (`export const openMenu = 42`) → `HOOK_FAILED` "not an exported function".
- [ ] Unknown hook name against `good.mjs` → `HOOK_FAILED` naming the missing export.
- [ ] `hook` step with no hooks module configured → `HOOK_FAILED` with the config hint.
- [ ] Syntax-error module → `HOOK_FAILED` at load, message includes the file path.

## Validation

- Browser project tests in CI (hooks need a live Page).

## Dependencies

- 03, 04 (path already confined at load), 09, 11 (step wiring).

## Non-goals

- Sandboxing/isolation (rejected, ADR-003). TypeScript hooks (v1 limits to .mjs/.js).
- Multiple hooks files (one module per project in v1).

## Design References

- DESIGN §8 (spec), §17.2 (trust boundary), ADR-003, §16 (`HOOK_FAILED`)
