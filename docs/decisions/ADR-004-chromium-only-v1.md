# ADR-004: Chromium-only capture in v1

- Status: accepted (2026-07-10, confirmed with product owner)

## Context

Playwright supports Chromium, Firefox, and WebKit. Docs screenshots serve illustration, not
cross-browser QA. Every additional engine multiplies rendering nondeterminism (fonts,
anti-aliasing, scrollbars) — the exact thing the diff gate must fight.

## Decision

v1 launches and supports **Chromium only**. The config schema has no browser field in v1
(adding one later is backward-compatible).

## Consequences

- Determinism toolkit and pixel thresholds are tuned for one engine (DESIGN §10, §12).
- CI needs only `npx playwright install chromium` — faster, smaller.
- Cross-browser visual QA is explicitly out of scope; users wanting it should use
  visual-regression tools (lost-pixel, Playwright snapshots).

## Alternatives considered

- Multi-browser from day one — rejected: high determinism cost, near-zero docs value.
