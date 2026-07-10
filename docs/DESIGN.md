# freshshot — Design Document (v1)

Status: canonical. This file and `docs/ISSUE_PLAN.md` + `docs/issues/*.md` are the source of truth
for what v1 is. GitHub Issues and PRs are derived artifacts.

Product name note: the npm name `freshshot` is taken by an unrelated project. The design is
**name-independent**; the identifier `freshshot` below is a working name to be swept at
publication time (ADR-007, issue 29).

---

## 1. Overview

**freshshot keeps the UI screenshots embedded in your documentation fresh.**

UI screenshots in READMEs and docs rot: the UI changes, the images quietly lie. freshshot lets a
project define, per image, an executable capture recipe (start the app, drive the browser to a
state, capture a region). One command re-captures everything deterministically and rewrites only
the images whose content **meaningfully** changed. A docs scanner reports which images in the
docs are managed by a recipe, which are unmanaged, orphaned, or broken — making screenshot
freshness a CI-enforceable property instead of a manual chore.

Elevator flow:

```
freshshot.config.yaml ──▶ boot app (command | static | baseUrl)
                      ──▶ per shot: fresh browser context → steps DSL → deterministic capture
                      ──▶ perceptual diff vs committed PNG → write only real changes
docs/**/*.md ─────────▶ scan image refs → coverage report (managed/unmanaged/orphan/broken)
```

Differentiation vs existing tools (see `docs/research/competitive-landscape.md`): freshshot is
**docs-aware**. Competing tools produce screenshots; none of them model the relationship between
docs image references and capture recipes, and none manage a real dev-server lifecycle.

## 2. Goals and non-goals

### 2.1 v1 goals

- G1. Declarative, deterministic re-capture of PNG screenshots referenced by Markdown docs.
- G2. Perceptual-diff-gated writes: zero binary noise in git history.
- G3. Docs coverage: scan Markdown, classify every image reference, enforce policy in CI.
- G4. Real-app support: managed dev-server command with readiness probing (plus static dir and
  external baseUrl modes).
- G5. Single-binary UX: `init`, `update`, `check`, `coverage`, `list` via one CLI; human and
  `--json` output.
- G6. Safe-by-default: path confinement, origin allow-list, strict config validation, no secret
  handling at all in v1.
- G7. Quality suitable for public OSS release (docs, SECURITY.md, packaging, CI).

### 2.2 v1 non-goals (explicitly out)

- Browsers other than Chromium; native/Electron apps.
- Authentication / logged-in screens, secret interpolation of any kind.
- Packaged GitHub Action, auto-PR creation (a CI **recipe** is documented instead).
- MDX/AsciiDoc/reST parsing; only Markdown (`.md`) in v1.
- Theme/locale capture matrices (one shot = one image).
- Watch mode, parallel capture workers, retry/auto-stabilization loops.
- Image formats other than PNG for managed outputs.
- Stable programmatic (library) API; the CLI is the only supported surface.
- Image optimization (oxipng/pngquant).

### 2.3 Design decisions locked with the user (2026-07-10)

| # | Question | Decision |
|---|---|---|
| Q1 | Concept | Docs screenshot freshness tool, as stated in §1 |
| Q2 | Target | General-purpose OSS tool |
| Q3 | Naming | Name-independent design; rename decision gated before publication (ADR-007) |
| Q4 | Scope | Web apps only, Chromium only (ADR-004) |
| Q5 | Recipe definition | Central YAML config + docs scanning/coverage as core differentiator (ADR-002) |
| Q6 | Step expressiveness | Declarative DSL + explicit JS hooks escape hatch; "config = code" trust model (ADR-003) |
| Q7 | Auth screens | Not in v1 |
| Q8 | GitHub Action | Not in v1; CI recipe documented |
| Q9 | Stack | TypeScript + Node.js ≥ 20 + Playwright, npm distribution (ADR-006) |
| Q10 | OSS posture | Public OSS from the start; license/SECURITY/packaging inside v1 |

## 3. Core workflows

### 3.1 Adopt (developer, local)

1. `npx freshshot init` → writes starter `freshshot.config.yaml`, adds `.freshshot/` to `.gitignore`.
2. Developer fills in `server` mode and first `shots` entry.
3. `freshshot update` → boots app, captures, writes `docs/images/home.png`.
4. Developer embeds `![Home](docs/images/home.png)` in README; commits image + config.

### 3.2 Refresh after UI change (developer, local)

