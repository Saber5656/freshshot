# ADR-005: Perceptual-diff-gated writes with a determinism toolkit

- Status: accepted (2026-07-10, confirmed with product owner)

## Context

Naive re-capture rewrites bytes every run (fonts anti-alias, animations land differently),
filling git history with noise — the reason teams abandon screenshot automation. Prior art
(mphinance/freshshot, doc-detective `maxVariation`) proves diff-gating works.

## Decision

1. A freshly captured image replaces the committed one only when
   `changedPixels / totalPixels > diff.maxDiffPixelRatio` (default 0.1 %), computed with
   pixelmatch at per-pixel `threshold` 0.1 (DESIGN §12).
2. Determinism is engineered, not hoped for: fixed viewport/locale/timezone, reduced motion,
   animation-freezing CSS, font readiness wait, scrollbar hiding, optional seeded `Math.random`
   and frozen clock (DESIGN §10).
3. The acceptance invariant for the whole product: **a second consecutive `update` run in the
   same environment reports every shot `unchanged`** — enforced as a required CI e2e test.
4. No retries/auto-stabilization in v1: flakiness must surface as a failure, not be averaged away.

## Consequences

- `update` is safe to run on every commit; `check` is a meaningful CI gate.
- Dimension changes always count as changed (ratio 1) — viewport edits intentionally rewrite.
- Threshold defaults may need tuning against Chromium sub-pixel AA at DSF 2 (known unknown #5).

## Alternatives considered

- Always-rewrite (shot-scraper model) — rejected: binary noise, meaningless diffs.
- Structural/DOM diffing — rejected: what docs show is pixels; DOM equality neither necessary
  nor sufficient.
