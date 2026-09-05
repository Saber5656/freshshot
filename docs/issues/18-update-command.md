# Title

`update` command and run orchestration

## Summary

Implement `src/core/runner.ts` (the shared update/check run loop) and wire
`src/cli/commands/update.ts`: load config, start the server, launch the browser, iterate shots
sequentially through capture → compare → persist, aggregate a `RunSummary`, report, and set the
exit code. This issue makes freshshot do its job end-to-end for the first time.

## Context

DESIGN §4.2 fixes the data flow and §12.2/§14.4 fix statuses and exit codes. The runner owns
sequencing, baseline reading, per-shot error containment (a failed shot never aborts the run),
and resource cleanup via `withServer` + finally blocks.

## Scope

- `src/core/runner.ts`, `src/cli/commands/update.ts` (replace stub), result model additions in
  `src/core/types.ts`, `tests/browser/runner-update.test.ts` (API-level, fixture project in a
  temp dir).

## Detailed Requirements

1. Result model (`core/types.ts`):
   ```ts
   export type ShotStatus = "new" | "updated" | "unchanged" | "forced" | "skipped" | "failed"
     | "fresh" | "stale" | "missing-baseline";           // closed set, DESIGN §12.2
   export interface ShotResult { id: string;
     output: string;                                     // normalized root-relative shot.output (POSIX)
     status: ShotStatus; changedRatio: number | null;
     reason: string | null; durationMs: number; diffArtifact: string | null; }
   export interface RunSummary { command: "update" | "check"; root: string; startedAt: string;
     durationMs: number; shots: ShotResult[]; counts: Record<string, number>; exitCode: 0 | 1 | 2 | 3; }
   ```
2. `export async function runShots(opts: { cfg: LoadedConfig; mode: "update" | "check";
   shotFilter: string[]; force: boolean; log: Logger }): Promise<RunSummary>`:
   a. Validate `shotFilter`: unknown ids → `FreshshotError("USAGE", …)` listing valid ids
      (exit 2; DESIGN §14.1).
   b. Selected shots = filter (empty ⇒ all), preserving config order. `skip: true` shots →
      immediate `skipped` result (never launched).
   c. `prepareRunDirs` (issue 17); `withServer` (issue 08); inside: `launchBrowser` (issue 09)
      and `loadHooks` (issue 13) once; finally-close the browser.
   d. Per shot (sequential, config order):
      - `captureShot` (issue 15); on `FreshshotError` → status `failed`,
        `reason = formatError` first line, continue to the next shot.
      - Read baseline: `fs.readFile(outputAbs)`; `ENOENT` → `null`; any other fs error →
        status `failed` with the error message (do not throw).
      - `compareImages` (issue 16); `baseline-undecodable` → emit
        `BASELINE_DECODE_FAILED` warning via `log.warn`.
      - `persistOutcome` (issue 17). `WRITE_FAILED` propagates (aborts the run — disk problems
        are environmental, exit 3).
   e. Aggregate counts by status plus `counts.total` = number of selected shots (DESIGN §14.3
      summary shape); `exitCode`: any `failed` (or check-mode `stale`/`missing-baseline`) → 1,
      else 0. Environmental errors are thrown, not encoded (the CLI layer maps them to 3 via
      issue 02).
3. `update.ts` command action:
   - Build `CliContext` → `const cfg = await loadConfig(…)` →
     `runShots({ cfg, mode: "update", shotFilter, force, log })`.
   - Human report (minimal until issue 24): one line per shot
     `  <status-padded>  <id>  (<ratio %> | reason)` via `log.info`, then a summary line; warnings
     already flow through `log.warn`.
   - `--json` → emit the §14.3 document via `writeJsonDocument` (fields: `schemaVersion: 1`,
     `command`, `root`, `startedAt`, `durationMs`, `shots`, `summary` = counts, `exitCode`).
     Issue 24 golden-tests and freezes this schema; keep field names exactly as §14.3.
   - `process.exitCode = summary.exitCode`.
4. Ordering guarantee: results array preserves config order regardless of failures.

## Acceptance Criteria

Fixture project (temp dir: config in static mode over `tests/fixtures/site`, two shots incl.
one element capture with a mask):

- [ ] First run: all shots `new`, PNGs exist at configured outputs, exit 0.
- [ ] Second run immediately after: all `unchanged`, output bytes untouched, exit 0
      (runner-level determinism check).
- [ ] Mutate the fixture site (CSS color change) → third run: affected shot `updated` with
      `changedRatio > 0`, diff artifact exists; unaffected shot `unchanged`; exit 0.
- [ ] A shot with a bad selector → that shot `failed` (message includes step attribution),
      other shots still processed, exit 1.
- [ ] `--shot home` runs only `home`; `--shot nope` → `USAGE` error, exit 2, valid ids listed.
- [ ] `--force` → all selected shots `forced` and rewritten.
- [ ] `skip: true` shot → `skipped`, no browser context created for it (assert via context
      count or capture spy).
- [ ] `--json` output parses; `shots[].status/changedRatio/diffArtifact` populated as specified;
      nothing but the JSON document on stdout.
- [ ] Server and browser are closed after success and after a thrown `WRITE_FAILED`
      (no lingering listeners/processes).

## Validation

- Browser-project tests in CI (they exercise real Chromium + static server).

## Dependencies

- 05, 08, 15, 17 (and transitively 16).

## Non-goals

- `check` specifics (issue 19). Final human formatting & JSON golden freeze (issue 24).
  Docs coverage (issues 20–22).

## Design References

- DESIGN §4.2 (flow), §12.2 (statuses), §14.1/§14.3/§14.4 (CLI contract), §16 (error routing)