1. `freshshot update` → all shots re-captured; only genuinely changed PNGs are rewritten.
2. `git diff --stat` shows exactly the images that changed; commit.

### 3.3 Guard freshness (CI)

1. CI job runs `freshshot check --json`.
2. Exit 1 with a stale-shot list when any committed image no longer matches the live UI
   beyond the threshold. Diff artifacts in `.freshshot/diffs/` are uploaded for review.

### 3.4 Guard coverage (CI)

1. CI job runs `freshshot coverage --fail-on broken,unmanaged`.
2. Exit 1 when docs reference missing images or images no recipe manages.

## 4. Architecture

### 4.1 Module map

All paths below are the required implementation layout.

```
src/
  cli/
    main.ts            # entry: commander program, global flags, error → exit-code mapping
    commands/
      init.ts
      update.ts
      check.ts
      coverage.ts
      list.ts
    reporter.ts        # human (stderr, TTY-aware) + JSON (stdout) reporting
  config/
    schema.ts          # zod schema (strict) + TS types (single source of config types)
    load.ts            # discovery, YAML parse, defaults, semantic validation
  core/
    errors.ts          # FreshshotError taxonomy + exit-code mapping
    paths.ts           # root confinement: resolveInsideRoot(), safe output paths
    runner.ts          # orchestrates update/check runs across shots
    types.ts           # shared result/status types (ShotResult, RunSummary, …)
  server/
    types.ts           # AppServer interface { baseUrl, stop() }
    static.ts          # 127.0.0.1 static file server
    command.ts         # spawn managed command, readiness probe, teardown
    resolve.ts         # pick mode from config, probe external baseUrl
  capture/
    browser.ts         # Chromium launch/close, context factory (viewport, DSF, colorScheme, …)
    determinism.ts     # init scripts, CSS injection, fonts wait, settle, optional clock
    steps.ts           # DSL interpreter (one function per step type)
    hooks.ts           # hooks module loading + invocation
    screenshot.ts      # capture primitives (viewport/element/fullPage, masks)
    shot.ts            # captureShot(): context → steps → screenshot bytes
  diff/
    compare.ts         # PNG decode, pixelmatch, ratio, dimension-mismatch policy
    write.ts           # atomic gated writes, diff artifacts
  scan/
    markdown.ts        # extract image refs (md syntax + <img>) with file:line
    coverage.ts        # classification engine (managed/unmanaged/orphan/broken)
  index.ts             # internal wiring only; NOT a public API
```

### 4.2 Data flow (update)

```
load config ─▶ validate paths ─▶ start server ─▶ launch browser
  ─▶ for each selected shot (sequential):
        new context ─▶ determinism setup ─▶ run steps ─▶ capture PNG bytes
        ─▶ compare with committed output ─▶ status: new|updated|unchanged|forced|failed
        ─▶ gated atomic write (+ diff artifact when changed)
  ─▶ aggregate RunSummary ─▶ report (human/JSON) ─▶ stop browser/server ─▶ exit code
```

`check` is identical except: no output writes ever; changed shots become `stale`; missing
baseline becomes `missing-baseline`.

### 4.3 Concurrency model

v1 is strictly sequential (one browser, one context at a time). Rationale: determinism and
simple failure attribution outweigh speed at docs scale (tens of shots). Parallel workers are v2.

## 5. Runtime and dependencies

- Node.js ≥ 20, ESM-only package, TypeScript strict mode.
- Runtime deps (exhaustive; adding one requires an ADR):
  - `playwright` (Chromium driver; browsers installed by the user via `npx playwright install chromium`)
  - `zod` (config validation)
  - `yaml` (config parsing)
  - `commander` (CLI)
  - `pixelmatch` + `pngjs` (perceptual diff)
  - `fast-glob` (docs file matching)
  - `mdast-util-from-markdown` + `unist-util-visit` (Markdown image extraction)
  - `picocolors` (TTY color, tiny)
- Dev deps: `typescript`, `tsup` (build), `vitest` (+coverage), `@biomejs/biome` (lint+format),
  `execa` (e2e process runs), `@types/node`.
- Browser availability is checked at runtime; missing Chromium yields `BROWSER_NOT_INSTALLED`
  with the exact install command (exit 3).

## 6. Configuration (`freshshot.config.yaml`)

### 6.1 Discovery and root

- Default path: `./freshshot.config.yaml` (fallback `./freshshot.config.yml`) relative to cwd;
  `--config <path>` overrides.
