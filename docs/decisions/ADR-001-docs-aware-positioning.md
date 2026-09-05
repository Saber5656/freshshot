# ADR-001: Position the product as a docs-aware screenshot freshness manager

- Status: accepted (2026-07-10, confirmed with product owner)
- Deciders: product owner (Saber5656), design agent

## Context

The core mechanic — capture screenshots deterministically and rewrite only on real change — is
already implemented by an abandoned npm package (`freshshot` by mphinance) and partially by
shot-scraper, doc-detective, and visual-regression tools. See
`docs/research/competitive-landscape.md`. Rebuilding only that mechanic would be a re-invention
with no adoption story.

## Decision

The differentiating core of v1 is **docs awareness**: scan Markdown for image references, model
the mapping image ↔ capture recipe, and report/enforce coverage (managed / unmanaged / broken /
orphan). Capture + perceptual-diff gating is adopted as table stakes, not as the differentiator.

## Consequences

- The docs scanner and coverage engine are first-class v1 modules with their own CLI command and
  CI policy flags (DESIGN §13, §14).
- Marketing/README must lead with the coverage story, not "screenshots in CI".
- v1 must keep the capture pipeline competitive (determinism + diff gate) but not exceed it
  (no multi-browser, no auth) — depth goes into docs integration instead.

## Alternatives considered

1. Pure capture tool (competitor parity) — rejected: no differentiation, occupied name space.
2. Full docs-testing framework (doc-detective territory) — rejected: far larger scope, existing
   active competitor, weak fit with "small sharp tool" positioning.
