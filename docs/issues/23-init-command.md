# Title

`init` command

## Summary

Wire `src/cli/commands/init.ts` (replacing the stub): scaffold a commented starter
`freshshot.config.yaml`, ensure `.freshshot/` is gitignored, refuse to clobber an existing
config without `--force`, and print actionable next steps (DESIGN §3.1, §14.1, §15).

## Context

`init` is the adoption entry point (`npx freshshot init`). It must be safe (never destroy an
existing config silently), idempotent for `.gitignore`, and produce a config that passes the
issue-04 loader after the user fills in one shot.

## Scope

- `src/cli/commands/init.ts`, the starter template as a TypeScript string constant in
  `src/cli/templates.ts`, `tests/integration/init-command.test.ts` (built CLI, temp dirs).

## Detailed Requirements

1. Behavior in cwd (no `--config` interplay: init always writes `./freshshot.config.yaml`):
   a. Config exists and no `--force` → `FreshshotError("USAGE",
      "freshshot.config.yaml already exists", { hint: "use --force to overwrite" })`, exit 2,
      file untouched. (Also detects the `.yml` variant and refuses, naming it.)
   b. Write the starter template (below). With `--force`, overwrite.
   c. `.gitignore` handling: create with a `.freshshot/` line if missing; append the line if
      present without it (preserving a trailing newline exactly once); do nothing if the line
      exists (exact-line match, also matching `.freshshot` without slash → treated as present).
   d. Print next steps to stderr: edit server mode, add a first shot, run `freshshot update`,
      install Chromium (`npx playwright install chromium`).
2. Starter template requirements:
   - The schema requires ≥1 shot, so the template ships one minimal live shot
     (`id: home`, `output: docs/images/home.png`, `steps: [goto: /]`) plus a fully commented-out
     richer example shot, and `server: { static: "." }` as a placeholder with a loud `# TODO`
     comment telling the user to replace the server mode. The generated file must pass
     `loadConfig` as-is, so `freshshot update` right after `init` fails only at runtime reality
     (missing page), never at validation.
   - Every top-level option from DESIGN §6.2 appears either live or as an explanatory comment
     with its default (so the config file doubles as reference documentation).
   - Template is asserted-in-tests against the loader (parse + validate).
3. Idempotence: running `init` twice without `--force` leaves both files byte-identical after
   the first run (config refusal + gitignore no-op).
4. No network, no browser, no server. Exit 0 on success.

## Acceptance Criteria

- [ ] Fresh temp dir: `init` exits 0; config file exists; `loadConfig` accepts it; `.gitignore`
      created with exactly one `.freshshot/` line.
- [ ] Existing `.gitignore` without trailing newline gets the line appended correctly (golden
      bytes); with the line already present (either form) → byte-identical file.
- [ ] Existing config: `init` exits 2, config byte-identical; `init --force` overwrites.
- [ ] `.yml`-variant config present → refusal message names `freshshot.config.yml`.
- [ ] Next-steps text includes the four documented actions.
- [ ] Template snapshot test: template mentions every §6.2 top-level key at least once
      (automated by extracting commented keys and comparing to the zod schema's key list —
      keeps template and schema in sync when the schema grows).

## Validation

- Built-CLI integration tests in CI.

## Dependencies

- 05 (skeleton), 04 (loader used in tests; template validated against it).

## Non-goals

- Auto-detecting the project's dev server or docs images (v2 idea: `init --scan`).
- Writing hooks templates.

## Design References

- DESIGN §3.1 (adopt workflow), §14.1 (init contract), §15 (gitignore), §6.2 (template keys)
