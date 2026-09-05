# Title

PNG decode and perceptual diff engine

## Summary

Implement `src/diff/compare.ts`: decode baseline/candidate PNGs, compute the changed-pixel
ratio with pixelmatch, produce a visual diff image, and classify the comparison per DESIGN
§12.1 — including the dimension-mismatch and undecodable-baseline policies.

## Context

The diff gate is the product's core promise (ADR-005): no noise commits. This module is pure
computation (no filesystem), so it must be exhaustively unit-tested with synthetic images —
it is one of the ≥95 % coverage modules.

## Scope

- `src/diff/compare.ts`, `tests/unit/diff-compare.test.ts`, synthetic PNG helpers in
  `tests/helpers/png.ts` (generate solid/checkerboard/one-pixel-changed images via pngjs).

## Detailed Requirements

1. Types and API:
   ```ts
   export type CompareOutcome =
     | { kind: "new" }                                       // no baseline bytes given
     | { kind: "baseline-undecodable" }                      // treat as new + warning (§12.1)
     | { kind: "unchanged"; changedRatio: number }
     | { kind: "changed"; changedRatio: number; reason: "pixels" | "dimension-mismatch";
         diffPng: Buffer | null }
   export function compareImages(
     baseline: Buffer | null,
     candidate: Buffer,
     opts: { threshold: number; maxDiffPixelRatio: number },
   ): CompareOutcome
   ```
2. Decoding: `PNG.sync.read`. Candidate failing to decode is a programming error → throw
   `FreshshotError("CAPTURE_FAILED", …)`. Baseline failing to decode → `baseline-undecodable`
   (caller emits the `BASELINE_DECODE_FAILED` warning and treats as `new`; DESIGN §12.1).
3. Dimension mismatch → `{ kind: "changed", changedRatio: 1, reason: "dimension-mismatch",
   diffPng: null }` (no pixel diff is computable; DESIGN §12.1).
4. Same dimensions → `pixelmatch(base.data, cand.data, diff.data, w, h,
   { threshold: opts.threshold, includeAA: false })`;
   `changedRatio = diffPixels / (w * h)` (floating point, not rounded).
   - `changedRatio > opts.maxDiffPixelRatio` → `changed` with `reason: "pixels"` and `diffPng`
     encoded from the pixelmatch output buffer.
   - else `unchanged` (still reporting the ratio — the reporter shows it).
   - Boundary: ratio exactly equal to the gate is `unchanged` (strict `>`; DESIGN §12.2 table).
5. Keep helpers (e.g. PNG encoding of the diff image) private to the module; the exported
   surface of `src/diff/compare.ts` is `compareImages` plus the outcome types only, and nothing
   from this module is re-exported via `src/index.ts` or package exports (DESIGN §2.2: no
   public programmatic API).
6. No filesystem access; buffers in, outcome out.

## Acceptance Criteria

- [ ] Identical 100×100 images → `unchanged`, `changedRatio === 0`.
- [ ] One changed pixel in 100×100 (ratio 0.0001) with gate 0.001 → `unchanged`,
      ratio ≈ 0.0001 (assert within 1e-9).
- [ ] 2 % changed block with gate 0.001 → `changed`/`pixels`; `diffPng` decodes to 100×100.
- [ ] Ratio exactly at the gate (e.g. 10 px / 10000 px, gate 0.001) → `unchanged`
      (strict-greater contract).
- [ ] `threshold` behavior: a moderate per-pixel delta (start with RGB +20 and calibrate
      empirically against pixelmatch's YIQ color metric, keeping the same two-threshold
      contrast) counts as changed at `threshold: 0.05` and as unchanged at `threshold: 0.3`
      (documents what the knob does).
- [ ] 100×100 vs 100×101 → `changed`/`dimension-mismatch`, ratio 1, `diffPng === null`.
- [ ] `baseline: null` → `new`; truncated-bytes baseline → `baseline-undecodable`; truncated
      candidate → throws `CAPTURE_FAILED`.
- [ ] Line coverage of `compare.ts` ≥ 95 %.

## Validation

- Unit tests only (pure module) — green in CI.

## Dependencies

- 01, 02.

## Non-goals

- Writing files, diff artifact paths (issue 17). Threshold auto-tuning (known unknown #5).

## Design References

- DESIGN §12.1 (comparison), §12.2 (strict-greater gate), ADR-005, §16
  (`BASELINE_DECODE_FAILED` semantics)
