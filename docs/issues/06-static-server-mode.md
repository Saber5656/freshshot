# Title

Static directory server mode

## Summary

Implement `src/server/static.ts`: a loopback-only `node:http` static file server for
`server.static` mode, with traversal-safe path resolution, correct content types, and clean
teardown, implementing the `AppServer` interface.

## Context

Static mode serves projects whose docs screenshots come from a built site (`site/`, `dist/`).
It is a network-facing (loopback) parser boundary; DESIGN §9.3 and §17.8 define strict behavior:
GET/HEAD only, no listing, no traversal, no caching.

## Scope

- `src/server/types.ts` (`AppServer` interface), `src/server/static.ts`,
  `tests/integration/static-server.test.ts`.

## Detailed Requirements

1. `types.ts`: `export interface AppServer { readonly baseUrl: string; stop(): Promise<void>; }`.
2. `export async function startStaticServer(opts: { root: string /* abs, validated by caller */ }): Promise<AppServer>`:
   - `http.createServer` listening on `127.0.0.1`, port `0` (ephemeral);
     `baseUrl = http://127.0.0.1:<assignedPort>`.
   - Request handling:
     a. Methods other than GET/HEAD → `405` with `Allow: GET, HEAD`.
     b. Parse `req.url` with `new URL(req.url, baseUrl)` and take `pathname` (this strips the
        query); decode it with `decodeURIComponent` (malformed → 400).
     c. Strip the leading `/` to obtain a root-relative path (empty string becomes `"."`),
        then resolve via `resolveInsideRoot(root, relPath)` (issue 03); any
        `CONFIG_PATH_ESCAPE` → plain `404` (never echo paths or reasons; DESIGN §9.3).
     d. Path is a directory → synthesize `<relPath>/index.html` and re-run it through
        `resolveInsideRoot` before stat/stream (a symlinked index escaping the root must 404);
        serve it if it exists, else `404` (no directory listing).
     e. File missing/unreadable → `404`. Success → `200` with the file streamed.
   - Headers on every response: `X-Content-Type-Options: nosniff`, `Cache-Control: no-store`.
   - Content types (minimum map): html, css, js/mjs, json, png, jpg/jpeg, gif, webp, svg, avif,
     ico, txt, woff, woff2, wasm, map. Unknown extension → `application/octet-stream`.
   - `stop()`: `server.close()` + destroy kept-alive sockets (track connections; destroy on
     stop) so it resolves promptly; idempotent.
   - No logging inside the module; it must be silent (runner logs lifecycle).
3. Error paths must never throw across the request handler boundary (wrap handler body in
   try/catch → 500 with empty body, counted in tests).

## Acceptance Criteria

- [ ] Serves `index.html` at `/`, nested assets, correct `Content-Type` for html/css/png/svg.
- [ ] `POST /` → 405 with `Allow` header. `HEAD /index.html` → 200, empty body,
      same headers as GET.
- [ ] Traversal attempts all return 404 with empty/generic body:
      `/../secret.txt`, `/%2e%2e/secret.txt`, `/a/%2e%2e/%2e%2e/secret.txt`,
      `//etc/passwd`, and a symlink inside root pointing outside root.
- [ ] Directory without `index.html` → 404 (no listing HTML).
- [ ] Malformed percent-encoding (`/%zz`) → 400.
- [ ] Every response carries `nosniff` and `no-store`.
- [ ] Server is reachable via `127.0.0.1` and NOT via the machine's LAN IP (test asserts the
      listen address family/host from `server.address()`).
- [ ] `stop()` resolves < 1 s even with an open keep-alive connection; double-`stop()` is a no-op.

## Validation

- Integration tests use real HTTP requests (`fetch`) against a fixture directory containing the
  files, a no-index subdirectory, and an escaping symlink created at test setup.

## Dependencies

- 01, 02, 03.

## Non-goals

- Range requests, compression, caching, HTTPS (out of scope for a capture-lifetime server).
- Mode selection and probing (issue 08).

## Design References

- DESIGN §9.1 (interface), §9.3 (static mode), §17.8 (exposure), §17.3 (confinement)