- **Project root = the directory containing the config file.** Every relative path in the config
  resolves from the root. All resolved paths MUST stay inside the root (§17.3).

### 6.2 Full schema (v1)

Validation is zod `.strict()` at every level: unknown keys are hard errors naming the exact YAML
path. `version` is the literal number `1`.

```yaml
version: 1                          # required

server:                             # required; exactly ONE of command|static|baseUrl
  # -- mode A: managed command
  command: "npm run dev"            # spawned with shell; trusted (§17.2)
  url: "http://localhost:5173"      # required with command; base URL for navigation
  readyPath: "/"                    # probe path appended to url origin; default "/"
  readyTimeoutMs: 60000             # default 60000; integer 1000..600000
  env: { PORT: "5173" }             # extra env (string→string); NO secret interpolation
  cwd: "."                          # working dir for command, default root
  # -- mode B: static directory
  static: "./site"                  # served on http://127.0.0.1:<ephemeral>
  # -- mode C: external
  baseUrl: "http://localhost:3000"  # already-running app; probed once before run

docs:
  include: ["README.md", "docs/**/*.md"]   # default; relative globs only
  exclude: []                              # e.g. ["docs/archive/**"]

capture:                            # global defaults; per-shot overridable (§6.3)
  viewport: { width: 1280, height: 800 }   # default
  deviceScaleFactor: 1              # 1 | 2; default 1
  colorScheme: light                # light | dark; default light
  reducedMotion: reduce             # reduce | no-preference; default reduce
  disableAnimations: true           # CSS freeze + screenshot animations:disabled; default true
  hideScrollbars: true              # default true (cross-OS noise)
  waitForFonts: true                # await document.fonts.ready; default true
  settleMs: 200                     # 0..10000; default 200
  seedRandom: false                 # replace Math.random with seeded PRNG; default false
  freezeTime: null                  # ISO 8601 string or null; uses Playwright clock
  stepTimeoutMs: 10000              # default per-step timeout; 100..120000
  diff:
    threshold: 0.1                  # pixelmatch per-pixel sensitivity 0..1; default 0.1
    maxDiffPixelRatio: 0.001        # rewrite gate; 0..1; default 0.001 (0.1 %)

allowedOrigins: []                  # extra origins for goto/redirect targets (§17.4)

hooks: null                         # optional path to ONE repo-local ESM module (§8)

shots:                              # required; ≥1 entries
  - id: home                        # required; unique; /^[a-z0-9][a-z0-9_-]{0,63}$/
    output: docs/images/home.png    # required; unique; .png; must resolve inside root
    description: "Landing page"     # optional
    skip: false                     # optional; reported as skipped
    steps:                          # required; ≥1; FIRST step MUST be goto
      - goto: /
    capture:                        # per-shot override, deep-merged over global capture
      type: viewport                # viewport | element | fullPage; default viewport
      selector: null                # required iff type: element
      padding: 0                    # px around element; 0..200; element only
    mask: []                        # selectors overlaid before capture
    maskColor: "#FF00FF"            # default
    viewport: null                  # per-shot viewport override
    colorScheme: null               # per-shot override
    diff: {}                        # per-shot override of capture.diff
```

### 6.3 Semantic validation rules (beyond types)

| Rule | Error code |
|---|---|
| Exactly one of `server.command` / `server.static` / `server.baseUrl` | `CONFIG_INVALID` |
| `server.url` present iff `command` mode | `CONFIG_INVALID` |
| `shots[].id` unique; `shots[].output` unique | `CONFIG_INVALID` |
| `shots[].output` ends with `.png` and resolves inside root | `CONFIG_PATH_ESCAPE` / `CONFIG_INVALID` |
| First step of every shot is `goto` | `CONFIG_INVALID` |
| `capture.type: element` requires `capture.selector` | `CONFIG_INVALID` |
| `docs.include`/`exclude` globs are relative and contain no `..` | `CONFIG_INVALID` |
| `hooks` / `server.static` / `server.cwd` resolve inside root | `CONFIG_PATH_ESCAPE` |
| `server.url` / `server.baseUrl` / `allowedOrigins[]` are `http:`/`https:` URLs | `CONFIG_INVALID` |
| `freezeTime` parses as ISO 8601 | `CONFIG_INVALID` |

Every config error message includes: YAML path (`shots[2].capture.selector`), what is wrong, and
one actionable hint.

## 7. Step DSL

