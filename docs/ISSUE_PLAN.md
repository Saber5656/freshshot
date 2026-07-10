# freshshot — v1 Issue Plan

Status: canonical execution plan derived from `docs/DESIGN.md`. GitHub Issues are generated from
`docs/issues/NN-*.md`; when they disagree, these files win.

## 1. v1 completion statement

**v1 is complete when issues 01–29 are all completed and validated.** At that point the product
is: an installable, ESM-only Node.js ≥ 20 CLI (working name `freshshot`) providing `init`,
`update`, `check`, `coverage`, and `list` exactly as specified in DESIGN §14; driving Chromium
via Playwright with the determinism toolkit (§10), step DSL (§7), hooks (§8), three server modes
(§9), perceptual-diff-gated writes (§12), and Markdown coverage scanning (§13); with the
security model (§17) implemented and audited, the test strategy (§18) green in CI including the
determinism invariant, user documentation and SECURITY.md published in-repo, and an npm package
that passes the pack smoke test. The only intentionally-open item after 01–29 is executing the
actual first `npm publish`, which is gated on the naming decision (issue 29, ADR-007) and on
human-managed npm credentials. Newly discovered implementation unknowns (§8 below) may add
issues; nothing else remains.

## 2. Issue list in recommended execution order

| # | Issue file | Title | Wave | Depends on | DESIGN |
|---|---|---|---|---|---|
| 01 | `issues/01-project-scaffolding-and-ci.md` | Project scaffolding, toolchain, and CI pipeline | 0 | — | §5, §18 |
| 02 | `issues/02-error-taxonomy-and-exit-codes.md` | Core error taxonomy and exit-code contract | 0 | 01 | §16, §14.4 |
| 03 | `issues/03-path-safety-module.md` | Filesystem root-confinement module | 0 | 01, 02 | §17.3 |
| 04 | `issues/04-config-schema-and-loader.md` | Config schema, loader, and semantic validation | 0 | 01, 02, 03 | §6 |
| 05 | `issues/05-cli-skeleton-and-global-flags.md` | CLI skeleton, global flags, output channels | 0 | 01, 02, 04 | §14.1–14.2, §14.4 |
| 06 | `issues/06-static-server-mode.md` | Static directory server mode | 1 | 01, 02, 03 | §9.1, §9.3, §17.8 |
| 07 | `issues/07-command-server-mode.md` | Managed command server mode | 1 | 01, 02 | §9.1–9.2, §17.7 |
| 08 | `issues/08-server-mode-resolution.md` | Server mode resolution and lifecycle wiring | 1 | 04, 06, 07 | §9.1, §9.4 |
| 09 | `issues/09-browser-session-manager.md` | Chromium session and context factory | 2 | 01, 02, 04 | §5, §10(1), §11 |
| 10 | `issues/10-determinism-toolkit.md` | Determinism toolkit | 2 | 09 | §10 |
| 11 | `issues/11-step-dsl-navigation-and-waits.md` | Step DSL: interpreter core, goto, waits, origin policy | 2 | 02, 04, 09 | §7, §17.4 |
| 12 | `issues/12-step-dsl-interactions.md` | Step DSL: interaction steps | 2 | 11 | §7 |
| 13 | `issues/13-hooks-loader-and-executor.md` | Hooks module loader and executor | 2 | 03, 04, 09 | §8 |
| 14 | `issues/14-screenshot-capture-engine.md` | Screenshot capture engine (viewport/element/fullPage/mask) | 2 | 09 | §11 |
| 15 | `issues/15-single-shot-orchestrator.md` | Single-shot orchestrator | 2 | 10, 11, 12, 13, 14 | §4.2, §11, §16 |
| 16 | `issues/16-png-perceptual-diff-engine.md` | PNG decode and perceptual diff engine | 3 | 01, 02 | §12.1 |
| 17 | `issues/17-gated-atomic-writer-and-diff-artifacts.md` | Gated atomic writer and diff artifacts | 3 | 03, 16 | §12.2, §15 |
| 18 | `issues/18-update-command.md` | `update` command and run orchestration | 3 | 05, 08, 15, 17 | §4.2, §12.2, §14 |
| 19 | `issues/19-check-command.md` | `check` command | 3 | 18 | §12.2, §14 |
| 20 | `issues/20-markdown-image-reference-scanner.md` | Markdown image reference scanner | 4 | 01, 02, 03 | §13.1, §17.6 |
| 21 | `issues/21-coverage-classification-engine.md` | Coverage classification engine | 4 | 04, 20 | §13.2 |
| 22 | `issues/22-coverage-and-list-commands.md` | `coverage` and `list` commands | 4 | 05, 21 | §13.3, §14 |
| 23 | `issues/23-init-command.md` | `init` command | 4 | 05 | §14.1, §15 |
| 24 | `issues/24-reporter-human-and-json.md` | Reporter: human output and stable JSON schema | 5 | 18, 19, 22 | §14.2–14.3, §17.7 |
| 25 | `issues/25-e2e-test-suite-and-fixture.md` | End-to-end CLI test suite and fixture project | 5 | 18, 19, 22, 23, 24 | §18 |
| 26 | `issues/26-user-documentation.md` | User documentation (README, references, CI recipe) | 6 | 25 | §3, §6–§14, §17.5 |
| 27 | `issues/27-security-policy-and-hardening-audit.md` | SECURITY.md and security hardening audit | 6 | 25 | §17 |
| 28 | `issues/28-npm-packaging-and-release-workflow.md` | npm packaging and release workflow | 6 | 25 | §19 |
| 29 | `issues/29-naming-decision-and-rename-readiness.md` | Naming decision and rename readiness (human-gated) | 6 | — | §19, ADR-007 |

