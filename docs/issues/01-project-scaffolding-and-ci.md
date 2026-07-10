# Title

Project scaffolding, toolchain, and CI pipeline

## Summary

Create the TypeScript/ESM project skeleton, install the exact dependency set from DESIGN §5,
configure build (tsup), tests (vitest + coverage thresholds), lint/format (Biome), and a GitHub
Actions CI workflow. After this issue, every later issue lands on a repo where `npm ci`,
`npm run lint`, `npm run typecheck`, `npm run test`, and `npm run build` all pass locally and in CI.

## Context

freshshot is an ESM-only Node.js ≥ 20 CLI built with TypeScript strict mode (ADR-006). All later
issues assume this toolchain exists and CI is enforcing it. Dependency budget is a security
property (DESIGN §17.9): the runtime dependency list is exhaustive and must not grow silently.

## Scope

- `package.json`, `tsconfig.json`, `tsup.config.ts`, `vitest.config.ts`, `biome.json`,
  `.gitignore`, `.editorconfig`, `LICENSE`, `.github/workflows/ci.yml`,
  `.github/dependabot.yml`, and a minimal `src/` + `tests/` proving the pipeline.
- No product logic beyond one placeholder module with one passing unit test.

## Detailed Requirements

1. `package.json`:
   - `"name": "freshshot"` (working name, ADR-007), `"version": "0.0.0"`, `"private": true`
     (removed by issue 28), `"type": "module"`, `"engines": { "node": ">=20" }`,
     `"license": "MIT"`.
   - `bin: { "freshshot": "dist/cli/main.js" }` (target exists after issue 05; that is fine —
     `bin` is only exercised from issue 05 on).
   - Scripts (exact names): `build` (tsup), `typecheck` (`tsc --noEmit`), `test`
     (`vitest run --coverage`), `test:watch`, `lint` (`biome check .`), `format`
     (`biome check --write .`).
   - Runtime dependencies, exhaustive (DESIGN §5): `playwright`, `zod`, `yaml`, `commander`,
     `pixelmatch`, `pngjs`, `fast-glob`, `mdast-util-from-markdown`, `unist-util-visit`,
     `picocolors`. Latest stable versions with caret ranges; `package-lock.json` committed.
   - Dev dependencies: `typescript`, `tsup`, `vitest`, `@vitest/coverage-v8`, `@biomejs/biome`,
     `execa`, `@types/node`, `@types/pngjs`.
2. `tsconfig.json`: `strict: true`, `module: "NodeNext"`, `moduleResolution: "NodeNext"`,
   `target: "ES2022"`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`,
   `outDir: "dist"`, `rootDir: "src"`, include `src`.
3. `tsup.config.ts`: entry `src/cli/main.ts`, format `esm`, `platform: "node"`, `clean: true`,
   banner `#!/usr/bin/env node` for the bin entry, no bundling of `playwright` (mark all runtime
   deps external), `target: "node20"`. Until issue 05 creates the real entry, point tsup at a
   placeholder `src/cli/main.ts` that prints `freshshot 0.0.0` and exits 0.
4. `vitest.config.ts`: `environment: "node"`, coverage provider `v8`, global thresholds
   `lines: 85, functions: 85, branches: 80`; per-file thresholds for
   `src/core/paths.ts`, `src/diff/compare.ts`, `src/scan/**`, `src/config/**` at `lines: 95`
   (configured now via `coverage.thresholds` with glob keys; files may not exist yet — use the
   documented vitest behavior that missing files are ignored, and leave a comment).
   Define two named vitest projects now: `unit` (include `tests/unit/**/*.test.ts`,
   `tests/integration/**/*.test.ts`) and `browser` (include `tests/browser/**/*.test.ts`;
   empty until issue 09).
5. `biome.json`: recommended rules, formatter enabled (2-space indent, 100-col line width),
   organize imports on. Exclude `dist/`, `coverage/`, `.freshshot/`.
6. `.gitignore`: `node_modules/`, `dist/`, `coverage/`, `.freshshot/`, `*.tsbuildinfo`.
7. `LICENSE`: MIT, copyright holder `Saber5656`.
8. `.github/workflows/ci.yml`:
   - Triggers: `push` to `main`, `pull_request`.
   - Job `test` matrix: `ubuntu-latest` × Node `20`, `22`; plus one `macos-latest` × Node `22`
     smoke job (same steps, `continue-on-error: false`).
   - Steps: checkout → setup-node (with `cache: npm`) → `npm ci` → `npm run lint` →
     `npm run typecheck` → `npm run test` → `npm run build`.
   - **All action references pinned to full commit SHAs** with a trailing version comment
     (DESIGN §17.9).
   - A separate job `browser-tests` (same triggers, ubuntu-latest, Node 22) that additionally
     runs `npx playwright install chromium --with-deps` and then
     `npx vitest run --project browser --passWithNoTests`; the `browser` project has no tests
     until issue 09, and `--passWithNoTests` keeps the job green until then (leave a
     `TODO(issue-09): drop --passWithNoTests` comment in the workflow).
9. `.github/dependabot.yml`: weekly updates for `npm` and `github-actions` ecosystems.
10. `src/core/types.ts`: create with a placeholder exported type
    `export type Placeholder = never;` replaced by later issues; plus
    `tests/unit/scaffold.test.ts` asserting `1 + 1 === 2` so vitest has one real test.
11. `README.md`: keep the existing two-line file; append nothing (user docs are issue 26).

## Acceptance Criteria

- [ ] `npm ci` succeeds on a clean checkout with Node 20 and Node 22.
- [ ] `npm run lint`, `npm run typecheck`, `npm run test`, `npm run build` all exit 0 locally.
- [ ] `node dist/cli/main.js` prints `freshshot 0.0.0` and exits 0 after `npm run build`.
- [ ] CI workflow runs green on a PR (both jobs, all matrix cells).
- [ ] Every `uses:` in workflows is a full 40-char commit SHA.
- [ ] `package-lock.json` is committed; runtime deps exactly match the list above (no extras).
- [ ] Dependabot config validates (GitHub UI shows both ecosystems).
- [ ] Coverage report is produced (thresholds pass trivially with the placeholder test).

## Validation

- Run the four npm scripts locally on Node 20; capture output in the PR description.
- Open a draft PR to trigger CI; all jobs green.
- `npm ls --omit=dev --depth=0` output matches the runtime dependency list exactly.

## Dependencies

None (first issue).

## Non-goals

- Any CLI behavior beyond the placeholder version print (issue 05).
- npm publish configuration (`private: true` stays until issue 28).
- Browser installation in the default test job (browser job only).

## Design References

- DESIGN §5 (runtime & dependencies), §18 (testing strategy), §17.9 (supply chain)
- ADR-006 (stack), ADR-007 (working name)
