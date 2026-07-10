# Title

Managed command server mode

## Summary

Implement `src/server/command.ts`: spawn the user's dev-server command, poll an HTTP readiness
probe, capture recent output for diagnostics, and tear the whole process tree down reliably,
implementing the `AppServer` interface and the state machine in DESIGN §9.2.

## Context

Managed command mode is the flagship differentiator for real apps (`npm run dev`). Its failure
modes (never ready, early exit, zombie children) are the top sources of CI flakiness in similar
tools, so the state machine, diagnostics, and teardown are specified exactly.

## Scope

- `src/server/command.ts`, `tests/integration/command-server.test.ts`,
  child-process fixtures under `tests/fixtures/servers/` (small Node scripts).

## Detailed Requirements

1. `export async function startCommandServer(opts): Promise<AppServer>` where `opts` =
   `{ command: string; url: string; readyPath: string; readyTimeoutMs: number;
   env: Record<string,string>; cwd: string /* abs */;
   onOutput?: (line: string) => void; graceMs?: number /* SIGTERM→SIGKILL grace, default 5000 */ }`.
2. Spawn: `child_process.spawn(command, { shell: true, cwd, env: { ...process.env, ...env },
   stdio: ["ignore", "pipe", "pipe"], detached: process.platform !== "win32" })`.
3. Output capture: merge stdout+stderr line streams into a ring buffer of the last 200 lines;
   expose the last 40 (sanitized via `sanitizeText`, issue 02) in error diagnostics. When the
   runner is in `--verbose` mode the caller may subscribe via the `onOutput` option; `onOutput`
   receives `sanitizeText(line)` output only (DESIGN §17.7) — raw child output is never
   forwarded for display.
4. Readiness: every 250 ms, probe `new URL(readyPath, new URL(url).origin).href`
   (origin-rooted per DESIGN §9.2) using
   `fetch(probeUrl, { redirect: "manual", signal: AbortSignal.timeout(2000) })`; ready when the
   **immediate** response status is 200–399 (3xx is counted as ready, never followed).
   Connection errors and other statuses continue polling.
5. State machine (DESIGN §9.2), enforced with an internal `state` field and tests:
   - child exits (any code) before ready → kill remnants, throw
     `FreshshotError("SERVER_EXITED_EARLY", …, { hint })` including exit code/signal + last 40
     lines.
   - `readyTimeoutMs` elapsed → full teardown, then throw
     `FreshshotError("SERVER_START_TIMEOUT", …)` including the probed URL + last 40 lines.
   - success → resolve with `baseUrl = url` and `stop()`.
6. Teardown (`stop()`, also used on failure paths):
   - POSIX: `process.kill(-child.pid, "SIGTERM")`; if not exited after 5 s,
     `process.kill(-child.pid, "SIGKILL")`. Guard `ESRCH` (already dead).
   - Windows: `spawnSync("taskkill", ["/pid", String(child.pid), "/T", "/F"])`.
   - `stop()` resolves only after the child's `exit` event; idempotent; never rejects.
7. After ready, a child exit does NOT fail the run by itself (shots will fail on navigation);
   record it so `stop()` doesn't hang. No auto-restart (v1 has no retries).
8. No secret filtering is attempted on output lines (v1 has no secrets, DESIGN §17.5), but
   sanitization per §17.7 is mandatory.

## Acceptance Criteria

- [ ] Fixture "slow server" (listens after 1 s) → ready resolves; `stop()` terminates the
      **process group** (fixture spawns a grandchild; test asserts the grandchild is gone).
- [ ] Fixture "exits immediately with code 7" → `SERVER_EXITED_EARLY`; message contains
      `code 7` and the fixture's goodbye line.
- [ ] Fixture "never listens" with `readyTimeoutMs: 1500` → `SERVER_START_TIMEOUT` in < 2.5 s;
      message contains the probe URL and captured output.
- [ ] Fixture "ignores SIGTERM" → `stop()` resolves via the SIGKILL path in ≈5–6 s (test uses a
      reduced grace via an injectable `graceMs` option, default 5000).
- [ ] Fixture asserting `env` merge (`process.env.FRESHSHOT_TEST` visible) and `cwd`.
- [ ] Readiness accepts an immediate 302 without following it (fixture asserts exactly one
      request per poll); keeps polling on 500 until the fixture flips to 200.
- [ ] ANSI escape sequences in fixture output do not appear in thrown error messages.
- [ ] POSIX tests pass on macOS + Linux CI; Windows-specific branch covered by unit-testing the
      command construction (no Windows CI in v1 — known unknown #4).

## Validation

- Integration tests covering all fixture behaviors listed in the acceptance criteria; run
  repeatedly (`vitest --repeat 3` locally) to check for teardown races.

## Dependencies

- 01, 02 (06 is a sibling, not a dependency).

## Non-goals

- Mode selection/probing of external URLs (issue 08).
- Readiness status allow-list configuration (known unknown #2 — file a new issue if needed).

## Design References

- DESIGN §9.1–9.2 (state machine), §17.7 (output hygiene), §16 (error codes)