## 3. Dependency graph (summary)

```
01 ─▶ 02 ─▶ 03 ─▶ 04 ─▶ 05
              │      ├──────────────▶ 08 ◀── 06 ◀─ 03
              │      │                 ▲
              │      │                 └───── 07 ◀─ 02
              │      └▶ 09 ─▶ 10 ──┐
              │          ├─▶ 11 ─▶ 12 ─┤
              ├──────────┼─▶ 13 ───────┼─▶ 15 ─┐
              │          └─▶ 14 ───────┘       ├─▶ 18 ─▶ 19 ─┐
02 ─▶ 16 ─▶ 17 ────────────────────────────────┘             ├─▶ 24 ─▶ 25 ─▶ 26,27,28
03 ─▶ 20 ─▶ 21 ─▶ 22 ────────────────────────────────────────┘
05 ─▶ 23 ─────────────────────────────────────────▶ 25        29 (independent; gates publish)
```

Within a wave, issues without mutual dependencies are parallelizable (e.g. 06‖07, 09–14 partially,
20 ‖ 16, 23 ‖ 22).

## 4. Implementation waves

| Wave | Issues | Gate to exit the wave |
|---|---|---|
| 0 Foundations | 01–05 | CI green: lint, typecheck, unit; `freshshot --version` runs; config fixtures validate |
| 1 App environment | 06–08 | All three server modes start/stop in integration tests, incl. traversal + teardown cases |
| 2 Capture pipeline | 09–15 | One real shot captured against the fixture page in CI with determinism setup applied |
| 3 Diff & runner | 16–19 | `update`/`check` lifecycle passes on the fixture; statuses and exit codes exact |
| 4 Docs coverage | 20–23 | Coverage classification matches the fixture's golden report; `init` idempotence tests |
| 5 UX & E2E | 24–25 | Full e2e lifecycle + **determinism invariant** (2nd `update` all-`unchanged`) required in CI |
| 6 Release readiness | 26–29 | Docs complete; security audit checklist closed; `npm pack` smoke passes; naming gate decided |

## 5. Coverage: DESIGN.md sections → issues

| DESIGN section | Covered by |
|---|---|
| §4 Architecture / §4.2 data flow | 15, 18 (and structure of all module issues) |
| §5 Runtime & dependencies | 01 |
| §6 Configuration | 04 (path rules with 03) |
| §7 Step DSL | 11, 12 |
| §8 Hooks | 13 |
| §9 Server modes | 06, 07, 08 |
| §10 Determinism | 09 (context options), 10 |
| §11 Capture | 09, 14, 15 |
| §12 Diff & update policy | 16, 17, 18, 19 |
| §13 Docs scanning & coverage | 20, 21, 22 |
| §14 CLI | 05, 18, 19, 22, 23, 24 |
| §15 Storage layout | 17, 23 |
| §16 Error taxonomy | 02 (consumed by all) |
| §17 Security model | 03 (17.3), 06 (17.8), 11 (17.4), 13 (17.2), 20 (17.6), 24 (17.7), 27 (audit + SECURITY.md), 28 (17.9) |
| §18 Testing strategy | per-issue Validation sections + 25 |
| §19 Packaging & release | 28, 29 |
| §20 v2 deferred / §21 unknowns | tracked below (§7, §8) |

Every externally reachable boundary in §17 has a named owning issue; security acceptance
criteria are embedded in those issues so implementation agents cannot skip them.

## 6. Validation strategy (whole product)

1. **Per-issue**: every issue ships with unit/integration tests listed in its Validation section;
   CI (from issue 01) must stay green after every issue.
2. **Wave gates**: table in §4; a wave is not done until its gate test exists and passes in CI.
3. **Cross-cutting invariants** (encoded as required CI tests by issue 25):
   - Determinism: second consecutive `update` → 100 % `unchanged` (ADR-005).
   - Exit-code contract table (§14.4) verified end-to-end per command.
   - `--json` outputs validate against the schema fixtures (golden files, normalized).
   - Security: path-traversal and origin-policy attack tests must exist and pass (03, 06, 11).
4. **Coverage thresholds**: vitest global ≥ 85 % lines; `core/paths.ts`, `diff/compare.ts`,
   `scan/*`, `config/*` ≥ 95 % (enforced from issue 01, tightened as modules land).
5. **Release gate**: 27 audit checklist + 28 pack smoke + 29 naming decision before any publish.

## 7. Deferred to v2 (not planned as issues)

GitHub Action + auto-PR; auth (storageState) + secret-safe env interpolation; Firefox/WebKit;
theme/locale matrices; MDX/AsciiDoc/reST; watch mode; parallel workers; retries/stabilization;
oxipng optimization; JS/TS config; static-site path aliases; sub-resource origin blocking;
programmatic API; advanced scroll steps. (DESIGN §20.)

## 8. Known unknowns that may create additional issues

| # | Unknown (DESIGN §21) | Likely trigger point |
|---|---|---|
| 1 | Playwright clock vs framework schedulers | issue 10 implementation |
| 2 | Readiness probe for redirecting apps (status allow-list) | issue 07 |
| 3 | Inline-HTML edge cases in Markdown extraction | issue 20 |
| 4 | Windows process-tree teardown reliability | issue 07 (Windows CI is smoke-only in v1) |
| 5 | pixelmatch thresholds at deviceScaleFactor 2 | issues 16/25 tuning |
| 6 | npm name availability at publication | issue 29 |

Discovery protocol: when an unknown materializes, the implementing agent files a new
`docs/issues/NN-*.md` (next free number) referencing the DESIGN section it refines, updates this
plan's §2 table, and only then creates the GitHub Issue.
