# Title

Reporter: human output and stable JSON schema

## Summary

Finalize `src/cli/reporter.ts`: polished, consistent human-readable reports for
`update`/`check`/`coverage`/`list` and the frozen, golden-tested `schemaVersion: 1` JSON
documents (DESIGN §14.2–14.3), replacing the minimal per-command formatting from issues 18/19/22.

## Context

The reporter is the product's face (humans) and its integration contract (CI parsing `--json`).
Freezing the JSON schema behind golden tests makes accidental breaking changes to CI consumers
impossible to merge silently.

## Scope

- `src/cli/reporter.ts` (extend), refactors in `commands/{update,check,coverage,list}.ts` to
  route ALL end-of-run presentation through the reporter,
  `tests/unit/reporter.test.ts`, `tests/integration/json-golden.test.ts` with golden files
  under `tests/fixtures/golden/`.

## Detailed Requirements

1. Human report (stderr), uniform across commands:
   - Per-shot line format: `<STATUS-6-padded> <id>  <detail>` where detail is
     `Δ<percent with 3 significant digits>` for compared shots, the reason for
     failed/dimension-mismatch, and the diff artifact path for updated/stale.
   - Status → color mapping (only when color enabled): new/updated cyan, unchanged/fresh green,
     forced magenta, skipped dim, failed/stale/missing-baseline red, warnings yellow.
   - Summary block: one line `N total: X updated, Y unchanged, …` (only non-zero buckets)
     plus elapsed time; check-mode hint line (issue 19) moves here.
   - Coverage report formatting from issue 22 moves here unchanged (single source).
   - Everything passes through `sanitizeText`; no line exceeds one ref/shot (grep-friendly).
2. JSON emitter:
   - `emitJson(summaryModel)` produces the §14.3 document; **key order fixed** (serialize via
     explicit object construction, not spread of internal state) so goldens are byte-stable.
   - `startedAt` ISO-8601 UTC with milliseconds; `durationMs` integers.
   - Exactly one document on stdout; trailing newline; nothing else ever printed to stdout.
3. Golden tests:
   - For each command, run the built CLI on the shared fixture project, capture stdout,
     normalize volatile fields (`root` → `<ROOT>`, `startedAt` → `<TS>`, all `durationMs` → 0,
     port numbers in messages → `<PORT>`) with a reusable normalizer, and compare against
     committed golden files.
   - A `SCHEMA.md`-style doc is NOT created; the golden files + §14.3 are the contract
     (documenting this in the test header).
4. Non-TTY behavior: without a TTY, human output has no colors and no spinner-like updates —
   plain lines only (CI logs must be clean); assert via captured stderr in integration tests.
5. `--quiet`: per-shot lines suppressed; summary + errors + warnings still print; JSON
   unaffected.

## Acceptance Criteria

- [ ] Golden JSON tests exist and pass for `update` (mixed statuses incl. failed),
      `check` (stale + fresh + missing-baseline), `coverage` (all categories), `list`.
- [ ] Byte-stability: two consecutive runs produce identical normalized stdout.
- [ ] Human snapshot tests (normalized) for the same four scenarios; colored variant asserted
      to contain ANSI codes only when forced TTY+color.
- [ ] `--quiet` and `--verbose` behaviors verified for `update`.
- [ ] Injecting a shot id containing a fake ANSI sequence (via config fixture) shows the
      sanitized form in both human and JSON outputs.
- [ ] No command writes anything to stdout without `--json` (integration assert across all
      four commands).

## Validation

- CI green; goldens reviewed by a human once at PR time (they are the contract).

## Dependencies

- 18, 19, 22.

## Non-goals

- Machine-readable formats beyond JSON (JUnit/SARIF — v2 candidates). Localization.

## Design References

- DESIGN §14.2 (channels), §14.3 (schema + additive-only rule), §17.7 (hygiene)