Steps run strictly in order inside one page. A step failure fails the shot (no retries in v1)
with `STEP_FAILED` naming shot id, step index (0-based), step type, and the Playwright error.
Selectors are Playwright selector strings. Every step accepts optional `timeoutMs` (overrides
`capture.stepTimeoutMs`).

| Step | Forms | Semantics (Playwright mapping) |
|---|---|---|
| `goto` | `goto: "/path"` or `{ goto: "/path", waitUntil: load\|domcontentloaded\|networkidle }` | `page.goto(resolve(baseUrl, path), { waitUntil })`, default `load`. Origin policy §17.4 applies to the request URL **and** the post-redirect final URL |
| `click` | `click: "sel"` or `{ click: "sel", button: left\|right\|middle, clickCount: 1..3 }` | `page.locator(sel).click(...)` |
| `hover` | `hover: "sel"` | `page.locator(sel).hover()` |
| `fill` | `{ fill: { selector, value } }` | `page.locator(selector).fill(value)`; `value` is a literal string — no interpolation (§17.5) |
| `press` | `press: "Enter"` or `{ press: "Control+K", selector: "sel" }` | with selector: `locator.press`; else `page.keyboard.press` |
| `select` | `{ select: { selector, value?\|label?\|index? } }` (exactly one of the three) | `locator.selectOption(...)` |
| `waitFor` | `{ waitFor: { selector, state: visible\|hidden\|attached\|detached } }` default `visible` | `locator.waitFor({ state })` |
| `wait` | `wait: 500` (ms, 1..10000) | `page.waitForTimeout(ms)`; reporter emits a lint warning: prefer `waitFor` |
| `scroll` | `scroll: "sel"` | `locator.scrollIntoViewIfNeeded()` |
| `hook` | `hook: "name"` | invokes exported `name` from the hooks module (§8); error iff no hooks module configured or export missing |

Unknown step keys are `CONFIG_INVALID` at load time (not at run time).

## 8. Hooks (escape hatch)

- One optional ESM module per project: `hooks: ./freshshot.hooks.mjs` (`.mjs` or `.js` only in
  v1; must resolve inside root).
- Loaded once per run via `import(pathToFileURL(abs).href)`.
- Every export used by a `hook` step must be `async function (ctx): Promise<void>` with
  `ctx = { page /* Playwright Page */, baseUrl: string, shotId: string, log(msg: string): void }`.
- Per-invocation timeout: `capture.stepTimeoutMs` unless the step sets `timeoutMs`. Timeout or
  throw → `HOOK_FAILED` naming the hook and shot.
- Trust model: the hooks file is **arbitrary code executed with the user's privileges**, exactly
  like `package.json` scripts. freshshot never downloads or generates hook code (§17.2).

## 9. Server modes and lifecycle

### 9.1 Common interface

```ts
interface AppServer { baseUrl: string; stop(): Promise<void>; }
```

`resolve.ts` picks the mode and returns a started `AppServer`. `stop()` is always called
(finally-block) even when the run fails.

### 9.2 State machine (command mode)

```
IDLE → STARTING → READY → STOPPING → STOPPED
         │  │
         │  └─ (process exits before ready) → FAILED_EARLY_EXIT   [SERVER_EXITED_EARLY, exit 3]
         └──── (readyTimeoutMs elapsed)     → FAILED_TIMEOUT      [SERVER_START_TIMEOUT, exit 3]
```

- Spawn: `spawn(command, { shell: true, cwd, env: {...process.env, ...cfg.env}, detached: posix })`.
- Readiness: `GET url.origin + readyPath` every 250 ms; ready on HTTP status 200–399.
- Failure diagnostics: both failure errors carry the last 40 lines of merged stdout/stderr,
  control characters stripped (§17.7).
- Teardown: POSIX `kill(-pid, SIGTERM)`, then `SIGKILL` after 5 s grace; Windows
  `taskkill /pid <pid> /T /F`. `stop()` resolves only when the process exited.

### 9.3 Static mode

- `node:http` server bound to `127.0.0.1`, ephemeral port; `baseUrl = http://127.0.0.1:<port>`.
- GET/HEAD only (405 otherwise). Path resolution is traversal-safe (§17.3): decode, normalize,
  verify the resolved path is inside the static root; otherwise 404 (never reveal paths).
- Directory request → `index.html` if present else 404. No directory listing. Correct
  `Content-Type` for common web types; `X-Content-Type-Options: nosniff`; `Cache-Control: no-store`.

### 9.4 External mode

