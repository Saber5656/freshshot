# Title

npm packaging and release workflow

## Summary

Make the package publishable: publish-grade `package.json` metadata, an `npm pack` smoke test
that installs and runs the packed tarball in a clean directory, a manually-triggered release
workflow publishing with npm provenance, `CHANGELOG.md`, and `RELEASING.md` documenting the
human steps (tokens, naming gate). Publishing itself is NOT executed in this issue.

## Context

DESIGN §19 defines packaging; ADR-007 gates the actual publish on the naming decision
(issue 29) and credentials are configured by a human (never by an agent). This issue delivers
everything up to — but excluding — pressing the button.

## Scope

- `package.json` (publish fields), `scripts/pack-smoke.mjs`, `.github/workflows/release.yml`,
  `CHANGELOG.md`, `RELEASING.md`, `.npmignore` NOT used (rely on `files`).

## Detailed Requirements

1. `package.json`:
   - Remove `"private": true`. Set `description` (one line matching README's value prop),
     `keywords` (`screenshots, documentation, playwright, visual-diff, docs, ci`),
     `repository` (`git+https://github.com/Saber5656/freshshot.git`), `bugs`, `homepage`,
     `author`, `license: "MIT"`, `files: ["dist", "README.md", "LICENSE", "CHANGELOG.md"]`,
     `publishConfig: { access: "public", provenance: true }`.
   - `version` stays `0.0.0`; the release workflow publishes whatever version a human tagged
     (see RELEASING.md flow below).
   - Working name caveat: name remains `freshshot` until issue 29's sweep; `pack-smoke` must
     not hardcode the name (read it from package.json).
2. `scripts/pack-smoke.mjs` (run via `npm run pack:smoke`):
   - `npm pack` into a temp dir → `npm install <tarball>` in a fresh temp project →
     run `npx <bin> --version` (assert semver output) → `npx <bin> init` (assert config file
     created) → `npx <bin> coverage` against a one-file docs fixture (assert exit 0). No
     browser required (choose commands that don't need Chromium so the smoke runs anywhere).
   - Fails loudly on any nonzero exit; cleans up temp dirs.
3. `.github/workflows/release.yml`:
   - `workflow_dispatch` with an explicit `version` input (must match `package.json` version;
     the job verifies and fails on mismatch) — tagging/version bumps are human git actions
     documented in RELEASING.md, not automated here.
   - Jobs: full CI reuse (lint/typecheck/test/build incl. browser tests) → `pack:smoke` →
     `npm publish` with `--provenance` (uses `permissions: { id-token: write, contents: read }`
     and npm Trusted Publishing if configured, else `NODE_AUTH_TOKEN` from repo secret).
   - All actions SHA-pinned; `npm publish` step guarded by an environment named `release`
     (environment protection rules configured by the human maintainer).
4. `CHANGELOG.md`: Keep a Changelog format, `## [Unreleased]` section listing v1 features
   (one bullet per shipped issue area).
5. `RELEASING.md` (maintainer doc): prerequisites checklist —
   naming gate done (issue 29), security audit gaps zero (issue 27), npm token or Trusted
   Publisher configured **manually by the maintainer**, GitHub `release` environment created;
   then the release steps: bump version + update CHANGELOG → commit → tag `vX.Y.Z` → push →
   run the workflow with the version input → verify npm listing + provenance badge → GitHub
   Release with CHANGELOG excerpt.

## Acceptance Criteria

- [ ] `npm run pack:smoke` passes locally and is wired into CI (ubuntu job) so every commit
      proves packability.
- [ ] Tarball content check: `npm pack --dry-run` file list contains only `dist/**`, README,
      LICENSE, CHANGELOG, package.json (assert in pack-smoke).
- [ ] `release.yml` is valid (actionlint or GitHub's parser via a dry PR), SHA-pinned, with the
      exact permissions block above, and does NOT run on push/PR triggers.
- [ ] `--version` output equals package.json version in the packed install.
- [ ] CHANGELOG and RELEASING exist with the specified structure; RELEASING marks every
      credential/settings step as "human maintainer" explicitly.
- [ ] No publish is performed by this issue (verify npm registry unchanged).

## Validation

- CI green including pack-smoke; a reviewer walks RELEASING.md and confirms each step is
  executable as written (dry read, no publish).

## Dependencies

- 25 (product complete). Publish execution additionally gated by 27 and 29 (documented, not a
  file dependency).

## Non-goals

- Executing `npm publish`; configuring npm/GitHub secrets (human-only per project rules);
  automated version bumping/changelog generation (v2 tooling decision).

## Design References

- DESIGN §19 (packaging & release), §17.9 (supply chain), ADR-006, ADR-007
