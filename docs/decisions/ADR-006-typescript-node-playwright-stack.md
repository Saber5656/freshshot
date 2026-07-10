# ADR-006: TypeScript + Node.js ≥ 20 + Playwright, distributed on npm

- Status: accepted (2026-07-10, confirmed with product owner)

## Context

Target users maintain web projects; their toolchain is npm-centric. shot-scraper's Python
toolchain is a documented adoption friction for JS teams. Playwright is the de-facto standard
browser automation library with first-class determinism affordances (clock, reducedMotion,
screenshot masks/animations options).

## Decision

- Language: TypeScript (strict), ESM-only. Runtime: Node.js ≥ 20.
- Browser automation: Playwright (Chromium per ADR-004); browsers installed by the user via
  `npx playwright install chromium` (never auto-downloaded by freshshot).
- Distribution: npm package exposing a single `bin`; no public programmatic API in v1.
- Toolchain: tsup (build), vitest (tests + coverage), Biome (lint + format), fast-glob, zod,
  yaml, commander, pixelmatch + pngjs, mdast-util-from-markdown. The runtime dependency list in
  DESIGN §5 is exhaustive; additions require an ADR.

## Consequences

- `npx <name> init` is the entire install story for JS projects.
- ESM-only excludes require() consumers — acceptable, the CLI is the only surface.
- Dependency budget is a reviewable security property (DESIGN §17.9).

## Alternatives considered

- Python (shot-scraper ecosystem) — rejected: audience mismatch.
- Bundling a browser — rejected: size, update cadence, security patch burden.
- Deno/Bun runtime — rejected: Playwright support and CI ubiquity favor Node.
