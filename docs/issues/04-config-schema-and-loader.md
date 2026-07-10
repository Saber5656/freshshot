# Title

Config schema, loader, and semantic validation

## Summary

Implement `src/config/schema.ts` (zod strict schema for `freshshot.config.yaml`, DESIGN §6.2)
and `src/config/load.ts` (discovery, YAML parsing, semantic validation per §6.3, defaults, and
per-shot effective-settings merge). Output is a fully-resolved, typed `LoadedConfig` every later
module consumes; no other module re-reads or re-validates config.

## Context

The config file is the product's main input and a trust boundary (DESIGN §17.2): strict
validation with precise, hint-bearing errors is both UX and security. All defaults live here so
downstream code never applies its own.

## Scope

- `src/config/schema.ts`, `src/config/load.ts`, extension of `src/core/types.ts` with resolved
  types, `tests/unit/config-schema.test.ts`, `tests/unit/config-load.test.ts`,
  YAML fixtures under `tests/fixtures/configs/`.

## Detailed Requirements

1. **Schema** (`schema.ts`), zod with `.strict()` on every object level, exactly modeling
   DESIGN §6.2 including all defaults and bounds:
   - `version: z.literal(1)`.
   - `server`: object with optional `command`, `url`, `readyPath` (default `"/"`),
     `readyTimeoutMs` (int, 1000–600000, default 60000), `env` (record string→string, default {}),
     `cwd` (default `"."`), `static`, `baseUrl`. Mode exclusivity is a semantic rule (below),
     not a zod union, so error messages can be precise.
   - `docs`: `include` (string[], default `["README.md", "docs/**/*.md"]`), `exclude`
     (string[], default `[]`).
   - `capture` global defaults exactly as §6.2 (viewport 1280×800, deviceScaleFactor `1|2`
     default 1, colorScheme `light|dark` default `light`, reducedMotion `reduce|no-preference`
     default `reduce`, disableAnimations true, hideScrollbars true, waitForFonts true,
     settleMs 0–10000 default 200, seedRandom false, freezeTime nullable ISO string default
     null, stepTimeoutMs 100–120000 default 10000, diff.threshold 0–1 default 0.1,
     diff.maxDiffPixelRatio 0–1 default 0.001).
   - `allowedOrigins`: string[] default `[]`.
   - `hooks`: nullable string default null.
   - `shots`: array min 1 of shot objects: `id` regex `/^[a-z0-9][a-z0-9_-]{0,63}$/`, `output`
     string, optional `description`, `skip` default false, `steps` (array min 1 of the step
     schema below), optional `capture` override (`type` `viewport|element|fullPage` default
     viewport, nullable `selector`, `padding` int 0–200 default 0), `mask` string[] default [],
     `maskColor` default `"#FF00FF"` (regex `/^#[0-9A-Fa-f]{6}$/`), nullable `viewport`,
     nullable `colorScheme`, `diff` partial override default {}.
   - **Step schema**: every step is either a shorthand or a strict object holding exactly one
     step key plus optional `timeoutMs` (int 100–120000). Shorthands (cannot carry
     `timeoutMs`): `goto: "/p"`, `click: "sel"`, `hover: "sel"`, `press: "Enter"`,
     `wait: 500` (int 1–10000), `scroll: "sel"`, `hook: "name"`. Object forms (each a strict
     zod object whose only keys are the step key, its params, and `timeoutMs`):
     `{ goto, waitUntil? }`, `{ click, button?, clickCount? }`, `{ hover }`,
     `{ fill: { selector, value } }`, `{ press, selector? }`,
     `{ select: { selector, value? | label? | index? } }` (exactly one of the three —
     semantic rule k), `{ waitFor: { selector, state? } }`, `{ wait }`, `{ scroll }`,
     `{ hook }`. This gives every step type a `timeoutMs` override path per DESIGN §7.
     Unknown keys anywhere → zod strict error.
