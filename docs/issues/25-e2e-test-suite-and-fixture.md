# Title

End-to-end CLI test suite and fixture project

## Summary

Build the product-level e2e suite: a realistic fixture project exercised through the **built**
CLI binary across the full lifecycle — `init` → `update` (new) → `update` (unchanged, the
determinism invariant) → app mutation → `update` (updated) → `check` (stale) → `coverage` —
asserting files, statuses, exit codes, and `--json` documents (DESIGN §18, ISSUE_PLAN §6).

## Context

Unit/integration tests prove modules; this suite proves the product story and pins the
cross-cutting invariants: the ADR-005 determinism invariant (second update fully `unchanged`)
and the §14.4 exit-code contract, end to end, in CI, on every commit.

## Scope

- `tests/e2e/lifecycle.test.ts`, `tests/e2e/exit-codes.test.ts`,
  fixture app `tests/fixtures/e2e-app/` (static site: two pages, deterministic layout,
  system fonts, one interactive panel, one masked "dynamic" region),
  fixture docs (`README.md` + `docs/guide.md` with managed/unmanaged/broken refs and one
  orphan-producing shot), plus an `npm run test:e2e` script and CI wiring in the browser job.

## Detailed Requirements

1. Suite setup per test file: copy the fixture project into a fresh temp dir, run the built CLI
   with execa (`node <repo>/dist/cli/main.js`, `cwd` = temp project). Never mutate the repo's
   own fixture copy. Build once via a global setup (`npm run build` precondition documented).
2. Lifecycle test (single serial test, ordered phases with assertions after each):
   a. `init --force` variant NOT used: fixture ships its own config (init is covered by issue
      23); instead assert `--config` discovery works from the project root.
   b. `update --json` → all shots `new`, exit 0, PNGs exist, JSON golden-normalized.
   c. `update --json` again → **every shot `unchanged`, exit 0** — the determinism invariant;
      the test MUST fail the suite loudly if any shot reports otherwise (custom assertion
      message referencing ADR-005).
   d. Mutate the app (sed-style CSS color swap in the temp copy) → `update` → affected shot
      `updated` (ratio > gate), unaffected `unchanged`; diff artifact exists.
   e. Revert file mutation; corrupt one baseline (truncate) → `update` → that shot `new` with
      `baseline-undecodable` warning on stderr.
   f. Mutate app again → `check` → `stale`, exit 1, baselines untouched (hash check), diff
      artifact present.
   g. Delete one baseline → `check` → `missing-baseline`, exit 1.
   h. `coverage --json` → fixture's known managed/unmanaged/broken/orphan sets exactly.
   i. `coverage --fail-on broken,unmanaged` → exit 1; after `update` fixed nothing (coverage is
      docs-level) — assert unchanged result to prove capture and coverage are orthogonal.
3. Exit-code matrix test: table-driven cases for §14.4 — usage error (unknown flag → 2),
   config error (broken YAML → 2), environment error (config pointing at a dead
   `server.baseUrl` → 3), policy failure (check stale → 1), success (→ 0). Each case asserts
   stderr contains the `error[CODE]` tag where applicable.
4. Runtime budget: the whole e2e suite ≤ 4 minutes on ubuntu-latest CI (document measured time
   in the PR; sequential Chromium runs are the cost driver — keep shots per phase ≤ 3).
5. CI: e2e runs inside the existing `browser-tests` job after integration tests; failure
   uploads `.freshshot/diffs/` and the temp project's images as workflow artifacts
   (actions/upload-artifact, SHA-pinned) for debugging.

## Acceptance Criteria

- [ ] Lifecycle phases a–i all assert as specified and pass in CI.
- [ ] Determinism invariant phase (c) is present, labeled, and green 3× consecutively in CI
      (re-run twice manually; note run links in the PR).
- [ ] Exit-code matrix covers 0/1/2/3 with the exact codes asserted.
- [ ] Suite leaves no processes/listeners (execa `cleanup` verified; CI job ends without hang).
- [ ] Artifacts uploaded on failure (verify once by forcing a failure in a scratch commit,
      then revert — link the red run in the PR).
- [ ] Total e2e wall time reported and ≤ 4 min in CI logs.

## Validation

- CI green on ubuntu + the macOS smoke job (macOS runs unit/integration only if minutes are a
  concern — but e2e MUST run on ubuntu; document the choice in the workflow comment).

## Dependencies

- 18, 19, 22, 23, 24.

## Non-goals

- Performance benchmarking; Windows e2e (known unknown #4); npm-pack smoke (issue 28).

## Design References

- DESIGN §18 (test strategy), §14.4 (exit codes), ADR-005 (invariant), ISSUE_PLAN §6
  (validation strategy)