- `baseUrl` probed once (`GET`, 5 s timeout, any HTTP response counts as reachable).
  Unreachable → `SERVER_UNREACHABLE` (exit 3) with a hint to start the app first.

## 10. Determinism

Applied per shot, before steps run, in this order:

1. Context creation with `viewport`, `deviceScaleFactor`, `colorScheme`, `reducedMotion`,
   `timezoneId: "UTC"`, `locale: "en-US"` (fixed in v1).
2. `freezeTime` set → Playwright `clock.install({ time })` on the context.
3. `seedRandom: true` → `addInitScript` replacing `Math.random` with mulberry32(seed=1).
4. After the final step, before capture:
   - `waitForFonts` → `page.evaluate(() => document.fonts.ready)`.
   - `disableAnimations` → inject CSS: `*,*::before,*::after{animation:none!important;transition:none!important;caret-color:transparent!important;scroll-behavior:auto!important}`.
   - `hideScrollbars` → inject CSS hiding scrollbars (`::-webkit-scrollbar{display:none}` +
     `html{scrollbar-width:none}`).
   - sleep `settleMs`.

The screenshot call itself additionally passes `animations: "disabled"`, `caret: "hide"`,
`scale: "device"`.

**Determinism acceptance invariant (used across tests):** two consecutive `update` runs against
an unchanged app MUST report every shot `unchanged` on the second run, in the same environment.

## 11. Capture

- One fresh browser context per shot (no state leaks), closed in a finally block.
- `type: viewport` → `page.screenshot({ fullPage: false })`.
- `type: fullPage` → `page.screenshot({ fullPage: true })`.
- `type: element` → bounding box of `selector` (error `CAPTURE_FAILED` if not visible), expanded
  by `padding` px, clamped to the page, captured via `page.screenshot({ clip })`.
- `mask` selectors → Playwright native `mask: [locators]` + `maskColor`.
- Output: PNG bytes in memory; persistence decided by the diff gate (§12).

## 12. Diff and update policy

### 12.1 Comparison

- Baseline = the committed file at `shot.output`. Candidate = freshly captured bytes.
- Decode both with `pngjs`. Baseline exists but fails to decode → treat as `new` (rewrite) and
  emit warning `BASELINE_DECODE_FAILED` (does not fail the run).
- Dimension mismatch (width/height differ) → changed by definition; `changedRatio = 1`.
- Same dimensions → `pixelmatch(threshold)` on RGBA; `changedRatio = diffPixels / (w*h)`.

### 12.2 Statuses and actions

| Command | Condition | Status | Action |
|---|---|---|---|
| update | no baseline | `new` | write |
| update | ratio > maxDiffPixelRatio | `updated` | write + diff artifact |
| update | ratio ≤ maxDiffPixelRatio | `unchanged` | discard candidate |
| update | `--force` | `forced` | write regardless |
| update/check | `skip: true` | `skipped` | nothing |
| update/check | any step/capture error | `failed` | nothing; run continues |
| check | no baseline | `missing-baseline` | nothing (counts as failure) |
| check | ratio > gate | `stale` | diff artifact only |
| check | ratio ≤ gate | `fresh` | nothing |

- Writes are atomic: temp file in the same directory (`<name>.png.tmp-<pid>`) then `rename`.
- Diff artifacts: pixelmatch visual diff written to `.freshshot/diffs/<id>.png` for
  `updated`/`stale`; directory recreated per run.
- Exit codes: any `failed`, `stale`, or `missing-baseline` → exit 1; see §14.4.

## 13. Docs scanning and coverage

### 13.1 Reference extraction (`scan/markdown.ts`)

- Input: files matched by `docs.include` minus `docs.exclude` (fast-glob, `dot: false`,
  followSymbolicLinks: false).
- Parse with `mdast-util-from-markdown`. Collect:
  - `image` nodes (`![alt](path)`)
  - `imageReference` + `definition` pairs
  - `html` nodes: every `<img ... src="…">` via a conservative regex (`/<img\b[^>]*?\bsrc\s*=\s*["']([^"']+)["']/gi`)
- Each hit → `{ docFile, line, column, rawRef }`.
- Skip refs that are absolute URLs (`http:`, `https:`, `//`), `data:`, or anchors.
- Resolve: refs starting with `/` resolve from project root; others relative to the doc's
  directory. Strip query/fragment. Refs resolving outside root are reported as `broken`
  (reason `outside-root`), never followed.
- Only image extensions are tracked: `.png .jpg .jpeg .gif .webp .svg .avif`.

