# Title

Markdown image reference scanner

## Summary

Implement `src/scan/markdown.ts`: enumerate docs files from the configured globs, parse each
Markdown file, and extract every image reference — Markdown syntax, reference-style, and inline
HTML `<img>` — with file/line positions and root-relative resolved paths, per DESIGN §13.1 and
the parser-hardening rules in §17.6.

## Context

The scanner feeds the coverage engine (issue 21), the docs-aware differentiator (ADR-001).
Correct extraction across Markdown dialects' common cases — and NOT matching things inside code
blocks — determines whether coverage reports are trustworthy.

## Scope

- `src/scan/markdown.ts`, `tests/unit/scan-markdown.test.ts`, Markdown fixtures under
  `tests/fixtures/docs/`.

## Detailed Requirements

1. API:
   ```ts
   export interface ImageRef {
     docFile: string;           // root-relative, POSIX separators
     line: number; column: number;   // 1-based, from the containing node
     rawRef: string;            // as written
     resolved: string | null;   // root-relative POSIX path; null when skipped/external
     kind: "markdown" | "html";
     outsideRoot: boolean;      // true when resolution escaped the root (→ broken)
   }
   export interface ScanError { docFile: string; message: string; }   // DOCS_PARSE_FAILED records
   export async function scanDocs(opts: { root: string; include: string[]; exclude: string[] }):
     Promise<{ refs: ImageRef[]; files: string[]; errors: ScanError[] }>
   ```
2. File enumeration: `fast-glob` with `{ cwd: root, dot: false, followSymbolicLinks: false,
   onlyFiles: true, ignore: exclude }`; results sorted lexicographically for deterministic
   output. Only files ending `.md` are parsed (others silently ignored even if globbed).
3. Per-file hardening (DESIGN §17.6): size > 2 MiB → push a `ScanError` (`file exceeds 2 MiB`)
   and skip. Parser throw → `ScanError`, continue with remaining files.
4. Extraction with `mdast-util-from-markdown` + `unist-util-visit`:
   - `image` nodes → `url`.
   - `imageReference` nodes joined with their `definition` node by identifier
     (case-insensitive per CommonMark); missing definition → ignore.
   - `html` nodes (block and inline): regex
     `/<img\b[^>]*?\bsrc\s*=\s*["']([^"']+)["']/gi` over the node value; line/column computed
     from the node's start position plus offset within the value.
   - Content inside `code`/`inlineCode` nodes is never visited for refs (mdast guarantees the
     separation; a fixture asserts it).
5. Skip (→ `resolved: null`): URLs with a scheme (`http:`, `https:`, `data:`, `mailto:`, …),
   protocol-relative `//…`, and pure anchors (`#…`). Strip `?query` and `#fragment` from kept
   refs before resolution.
6. Resolution: leading `/` → from root; otherwise relative to the doc file's directory.
   Percent-decode the path (`%20` → space). Use issue-03 semantics for confinement: a ref
   resolving outside the root sets `outsideRoot: true` (and `resolved` to the normalized
   attempt) — it is never read from disk here.
7. Extension filter: track only `.png .jpg .jpeg .gif .webp .svg .avif` (case-insensitive).
   Other extensions are not image refs (ignored).
8. The scanner does not check file existence (issue 21 does) and never reads referenced files.

## Acceptance Criteria

Fixtures cover, and tests assert exact `ImageRef` lists for:

- [ ] `![alt](docs/images/a.png)`, with title `![a](b.png "t")`, angle-bracket destination
      `![a](<sp ace.png>)`, and `%20`-encoded refs.
- [ ] Reference style `![alt][id]` + `[id]: images/c.png` (case-insensitive id match).
- [ ] `<img src="images/d.png">` in block HTML and inline HTML, single and double quotes.
- [ ] Root-relative `/docs/images/e.png` resolves to `docs/images/e.png`; parent-relative
      `../images/f.png` referenced from `docs/guide/page.md` resolves to `docs/images/f.png`
      (tests pin the exact normalized root-relative values).
- [ ] `![x](../../outside.png)` from a root-level doc → `outsideRoot: true`.
- [ ] Fenced code block containing `![x](nope.png)` and `<img src="nope2.png">` produces zero
      refs; same for `inlineCode`.
- [ ] `http(s)://`, `//cdn`, `data:` and `#anchor` refs → `resolved: null` entries (kept for
      diagnostics) and never resolved.
- [ ] `.PNG` uppercase is tracked; `.pdf` is ignored.
- [ ] Query/fragment stripped: `a.png?v=2#top` → `a.png`.
- [ ] CRLF file parses with correct line numbers; 3 MiB fixture → `ScanError`, other files
      still scanned.
- [ ] Deterministic ordering: two runs return identical arrays.
- [ ] Line coverage of `scan/markdown.ts` ≥ 95 %.

## Validation

- Unit tests in CI; fixture set reviewed against CommonMark image syntax cases.

## Dependencies

- 01, 02, 03.

## Non-goals

- Existence checks and classification (issue 21). MDX/AsciiDoc/reST (v2). `<picture>/srcset`
  (v2; document as a code comment).

## Design References

- DESIGN §13.1 (extraction rules), §17.6 (hardening), §21 unknown #3 (inline-HTML edge cases)
