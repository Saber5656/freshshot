# ADR-002: Central YAML config for recipes; docs are scanned, never annotated

- Status: accepted (2026-07-10, confirmed with product owner)

## Context

Recipes (capture instructions per image) could live in: (A) one central config file, (B) inline
annotations inside Markdown near each image, (C) existing Playwright test files via markers, or
(D) a hybrid. Inline annotations couple docs to tool syntax and scatter execution config across
prose; test-file reuse limits the audience to Playwright users.

## Decision

Recipes live in **one central `freshshot.config.yaml`** (shots list). Markdown is treated as
**read-only input**: the scanner extracts image references to compute coverage but freshshot
never writes to or requires annotations in docs files.

## Consequences

- Single strictly-validated schema (DESIGN §6); one place to review in PRs; hooks/trust model
  attaches to one file.
- Coverage (ADR-001) is what ties docs to recipes — the join key is the image path
  (`shots[].output` == resolved doc reference).
- Renaming an image requires touching config + docs; the `coverage` command surfaces mismatches
  (`orphan`/`broken`) immediately, which is the designed mitigation.

## Alternatives considered

- Inline Markdown annotations — rejected: pollutes docs, per-file parsing/writing complexity,
  weak schema validation, merge conflicts in prose files.
- Playwright test reuse — rejected: excludes non-Playwright projects; hidden coupling to test
  suite structure. May return as a v2 import helper.