### 13.2 Classification (`scan/coverage.ts`)

Inputs: `refs[]`, `shots[]` (output paths), filesystem existence. All path comparisons use
root-relative normalized paths.

| Class | Definition |
|---|---|
| `managed` | ref path == some `shot.output` |
| `unmanaged` | ref path != every shot output AND file exists |
| `broken` | ref path != every shot output AND file missing (or resolves outside root) |
| `orphan` (shot-level) | `shot.output` not referenced by any scanned doc |

Notes: a `managed` ref whose file does not exist yet is still `managed` (it will be created by
`update`; `check` reports it `missing-baseline`). Non-PNG refs can never be `managed` (v1),
so they classify as `unmanaged`/`broken` like any other ref.

### 13.3 Policy

`coverage --fail-on <list>` where list ⊆ `unmanaged,broken,orphan` (comma-separated). Default:
report-only (exit 0). Selected categories non-empty → exit 1.

## 14. CLI specification

### 14.1 Commands

| Command | Purpose | Key flags |
|---|---|---|
| `freshshot init` | Scaffold `freshshot.config.yaml` + ensure `.freshshot/` in `.gitignore` | `--force` (overwrite config) |
| `freshshot update` | Capture and gated-write all (or selected) shots | `--shot <id>` (repeatable), `--force`, `--json` |
| `freshshot check` | CI freshness gate; never writes outputs | `--shot <id>` (repeatable), `--json` |
| `freshshot coverage` | Docs scan + classification report | `--fail-on <cats>`, `--json` |
| `freshshot list` | Table of shots: id, output, description, docs refs count | `--json` |

Global flags: `--config <path>`, `--verbose`, `--quiet`, `--no-color`, `--version`, `--help`.
Unknown `--shot` id → `USAGE` error (exit 2) listing valid ids.

### 14.2 Output channels

- Human-readable progress/report → **stderr** (TTY-aware colors via picocolors, disabled by
  `--no-color` or `NO_COLOR` or non-TTY).
- `--json` → exactly one JSON document on **stdout**, nothing else on stdout.

### 14.3 JSON schema (stable, versioned)

```jsonc
{
  "schemaVersion": 1,
  "command": "update",                   // update | check | coverage | list
  "root": "/abs/project",
  "startedAt": "2026-07-10T12:00:00Z",   // ISO 8601 UTC
  "durationMs": 12345,
  "shots": [                              // update/check/list
    { "id": "home", "output": "docs/images/home.png",
      "status": "updated",               // §12.2 status set
      "changedRatio": 0.0231,            // null when not compared
      "reason": null,                    // e.g. "dimension-mismatch", error summary for failed
      "durationMs": 830 }
  ],
  "coverage": {                           // coverage command only
    "managed": [ { "ref": "docs/images/home.png", "docFile": "README.md", "line": 12 } ],
    "unmanaged": [ ... ], "broken": [ ... ],
    "orphans": [ { "shotId": "old", "output": "docs/images/old.png" } ]
  },
  "summary": { "total": 5, "updated": 1, "unchanged": 3, "failed": 1 },
  "exitCode": 1
}
```

Additive changes only within `schemaVersion: 1`.

### 14.4 Exit codes (process-wide contract)

| Code | Meaning | Examples |
|---|---|---|
| 0 | Success / nothing to report | all fresh; coverage clean |
| 1 | Policy failure | stale shots, failed shots, missing baseline, coverage `--fail-on` hit |
| 2 | Usage or config error | bad flag, unknown shot id, invalid config, path escape |
| 3 | Environment/runtime error | server start timeout, unreachable baseUrl, Chromium missing |

Precedence when multiple apply: 3 > 2 > 1.

## 15. Storage layout and git conventions

```
<root>/
  freshshot.config.yaml
  freshshot.hooks.mjs          # optional, user-authored
  docs/images/*.png            # committed outputs (paths are user-chosen)
  .freshshot/                  # gitignored, safe to delete anytime
    diffs/<shotId>.png         # visual diffs for updated/stale shots
    tmp/                       # scratch during capture
```

`init` appends `.freshshot/` to `.gitignore` (creating the file if needed, no duplicate lines).

## 16. Error taxonomy (`core/errors.ts`)

`class FreshshotError extends Error { code: ErrorCode; hint?: string; cause?: unknown }`

