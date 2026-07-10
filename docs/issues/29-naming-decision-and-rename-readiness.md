# Title

Naming decision and rename readiness (human-gated)

## Summary

Resolve ADR-007: generate and verify name candidates (npm/GitHub availability, collision scan),
present a decision package to the product owner, and — after the human decision — execute the
mechanical rename sweep across package, binary, config filename, state directory, and docs.
The actual first `npm publish` (a human-executed step whose prerequisites issue 28 documents in
RELEASING.md) is blocked until this issue closes.

## Context

The npm name `freshshot` is taken by an abandoned same-concept package (verified 2026-07-10;
see `docs/research/competitive-landscape.md`). ADR-007 locked: design name-independently,
decide before publishing. This issue has an explicit human decision point — the agent prepares
evidence and executes the outcome, but never chooses the name.

## Scope

- Research artifact `docs/research/naming-candidates.md`; `docs/decisions/ADR-008-final-name.md`
  (status `proposed` until owner accepts); after acceptance: the rename sweep commit(s) and a
  GitHub repo rename checklist for the owner.

## Detailed Requirements

1. Candidate generation: ≥ 10 candidates. Seed list to verify and extend:
   `docshot`, `shotdoc`, `docsnap`, `snapdocs`, `evershot`, `screenfresh`, `docfresh`,
   `freshframe`, `stillshot`, `shotkeeper`. Selection criteria (score each 1–5 in a table):
   pronounceable, evokes "docs + screenshots + freshness", short (≤ 10 chars), no trademark
   red flags from a basic web search, npm-unscoped available, GitHub availability (no exact
   `github.com/<any-owner>/<name>` collision with an **active same-purpose** repo within the
   top 10 `gh search repos` results).
2. Availability verification per candidate, recorded with command + date:
   - `npm view <name>` (must 404 for "available");
   - `gh search repos <name> --limit 10` (note collisions and their activity);
   - optional: obvious domain check (`<name>.dev` via whois/registrar search) — informational.
3. Decision package: `docs/research/naming-candidates.md` with the scored table and a
   recommended top-3; `ADR-008-final-name.md` drafted with the recommendation, consequences
   (repo rename, npm name, config filename `<name>.config.yaml`, state dir `.<name>/`), and
   status `proposed`.
4. **HUMAN GATE**: present the package to the owner (A/B/C choice). Do not proceed past this
   point without an explicit answer. Record the decision + date in ADR-008 (status `accepted`).
5. Rename sweep (only after acceptance; single focused change):
   - `package.json` `name`/`bin`/`repository`/`homepage`; the config-filename constants and
     discovery list (issue 04); the state-directory constant `.freshshot/` (issue 17) and
     `.gitignore` template (issue 23); CLI program name (issue 05); all docs (`README.md`,
     `docs/reference/**`, templates).
   - Keep a compatibility note ONLY if the owner asks; default is a clean cut (pre-1.0, no
     users yet).
   - Canonical planning docs are NOT rewritten; instead each of `docs/DESIGN.md`,
     `docs/ISSUE_PLAN.md`, and `docs/decisions/ADR-007-*.md` receives a one-line banner
     immediately under its H1:
     `> Renamed: the working name "freshshot" in this document refers to the published product "<final-name>" (ADR-008).`
     — canonical docs stay non-stale without history rewriting (ADR-007).
   - Acceptance: `git grep -i freshshot` returns matches only in `docs/research/**`,
     `docs/decisions/**`, and historical planning docs (`docs/DESIGN.md`, `docs/ISSUE_PLAN.md`,
     `docs/issues/**` keep the working name as history); zero matches in
     `src/**`, `tests/**`, `package.json`, `README.md`, `docs/reference/**`.
   - Full test suite + pack-smoke green after the sweep.
6. Owner checklist (manual): GitHub repo rename (redirect preserved), update any external
   links, npm org/scope prep if a scoped fallback was chosen.

## Acceptance Criteria

- [ ] `naming-candidates.md`: ≥ 10 candidates, all with dated npm + GitHub evidence, scored
      table, top-3 recommendation.
- [ ] ADR-008 drafted (`proposed`), then updated to `accepted` with the owner's recorded choice.
- [ ] Human decision explicitly obtained and quoted — the agent did not self-select.
- [ ] Post-sweep: grep criterion above holds; CI + pack-smoke green; `--version` and `init`
      show the new name.
- [ ] If `RELEASING.md` exists, its naming-gate prerequisite checkbox flips to done; otherwise
      ADR-008 records the decision as the publish-gate evidence for issue 28 to consume.

## Validation

- Reviewer re-runs `npm view <chosen>` (still available at execution time) immediately before
  the sweep and again before the actual publish.

## Dependencies

- None. Sequencing guidance (not a dependency): the research half can run at any time; execute
  the rename sweep as the final pre-publish step so that docs (26) and packaging (28)
  artifacts, where already present, are swept together. Blocks: the actual first `npm publish`.

## Non-goals

- Logo/branding, domains purchase, trademark filing. Rewriting historical design docs to the
  new name.

## Design References

- ADR-007, DESIGN §19 (pre-publish gates), `docs/research/competitive-landscape.md`
