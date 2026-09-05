# Title

`check` command

## Summary

Wire `src/cli/commands/check.ts` on the shared runner (issue 18) in `mode: "check"`: capture and
compare without ever writing outputs, report `fresh`/`stale`/`missing-baseline` statuses, and
exit 1 when the docs' committed screenshots are out of date — the CI freshness gate.

## Context

`check` is the command teams put in CI (DESIGN §3.3). Its contract: read-only for committed
files (diff artifacts under `.freshshot/` are allowed), status semantics per §12.2, exit codes
per §14.4.

## Scope

- `src/cli/commands/check.ts` (replace stub), small runner adjustments if any surface in
  `mode: "check"`, `tests/browser/runner-check.test.ts`.

## Detailed Requirements

1. Command action mirrors `update.ts` with `mode: "check"`, no `--force` flag (rejected by
   commander as unknown), `--shot` supported, `--json` supported (same §14.3 schema with
   `command: "check"`).
2. Exit code: any `stale`, `missing-baseline`, or `failed` → 1; all `fresh`/`skipped` → 0
   (issue 18's aggregate already encodes this; verify for check mode).
3. Corrupt (undecodable) baseline → status `missing-baseline` with reason
   `baseline-undecodable` (issue 17 mapping), exit 1, no writes outside `.freshshot/`.
4. Human output: stale shots list their `changedRatio` and diff artifact path; a final hint
   line `run 'freshshot update' to refresh N out-of-date screenshot(s)` where
   `N = staleCount + missingBaselineCount`, emitted iff `N > 0` (not emitted when failures are
   the only cause of exit 1).
5. Guarantee: no file under the project root outside `.freshshot/` is created, modified, or
   deleted by a check run — enforced by test comparing a full recursive directory hash of the
   fixture before/after (excluding `.freshshot/`).

## Acceptance Criteria

- [ ] Fresh fixture (after an `update`) → all `fresh`, exit 0.
- [ ] Mutated app → `stale` with ratio and existing diff artifact; baseline bytes untouched;
      exit 1; hint line present.
- [ ] Deleted baseline file → `missing-baseline`, exit 1.
- [ ] Corrupted (truncated) baseline → `missing-baseline` with reason `baseline-undecodable`,
      exit 1, baseline bytes untouched.
- [ ] `check --json` emits exactly one JSON document on stdout and nothing else on stdout;
      human diagnostics appear on stderr only.
- [ ] Failing shot (bad selector) → `failed`, exit 1, other shots evaluated.
- [ ] Read-only guarantee test (directory hash) passes for the stale scenario.
- [ ] `--json` document has `command: "check"` and check-mode statuses; exit code mirrored in
      the document.

## Validation

- Browser-project tests in CI.

## Dependencies

- 18.

## Non-goals

- Coverage policy (issue 22 owns docs-related CI gating). Auto-update or PR creation (v2).

## Design References

- DESIGN §3.3 (workflow), §12.2 (check statuses), §14 (CLI/exit codes)