| Code | Exit | Raised by |
|---|---|---|
| `USAGE` | 2 | CLI arg problems |
| `CONFIG_NOT_FOUND` | 2 | load.ts |
| `CONFIG_INVALID` | 2 | schema/semantic validation |
| `CONFIG_PATH_ESCAPE` | 2 | paths.ts confinement |
| `SERVER_START_TIMEOUT` | 3 | server/command.ts |
| `SERVER_EXITED_EARLY` | 3 | server/command.ts |
| `SERVER_UNREACHABLE` | 3 | server/resolve.ts |
| `BROWSER_NOT_INSTALLED` | 3 | capture/browser.ts |
| `NAV_BLOCKED_ORIGIN` | 1 (shot `failed`) | steps.ts goto policy |
| `STEP_FAILED` | 1 (shot `failed`) | steps.ts |
| `HOOK_FAILED` | 1 (shot `failed`) | hooks.ts |
| `CAPTURE_FAILED` | 1 (shot `failed`) | shot.ts |
| `WRITE_FAILED` | 3 | diff/write.ts |
| `DOCS_PARSE_FAILED` | 1 (file-level, scan continues) | scan/markdown.ts |
| `BASELINE_DECODE_FAILED` | warning only | diff/compare.ts |

Per-shot errors never abort the run; they mark the shot `failed` and the run exits 1.

## 17. Security model

### 17.1 Threat model summary

freshshot is a **local/CI developer tool**. It does not listen on external interfaces, has no
telemetry, and handles no credentials in v1. The interesting boundaries are: config-driven file
writes, config-driven process spawn, config-driven browser navigation, Markdown parsing, the
loopback static server, and the npm supply chain.

### 17.2 Trust boundaries

| Asset / boundary | Trust level | Rules |
|---|---|---|
| `freshshot.config.yaml` | Trusted input, strictly validated | zod strict; semantic rules §6.3; treat like `package.json` |
| Hooks module | **Trusted code execution** | explicit opt-in by path; repo-local only; documented as equivalent to npm scripts |
| `server.command` | Trusted code execution | user-authored; spawned with shell; documented |
| Target app content | Untrusted web content | rendered inside Chromium's sandbox; freshshot only injects its own fixed scripts/CSS |
| Docs Markdown | Parsed, never executed/rendered | parser hardening §17.6 |
| CI environment | Reviewed-code context | config/hooks changes must be reviewed like workflow changes (documented in SECURITY.md) |

### 17.3 Filesystem confinement (`core/paths.ts`)

- `resolveInsideRoot(root, p)`: reject absolute paths outside root, reject any resolution whose
  `path.relative(root, resolved)` starts with `..`, and reject **realpath** escapes through
  symlinked ancestors (compare `fs.realpath` of the deepest existing ancestor).
- Applied to: `shots[].output`, `hooks`, `server.static`, `server.cwd`, every static-server
  request path, docs globs results (fast-glob confined to root, symlinks not followed).
- Output writes additionally require the final extension `.png` and create parent directories
  only inside root.

### 17.4 Navigation origin policy

- Allowed origins = { server origin } ∪ `allowedOrigins`.
- `goto` target URL must belong to an allowed origin (relative paths always do).
- After navigation, the final (post-redirect) main-frame URL is re-checked; violation →
  `NAV_BLOCKED_ORIGIN`, shot fails. Sub-resource requests are NOT blocked in v1 (documented).
- Rationale: prevents typo/config-injection from screenshotting arbitrary internal services in
  CI (SSRF-style) by default.

### 17.5 Secrets posture (v1)

- No secret interpolation, no env substitution in config, no auth features. `fill` values are
  committed plaintext by design.
- Documentation states: never type real credentials via `fill`; screenshots of real data are a
  leak vector — use masks and seeded/demo data.

### 17.6 Parser hardening

- Markdown files larger than 2 MiB are skipped with `DOCS_PARSE_FAILED` (docs files this large
  indicate misconfiguration). Parse errors affect only that file; scan continues.
- YAML loaded with the `yaml` package defaults (no custom tags, no anchors bombs: document set
  `maxAliasCount` low (=100)).
- PNG decode failures are contained (§12.1).

### 17.7 Output hygiene

- Any text echoed from child processes or page errors is stripped of ANSI/control characters
  (except `\n`, `\t`) before terminal display to prevent escape-sequence injection.

### 17.8 Static server exposure

- Binds `127.0.0.1` only; ephemeral port; lifetime = one run; traversal-safe; GET/HEAD only;
  `nosniff`; no directory listing (§9.3).

### 17.9 Supply chain and release

