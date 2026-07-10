# Title

Filesystem root-confinement module

## Summary

Implement `src/core/paths.ts`: the single gatekeeper that resolves user-supplied paths and
guarantees they stay inside the project root, including through symlinks. Every filesystem
touchpoint in later issues (outputs, hooks path, static dir, static-server requests, docs globs)
must go through this module.

## Context

Config-driven file writes are the highest-impact abuse surface (DESIGN §17.1): a malicious or
typo'd config must not be able to write `../../.ssh/authorized_keys` or serve files outside the
static root. DESIGN §17.3 defines the confinement rules; this issue implements them once,
correctly, with exhaustive tests.

## Scope

- `src/core/paths.ts`, `tests/unit/paths.test.ts`.

## Detailed Requirements

1. `export async function resolveInsideRoot(root: string, p: string, opts?: { label?: string }): Promise<string>`
   - `root` is an absolute, existing directory (caller guarantees; assert with a plain Error).
   - Returns the absolute resolved path when safe; throws
     `FreshshotError("CONFIG_PATH_ESCAPE", …)` otherwise. `label` (e.g. `shots[2].output`)
     is included in the message.
   - Algorithm (must be implemented exactly):
     a. `const abs = path.resolve(root, p)` (absolute `p` allowed only if inside root).
     b. Lexical check: `path.relative(root, abs)` must not start with `..` and must not be
        absolute (Windows cross-drive).
     c. Symlink check: walk from `abs` upward to the deepest **existing** ancestor; compute
        `fs.realpath` of that ancestor; recompute the would-be final path by re-appending the
        non-existing tail; verify `path.relative(realRoot, result)` (where
        `realRoot = await fs.realpath(root)`) does not start with `..` and is not absolute.
     d. Return `abs` (the lexical path, not the realpath — callers keep root-relative layout).
2. `export function toRootRelative(root: string, abs: string): string` — POSIX-separator
   (`/`) relative path used for display, JSON output, and path-equality joins (DESIGN §13.2).
3. `export async function assertPngOutputPath(root: string, p: string, label: string): Promise<string>`
   — `resolveInsideRoot` + extension check `.png` (lowercase; reject others with
   `CONFIG_INVALID`).
4. `export async function ensureParentDir(root: string, absFile: string): Promise<void>` —
   re-verifies `absFile` is inside root, then `mkdir -p` its parent.
5. No other module may call `path.resolve` on config-derived paths — add a Biome/`eslint`-style
   comment note in the file header documenting this convention (enforced by review, not tooling,
   in v1).
6. Windows correctness: use `path` (not `path.posix`) for resolution; `toRootRelative` converts
   separators to `/`. Do not lowercase paths (document macOS case-insensitivity as accepted).

## Acceptance Criteria

- [ ] Inside-root relative (`docs/img/a.png`), nested-new (`a/b/c.png` where `a/` doesn't exist),
      and absolute-inside-root paths resolve successfully.
- [ ] Rejected with `CONFIG_PATH_ESCAPE`: `../x.png`, `a/../../x.png`, absolute path outside
      root, and a path whose existing ancestor is a symlink pointing outside root.
- [ ] `resolveInsideRoot(root, ".")` succeeds (root itself is inside root), while
      `assertPngOutputPath(root, ".", label)` rejects with `CONFIG_INVALID` (not a `.png` file).
- [ ] Backslash handling is platform-specific and tested as such: on POSIX, `..\\x.png` is an
      ordinary filename that resolves inside root; on Windows, backslashes are separators and
      the same input must be rejected as traversal.
- [ ] Symlink test: `root/link -> /tmp/outside` exists; `resolveInsideRoot(root, "link/f.png")`
      throws; a symlink pointing **inside** root passes.
- [ ] `assertPngOutputPath` rejects `.PNG`, `.jpg`, no-extension with `CONFIG_INVALID`.
- [ ] `toRootRelative` returns `/`-separated paths on all platforms.
- [ ] `ensureParentDir` creates nested parents for a safe file path; rejects an outside-root or
      symlink-escaping `absFile` with `CONFIG_PATH_ESCAPE`; and never creates any directory
      outside root (assert the escape target's parent directory was not created).
- [ ] Line coverage of `paths.ts` ≥ 95 %.

## Validation

- Unit tests create real temp directories/symlinks via `fs.mkdtemp` (no mocking of `fs`).
- Run tests on macOS locally and Linux CI (both green).

## Dependencies

- 01 (toolchain), 02 (`FreshshotError`).

## Non-goals

- Glob handling (fast-glob confinement is configured at call sites, issues 20/04).
- Static-server request-path handling (issue 06 reuses this module).

## Design References

- DESIGN §17.3 (filesystem confinement), §13.2 (path-equality joins), §16 (error codes)
