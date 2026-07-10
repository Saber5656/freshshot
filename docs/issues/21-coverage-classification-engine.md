# Title

Coverage classification engine

## Summary

Implement `src/scan/coverage.ts`: join the scanner's image references (issue 20) with the
config's shots and filesystem existence to classify every reference as `managed`, `unmanaged`,
or `broken`, and every shot as referenced or `orphan` — the data model behind the `coverage`
command (DESIGN §13.2).

## Context

This pure joining/classification logic is the docs-aware core (ADR-001). Its rules must match
DESIGN §13.2 exactly, because CI policy (`--fail-on`) hangs off these categories.

## Scope

- `src/scan/coverage.ts`, `tests/unit/scan-coverage.test.ts`.

## Detailed Requirements

1. API:
   ```ts
   export interface CoverageReport {
     managed:   Array<{ ref: ImageRef; shotId: string }>;
     unmanaged: Array<{ ref: ImageRef }>;
     broken:    Array<{ ref: ImageRef; reason: "missing-file" | "outside-root" }>;
     orphans:   Array<{ shotId: string; output: string }>;   // output root-relative
     skippedExternal: number;    // refs with resolved === null (diagnostic count)
     files: string[];            // scanned docs files (root-relative)
     errors: ScanError[];        // pass-through from the scanner
   }
   export async function classifyCoverage(opts: {
     root: string; refs: ImageRef[]; files: string[]; errors: ScanError[];
     shots: Array<{ id: string; outputRel: string }>;
   }): Promise<CoverageReport>
   ```
2. Rules (DESIGN §13.2), evaluated per ref with `resolved !== null`:
   - `outsideRoot` → `broken` with reason `outside-root` (never touch the filesystem for it).
   - `resolved` equals some shot's `outputRel` (exact string equality of normalized
     root-relative POSIX paths) → `managed` (existence NOT required — a managed-but-not-yet-
     captured image is still managed; §13.2 note).
   - else `fs.access` existence check: exists → `unmanaged`; missing → `broken`
     (`missing-file`).
3. Orphans: every shot whose `outputRel` matches no ref's `resolved`. Shots with `skip: true`
   still count (skip affects capture, not coverage).
4. A ref appearing in multiple docs files produces one entry per occurrence; a shot referenced
   at least once is not an orphan regardless of other unreferenced copies.
5. Determinism: all arrays sorted — refs by `(docFile, line, column)`, orphans by `shotId`.
6. Pure logic apart from the existence checks; no config loading (caller passes shots), no
   markdown parsing.

## Acceptance Criteria

Given a synthetic root (temp dir) with fixture files and a shots list, tests assert the exact
report for:

- [ ] Managed: ref matches a shot output; also when the file does not exist yet (still
      `managed`).
- [ ] Unmanaged: existing PNG and existing SVG referenced with no matching shot.
- [ ] Broken/missing-file: ref to a non-existent image; Broken/outside-root: scanner-flagged
      ref (no fs access attempted — assert via a path that would throw if touched, e.g. a
      non-existent absolute prefix).
- [ ] Orphan: shot whose output no doc references; non-orphan when referenced from any one of
      two docs; `skip: true` shot can be an orphan.
- [ ] Same image referenced from two files → two managed entries, one shot, zero orphans.
- [ ] External refs counted in `skippedExternal`, absent from all category lists.
- [ ] Ordering is deterministic (shuffled input → identical report).
- [ ] Line coverage of `scan/coverage.ts` ≥ 95 %.

## Validation

- Unit tests in CI.

## Dependencies

- 04 (shot types), 20 (`ImageRef`).

## Non-goals

- CLI presentation and `--fail-on` policy mapping (issue 22). Suggestions/auto-fix (v2).

## Design References

- DESIGN §13.2 (classification table and notes), ADR-001