2. **Loader** (`load.ts`):
   - `export async function loadConfig(opts: { cwd: string; configPath?: string }): Promise<LoadedConfig>`.
   - Discovery: explicit `configPath` (relative to cwd) or `freshshot.config.yaml` then
     `freshshot.config.yml` in cwd. Missing → `CONFIG_NOT_FOUND` with hint to run
     `freshshot init`.
   - Root = directory containing the config file (absolute).
   - Parse YAML with the `yaml` package, `{ maxAliasCount: 100 }`; parse errors →
     `CONFIG_INVALID` including the YAML error position.
   - zod errors → `CONFIG_INVALID`; message lists up to 5 issues, each as
     `<yamlPath>: <zod message>` where yamlPath uses `shots[2].capture.selector` notation.
   - **Semantic validation** (each violation → error per DESIGN §6.3 with the exact code):
     a. exactly one of `server.command|static|baseUrl`;
     b. `server.url` required iff command mode; forbidden otherwise;
     c. unique `shots[].id`, unique normalized `shots[].output`;
     d. every `output` passes `assertPngOutputPath` (issue 03);
     e. first step of every shot is `goto`;
     f. `capture.type === "element"` ⇒ `selector` non-empty (per-shot, after merge);
     g. docs globs: relative, no `..` segment, no absolute (`CONFIG_INVALID`);
     h. `hooks`, `server.static`, `server.cwd` resolve inside root (via issue 03) — resolved
        absolute paths are stored on the result;
     i. `server.url`, `server.baseUrl`, `allowedOrigins[]` parse with `new URL` and scheme
        `http:`/`https:`;
     j. `freezeTime` non-null ⇒ matches ISO 8601
        (`/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(\.\d{1,3})?(Z|[+-]\d{2}:\d{2})$/`) AND
        `Date.parse` yields a finite timestamp;
     k. `select` steps: exactly one of `value|label|index`.
   - **Merge**: produce `shots: ResolvedShot[]` where each shot carries `effective` settings =
     global `capture` deep-merged with per-shot `viewport`/`colorScheme`/`capture`/`diff`
     overrides (per-shot non-null wins; `diff` merges per-field). Merge logic in one exported
     pure function `mergeShotSettings(global, shot)` for direct unit testing.
   - Result type:
     ```ts
     interface LoadedConfig {
       root: string;            // absolute
       configFile: string;      // absolute
       config: ResolvedConfig;  // defaults applied
       shots: ResolvedShot[];   // id, outputAbs, outputRel, steps, effective, skip, …
     }
     ```
3. Do not read `process.env` or perform any interpolation (DESIGN §17.5).

## Acceptance Criteria

- [ ] Minimal valid config (version, server.static, one shot with one goto) loads with all
      documented defaults applied (assert full resolved object snapshot).
- [ ] Each semantic rule a–k has at least one failing fixture asserting the exact `ErrorCode`
      and that the message contains the YAML path and a hint.
- [ ] Unknown key at top level, in a shot, and in a step each produce `CONFIG_INVALID` naming
      the offending path.
- [ ] `mergeShotSettings` precedence verified: shot.diff.maxDiffPixelRatio overrides global;
      untouched fields keep global values.
- [ ] Both discovery filenames and `--config`-style explicit path (incl. relative) work;
      missing file → `CONFIG_NOT_FOUND`.
- [ ] YAML alias bomb fixture (>100 aliases) is rejected as `CONFIG_INVALID`, not by hanging.
- [ ] Line coverage of `src/config/**` ≥ 95 %.

## Validation

- `npm run test` green; fixture files reviewed against DESIGN §6.2 example line by line.

## Dependencies

- 01, 02, 03.

## Non-goals

- CLI flag parsing (issue 05 passes `configPath` in).
- Runtime behavior of steps/servers (later issues consume the types only).

## Design References

- DESIGN §6 (schema, defaults, semantic rules), §7 (step forms), §17.2/§17.5 (trust, secrets),
  §16 (error codes)
