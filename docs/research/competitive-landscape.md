# Competitive Landscape: Documentation Screenshot Freshness

Status: verified 2026-07-10 (facts below were checked against live GitHub/npm on this date)

## Why this research exists

freshshot's concept — "automatically re-capture UI screenshots embedded in docs via E2E automation" —
overlaps with several existing tools. This document records what already exists, what gap remains,
and which proven ideas we deliberately adopt. It directly informs `docs/DESIGN.md` and
`docs/decisions/ADR-001` / `ADR-002` / `ADR-007`.

## Tools surveyed

| Tool | Stack | Stars / activity (2026-07-10) | What it does | What it lacks for our use case |
|---|---|---|---|---|
| [mphinance/freshshot](https://github.com/mphinance/freshshot) (npm `freshshot` v0.1.0) | JS + Playwright | 0 stars; created 2026-05-22, **development stopped the same day** | README screenshot freshness: serve a static folder or hit a `baseUrl`, declarative JSON steps, determinism (seed `Math.random`, freeze animations, wait for fonts, `settleMs`), perceptual-diff gate (default 0.15%) so only real changes are rewritten | No docs awareness at all (writes PNGs to an `outDir`; never reads Markdown), no managed dev-server command, no coverage concept, JSON-only config, abandoned. **Occupies the npm name `freshshot`** |
| [simonw/shot-scraper](https://github.com/simonw/shot-scraper) (release 1.10) | Python + Playwright | 2,494 stars; active (pushed 2026-06-30) | General screenshot CLI: single shots or YAML `multi` batches, JS injection, auth via saved storage state, widely used with the `shot-scraper-template` GitHub Actions pattern to keep README images updated | No perceptual-diff gate (every run rewrites bytes → noisy git history unless paired with extra tooling), no docs scanning/coverage, no dev-server lifecycle management, Python toolchain (friction for JS/TS projects) |
| [doc-detective/doc-detective](https://github.com/doc-detective/doc-detective) (v4.26.2) | Node | 125 stars; very active (pushed 2026-07-10) | Documentation *testing* framework: specs of user actions (`goTo`, `click`, `typeKeys`, `saveScreenshot`, …) validate that documented procedures still work; can save screenshots with a variation threshold | Screenshot freshness is a side feature of a much heavier "test your docs" concept; no repo-wide image↔recipe coverage model; spec format optimized for doc testing, not image asset management |
| [lost-pixel/lost-pixel](https://github.com/lost-pixel/lost-pixel) (v3.22.0) | Node | 1,682 stars; active (pushed 2026-04-22) | OSS visual regression (Percy/Chromatic alternative) for Storybook/pages; baseline approval workflow in CI | Solves "detect unintended UI change", not "keep doc assets current"; baselines are test artifacts, not the images your docs embed |
| Playwright `expect(page).toHaveScreenshot()` | built-in | — | DIY snapshot assertions inside a test suite | Baselines are per-OS test fixtures, not docs assets; no docs mapping; requires writing/maintaining a test project |
| Percy / Chromatic / Argos | SaaS | — | Hosted visual review | Paid/hosted review workflow; does not update files referenced by docs |

## Gap analysis

Every surveyed tool stops at "produce/compare screenshots". None of them answer the questions a
docs maintainer actually has:

1. **Which images in my docs are managed?** (image reference in Markdown ↔ capture recipe mapping)
2. **Which images are unmanaged / orphaned / broken?** (coverage as a CI-enforceable policy)
3. **Can one command boot my real app (dev server), drive it, and refresh exactly the files my docs reference — without committing pixel noise?**

That combination — a **docs-aware screenshot freshness manager** — is the open gap and the core
differentiation of this project (see ADR-001, ADR-002).

## Ideas we deliberately adopt (proven elsewhere)

| Idea | Proven by | Our adoption |
|---|---|---|
| Determinism toolkit: disable animations, wait for web fonts, settle delay, seeded randomness | mphinance/freshshot | v1, on by default where safe (DESIGN §10) |
| Perceptual-diff gate: rewrite a committed image only when change exceeds a threshold | mphinance/freshshot, doc-detective (`maxVariation`) | v1 core (DESIGN §12) |
| Declarative multi-shot config in YAML | shot-scraper `multi` | v1 (`freshshot.config.yaml`, DESIGN §6) |
| Saved-auth storage state for logged-in screens | shot-scraper `auth` | **v2** (deferred; DESIGN §20) |
| CI recipe: scheduled/regenerating workflow that opens a PR | shot-scraper-template pattern | v1 documents the recipe; packaged GitHub Action is v2 |

## Naming collision (input to ADR-007)

npm package name `freshshot` is taken by the abandoned mphinance project (v0.1.0, MIT).
Consequence: we design name-independently and gate publication on a rename decision
(see `docs/issues/29-naming-decision-and-rename-readiness.md`).

## Sources

- `npm view freshshot` — npm registry, checked 2026-07-10
- `gh repo view` / README for mphinance/freshshot, simonw/shot-scraper, doc-detective/doc-detective, lost-pixel/lost-pixel — checked 2026-07-10
