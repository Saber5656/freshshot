# Title

`coverage` and `list` commands

## Summary

Wire `src/cli/commands/coverage.ts` and `src/cli/commands/list.ts` (replacing stubs): run the
scanner + classification engine, present the report for humans and `--json`, and implement the
`--fail-on` CI policy (DESIGN §13.3). `list` shows the shots table with docs-reference counts.

## Context

These commands surface the docs-aware differentiator to users and CI. They are fast (no browser,
no server) and must work even when the app cannot run — coverage is a pure docs/config check.

## Scope

- `src/cli/commands/coverage.ts`, `src/cli/commands/list.ts`,
  `tests/integration/coverage-command.test.ts`, `tests/integration/list-command.test.ts`
  (built-CLI level via execa, fixture project in temp dir).

## Detailed Requirements

1. `coverage` action: `loadConfig` → `scanDocs` → `classifyCoverage` → report.
   - Human output (stderr): counts summary
     (`managed N / unmanaged N / broken N / orphans N / external N / files N`, where `external`
     is the scanner's `skippedExternal` and `files` the number of scanned docs files —
     DESIGN §13.1), then per
     category the entries as `<docFile>:<line>  <rawRef>` (broken entries append the reason;
     orphans as `shot '<id>' → <output>`). Empty categories print nothing.
   - Scanner `errors` (e.g. oversized file) are listed as warnings and force exit code 1 only
     when `--fail-on` includes `broken` (they undermine coverage guarantees; document this rule
     in `--help` text).
   - `--fail-on <cats>`: comma-separated subset of `unmanaged,broken,orphan`; invalid token →
     `USAGE` (exit 2). Selected non-empty category → exit 1. Default: report-only, exit 0
     (DESIGN §13.3).
   - `--json`: document per §14.3 with the `coverage` object — `managed`/`unmanaged`/`broken`
     entries as `{ ref, docFile, line }` (plus `shotId` for managed and `reason` for broken);
     `orphans` entries as `{ shotId, output }` exactly as §14.3 shows — plus `summary` counts
     (including `external` and `files`) and `exitCode`.
2. `list` action: `loadConfig` → `scanDocs` + `classifyCoverage` (reuse; cheap) → one row per
   shot: `id`, `output`, `refs` (count of managed refs pointing at it), `description?`,
   `skip` flag. Human table on stderr; `--json` emits the full §14.3 envelope
   (`schemaVersion`, `command: "list"`, `root`, `startedAt`, `durationMs`, `summary`,
   `exitCode`) with `shots` entries `{ id, output, description, refs, skip }`.
3. Neither command starts a server or browser; both work with `server.command` configs without
   running the command (assert in tests).
4. Exit codes: coverage per policy above; list always 0 unless config/usage errors.

## Acceptance Criteria

Fixture project with: one managed ref, one unmanaged existing image, one broken ref, one orphan
shot, one external URL image:

- [ ] `coverage` (no flags) exits 0; human output contains all four categories with exact
      `file:line` locations.
- [ ] `coverage --fail-on broken` exits 1; `--fail-on unmanaged,orphan` exits 1;
      after fixing the corresponding entries, exits 0.
- [ ] `coverage --fail-on nope` exits 2 with `USAGE`.
- [ ] `coverage --json` matches the §14.3 shape (golden test with normalized `root`).
- [ ] `list` shows both shots with correct managed-ref counts (1 and 0) and exits 0;
      `list --json` golden-tested.
- [ ] With a `server.command` config whose command would `exit 1` if executed, both commands
      still succeed (proves no server start).
- [ ] Oversized markdown fixture: warning shown; exit 0 without `--fail-on`; exit 1 with
      `--fail-on broken`.

## Validation

- Built-CLI integration tests in CI (execa against `dist/cli/main.js`).

## Dependencies

- 05, 21.

## Non-goals

- Fixing/suggesting config entries automatically (v2). Coverage of non-Markdown docs (v2).

## Design References

- DESIGN §13.3 (policy), §14.1–14.3 (CLI/JSON), §3.4 (CI workflow)