- Exhaustive runtime dependency list (§5); lockfile committed; `npm ci` in CI; Dependabot
  (npm + GitHub Actions ecosystems) enabled; GitHub Actions pinned to full commit SHAs;
  npm publish with `--provenance` from a release workflow; secrets configured manually by a
  human (never by an agent).
- `SECURITY.md`: private vulnerability reporting via GitHub, 90-day response policy, supported
  versions = latest minor.

### 17.10 Secure defaults recap

deny-by-default origins; report-only coverage (opt-in fail); no retries hiding real breakage;
no network calls besides the configured app; no telemetry; strict schema (typos fail loudly);
loopback-only server; path confinement everywhere.

## 18. Testing strategy

| Layer | Tooling | Scope |
|---|---|---|
| Unit | vitest | config schema/semantics, paths confinement (incl. symlink cases), diff math, markdown extraction, coverage classification, error mapping, reporter JSON shape |
| Integration (browser) | vitest + real Chromium | determinism setup, steps DSL against a local fixture page, capture types/masks, hooks |
| Integration (server) | vitest | command spawn/ready/teardown with a scripted child; static server behavior incl. traversal attempts |
| E2E | vitest + execa on built CLI | fixture project lifecycle: init → update (new) → update (unchanged) → mutate app → update (updated) → check (stale) → coverage; asserts files, statuses, exit codes, `--json` against golden (normalized) output |
| Determinism invariant | e2e | second consecutive `update` is 100 % `unchanged` (same env) — REQUIRED in CI |

- Fixture app: `tests/fixtures/site/` static HTML/CSS using system font stack, no JS timers.
- CI: GitHub Actions; `ubuntu-latest` (Node 20 + 22) required, `macos-latest` (Node 22) smoke.
  Steps: `npm ci` → biome → typecheck → unit → `npx playwright install chromium --with-deps` →
  integration + e2e.
- Coverage thresholds (vitest): global ≥ 85 % lines; `core/paths.ts`, `diff/compare.ts`,
  `scan/*.ts`, `config/*.ts` ≥ 95 %.

## 19. Packaging and release (v1)

- ESM-only npm package; `bin: { freshshot: "dist/cli/main.js" }`; `files: ["dist", "README.md", "LICENSE"]`;
  `engines: { node: ">=20" }`; no `exports` API surface beyond the bin (private API).
- Build with tsup (single entry, shebang banner, sourcemaps off for publish).
- License: MIT. CHANGELOG.md maintained manually (Keep a Changelog format).
- Release workflow (manual `workflow_dispatch`): build → test → `npm publish --provenance --access public`.
  npm token/trusted-publishing configured manually by the maintainer (agents only document steps).
- Pre-publish gates: naming decision executed (ADR-007, issue 29); `npm pack` smoke test runs
  `freshshot --version` and `init` in a temp dir.

## 20. Deferred to v2 (explicitly)

GitHub Action + auto-PR; auth via Playwright storageState + env-var interpolation with secret
redaction; Firefox/WebKit; per-shot theme/locale matrices; MDX/AsciiDoc/reST; watch mode;
parallel workers; retry/stabilization loops; oxipng optimization; JS/TS config format; path
aliases for static-site image roots; sub-resource origin blocking; programmatic API;
`scroll` beyond scrollIntoView.

## 21. Known unknowns (may spawn new issues during implementation)

1. Playwright `clock.install` interactions with framework schedulers (React concurrent,
   requestAnimationFrame loops) — freezeTime may need a "tick" escape hatch.
2. Readiness probing for apps that redirect `/` to a heavy route — `readyPath` may need
   status-code allow-list configuration.
3. `<img>` extraction inside fenced code blocks must NOT match (mdast html nodes vs code nodes —
   verify; code blocks are `code` nodes so regex never sees them, but inline HTML edge cases exist).
4. Windows process-tree teardown reliability (`taskkill` vs job objects).
5. pixelmatch defaults vs Chromium sub-pixel AA at deviceScaleFactor 2 — thresholds may need tuning.
6. npm package name availability at publication time (ADR-007).

## 22. Document map

| Doc | Purpose |
|---|---|
| `docs/DESIGN.md` | This file — canonical v1 design |
| `docs/ISSUE_PLAN.md` | Execution order, waves, dependencies, coverage mapping |
| `docs/issues/NN-*.md` | One implementable unit each (English, agent-executable) |
| `docs/decisions/ADR-*.md` | Locked architecture decisions |
| `docs/research/*.md` | External research inputs |
