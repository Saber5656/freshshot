# Title

Gated atomic writer and diff artifacts

## Summary

Implement `src/diff/write.ts`: translate a `CompareOutcome` (issue 16) into the per-shot status
and filesystem effects for `update` and `check` modes per the DESIGN §12.2 table — atomic gated
writes of output PNGs, visual diff artifacts under `.freshshot/diffs/`, and never any output
write in check mode.

## Context

This module owns the only writes freshshot ever makes to user-visible files. It must be atomic
(no torn PNGs on crash), gated (unchanged captures are discarded), and confined (issue 03).
`check` is contractually read-only for outputs — CI trust depends on it.

## Scope

- `src/diff/write.ts`, `tests/unit/diff-write.test.ts` (temp-dir based, no mocking of fs).

## Detailed Requirements

1. API:
   ```ts
   export type UpdateStatus = "new" | "updated" | "unchanged" | "forced" | "skipped" | "failed";
   export type CheckStatus  = "fresh" | "stale" | "missing-baseline" | "skipped" | "failed";
   export interface PersistResult {
     status: UpdateStatus | CheckStatus;
     changedRatio: number | null;
     reason: string | null;            // "dimension-mismatch" | "baseline-undecodable" | null
     wrote: boolean;                   // output file written?
     diffArtifact: string | null;      // root-relative path when written
   }
   export async function prepareRunDirs(root: string): Promise<void>   // recreate .freshshot/diffs, ensure .freshshot/tmp
   export async function persistOutcome(args: {
     root: string; shotId: string; outputAbs: string; outputRel: string;
     candidate: Buffer; outcome: CompareOutcome;
     mode: "update" | "check"; force: boolean;
   }): Promise<PersistResult>
   ```
2. Status mapping (DESIGN §12.2, exact):
   - update: `new`→write; `baseline-undecodable`→write, status `new`, reason
     `baseline-undecodable`; `changed`→write, `updated`; `unchanged`→no write; `force: true`
     short-circuits everything except `skipped/failed` (handled by the runner) to `forced`+write.
   - check: `new`/`baseline-undecodable`→`missing-baseline` (no write);
     `changed`→`stale`; `unchanged`→`fresh`. `force` is ignored in check mode.
3. Atomic write: `ensureParentDir` (issue 03) → write to `<outputAbs>.tmp-<pid>` (i.e.
   `home.png.tmp-<pid>` next to the target, DESIGN §12.2) → `fs.rename` over the target. On any
   error, best-effort unlink the temp file, then throw
   `FreshshotError("WRITE_FAILED", …, { cause })`.
4. Diff artifacts: for update `updated` and check `stale` where `outcome.diffPng` is non-null,
   write `.freshshot/diffs/<shotId>.png` (atomic not required; simple write). Record the
   root-relative path in `diffArtifact`. Filesystem failures here also throw
   `FreshshotError("WRITE_FAILED", …, { cause })`.
5. `prepareRunDirs`: delete and recreate `.freshshot/diffs`, ensure `.freshshot/tmp` exists
   (DESIGN §15). Called once per run by the runner. Its filesystem failures also throw
   `WRITE_FAILED`.
6. The module never reads baselines (the runner reads and passes the outcome) and never touches
   paths other than: the given `outputAbs` (pre-confined by issue 04) and `.freshshot/**`.

## Acceptance Criteria

- [ ] update/`new` writes the PNG (bytes identical to candidate) and reports
      `wrote: true, status: "new"`.
- [ ] update/`unchanged` leaves the baseline byte-identical and `wrote: false` (compare bytes
      before/after, not mtime).
- [ ] update/`changed` replaces the file and writes `.freshshot/diffs/<id>.png`.
- [ ] update with `force: true` on an `unchanged` outcome → `forced`, file rewritten.
- [ ] check mode: for `changed`, baseline bytes untouched, diff artifact written, status
      `stale`; for `new` → `missing-baseline`, nothing written outside `.freshshot/`.
- [ ] No `.tmp-` residue after success; after an injected failure (parent dir made read-only,
      POSIX-only test with `chmod 0o555`) → `WRITE_FAILED` and no partial target file.
- [ ] `prepareRunDirs` clears stale diffs from a previous run.
- [ ] `baseline-undecodable` in update mode → status `new`, reason `baseline-undecodable`,
      file rewritten.
- [ ] `baseline-undecodable` in check mode → status `missing-baseline`, reason
      `baseline-undecodable`, baseline/output bytes untouched, no diff artifact written.

## Validation

- Unit tests on real temp dirs; POSIX-only cases guarded with `it.skipIf(win32)`.

## Dependencies

- 03, 16.

## Non-goals

- Deciding what to capture or reading baselines (runner, issue 18). Reporting (issue 24).

## Design References

- DESIGN §12.2 (status/action table), §15 (storage layout), §17.3 (confinement), §16
  (`WRITE_FAILED`)
