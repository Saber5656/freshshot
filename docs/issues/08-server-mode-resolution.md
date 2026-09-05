# Title

Server mode resolution and lifecycle wiring

## Summary

Implement `src/server/resolve.ts`: choose the server mode from `LoadedConfig`, start the right
`AppServer` (command/static/external), probe external `baseUrl`, and provide a `withServer`
helper guaranteeing teardown. This is the only entry point the runner (issue 18) uses.

## Context

DESIGN §9 defines three mutually-exclusive modes (validated by issue 04). This issue is thin
glue, but teardown-on-every-path and the external probe behavior are contractual: a failed run
must never leave a dev server or static server running.

## Scope

- `src/server/resolve.ts`, `tests/integration/server-resolve.test.ts`.

## Detailed Requirements

1. `export async function startServer(cfg: LoadedConfig): Promise<AppServer>`:
   - command mode → `startCommandServer` with resolved `cwd` (absolute, from issue 04),
     `url`, `readyPath`, `readyTimeoutMs`, `env`.
   - static mode → `startStaticServer({ root: cfg.staticDirAbs })` (absolute path resolved and
     confined by issue 04).
   - external mode → probe exactly once with
     `fetch(cfg.server.baseUrl, { method: "GET", redirect: "manual", signal: AbortSignal.timeout(5000) })`;
     **any** immediate HTTP response (any status, including 3xx — redirects are never followed,
     so exactly one request is made) counts as reachable; network error/timeout →
     `FreshshotError("SERVER_UNREACHABLE", …, { hint: "start your app, or check server.baseUrl" })`.
     Return `{ baseUrl, stop: async () => {} }`.
2. `export async function withServer<T>(cfg: LoadedConfig, fn: (server: AppServer) => Promise<T>): Promise<T>`:
   - starts the server, runs `fn`, and calls `stop()` in a `finally` block; a `stop()` failure
     is logged (debug) but never masks `fn`'s result/error.
3. The module performs no config validation (issue 04 already guaranteed exclusivity and paths).
4. Base URL normalization: strip a trailing slash from `baseUrl` once, consistently, so step
   resolution (`new URL(path, baseUrl + "/")`) behaves identically across modes — export
   `normalizeBaseUrl(u: string): string` and unit-test it.

## Acceptance Criteria

- [ ] Config fixtures for each mode start the correct server type (spy/instance checks) and
      produce a working `baseUrl` (fetch succeeds for static; fixture command for command mode).
- [ ] External mode: local listener returning 500 still counts reachable; closed port →
      `SERVER_UNREACHABLE` within ~5 s.
- [ ] `withServer` calls `stop()` when `fn` resolves, when `fn` throws, and the error from `fn`
      is rethrown unchanged (assert same instance).
- [ ] `normalizeBaseUrl("http://x:1/") === "http://x:1"` and idempotent.
- [ ] Command/static modes leave no freshshot-owned listening socket after `withServer` returns
      (assert connection refused); external mode leaves the pre-existing user-owned listener
      untouched (assert it is still reachable afterwards).

## Validation

- Integration tests reuse fixtures from issues 06/07; run in CI.

## Dependencies

- 04, 06, 07.

## Non-goals

- Any browser/capture logic (issues 09+); readiness policy changes (issue 07).

## Design References

- DESIGN §9.1 (interface), §9.4 (external mode), §4.2 (lifecycle position)
