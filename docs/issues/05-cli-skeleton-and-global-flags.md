# Title

CLI skeleton, global flags, and output channels

## Summary

Implement `src/cli/main.ts` with commander: program metadata, the five subcommands registered as
explicit "not implemented" stubs, global flags (`--config`, `--verbose`, `--quiet`,
`--no-color`), the top-level error handler mapping `FreshshotError` to the exit-code contract,
and the output-channel policy (human text → stderr, stdout reserved for `--json` documents).

## Context

Every command issue (18, 19, 22, 23) plugs an action into this skeleton. Centralizing flag
parsing, error handling, and stream discipline here keeps command implementations small and the
exit-code contract (DESIGN §14.4) enforced in exactly one place.

## Scope

- `src/cli/main.ts`, `src/cli/context.ts` (shared command context), a minimal
  `src/cli/reporter.ts` (logger levels + color handling; full reporting is issue 24),
  `tests/unit/cli-skeleton.test.ts`, `tests/integration/cli-exit-codes.test.ts`.

## Detailed Requirements

1. `main.ts`:
   - commander `program` named `freshshot`, description one-liner, version read from
     `package.json` at build time (tsup `define` or JSON import with assertion — pick one and
     leave a comment; `--version` prints `freshshot <semver>`).
   - Global options: `--config <path>`, `--verbose`, `--quiet`, `--no-color`, plus `--json`
     declared **per command** (not global) so `--help` shows it only where meaningful.
   - Subcommands registered: `init`, `update`, `check`, `coverage`, `list` with descriptions
     from DESIGN §14.1. Until their issues land, each action throws
     `new FreshshotError("USAGE", "command '<name>' is not implemented yet")` — tests pin the
     exit code 2 so stubs cannot be mistaken for success.
   - Full per-command flag matrix (parsing only, no behavior yet):
     `init --force`; `update --shot <id>` (repeatable), `--force`, `--json`;
     `check --shot <id>` (repeatable), `--json`; `coverage --fail-on <cats>`, `--json`;
     `list --json`.
   - Top-level runner: `program.parseAsync().catch(...)` never lets an exception escape; on
     error, print via `formatError(e, { verbose })` to stderr and set `process.exitCode =
     exitCodeForError(e)`. **Never call `process.exit()`** (streams must flush); document in a
     comment.
   - Unknown command/flag: commander configured to write its message to stderr and produce exit
     code 2 (`exitOverride` mapped to `USAGE`).
2. `context.ts`: `export interface CliContext { configPath?: string; verbose: boolean; quiet: boolean; color: boolean; json: boolean }`
   built from parsed flags;
   `color: options.color !== false && !process.env.NO_COLOR && process.stderr.isTTY === true`
   (commander maps `--no-color` to `options.color === false`).
3. Minimal `reporter.ts`:
   - `createLogger(ctx)` returning `{ info(msg), warn(msg), error(msg), debug(msg) }`;
     `debug` only when verbose; `info` suppressed when quiet; all write to **stderr**; colors
     via picocolors only when `ctx.color`; all inputs pass through `sanitizeText` (issue 02).
   - Nothing may write to stdout except a future `--json` emitter (issue 24); add a
     `writeJsonDocument(obj)` helper here that `JSON.stringify`s with 2-space indent to stdout —
     command issues use it.
4. Build: ensure tsup entry produces an executable `dist/cli/main.js` with shebang (issue 01
   banner) and `npx .`-style local invocation works.

## Acceptance Criteria

- [ ] `freshshot --version` prints `freshshot <version>` (matches package.json) and exits 0.
- [ ] `freshshot --help` lists all five commands; each command's `--help` shows its specific
      flags (`--shot`, `--force`, `--fail-on`, `--json` as applicable).
- [ ] `freshshot update` (stub) exits 2 with `error[USAGE]: command 'update' is not implemented yet` on stderr, nothing on stdout.
- [ ] `freshshot nonsense` and `freshshot update --bogus` exit 2 with the message on stderr.
- [ ] With `NO_COLOR=1` or `--no-color`, stderr output contains no ANSI escapes.
- [ ] Integration test runs the **built** binary (`node dist/cli/main.js`) via execa and asserts
      exit codes and stream separation for the cases above.

## Validation

- `npm run build && node dist/cli/main.js --help` manually inspected once; automated tests as
  above run in CI.

## Dependencies

- 01, 02, 04 (flag `--config` is threaded to `loadConfig` by later command issues; this issue
  only stores it in `CliContext`).

## Non-goals

- Any real command behavior (issues 18, 19, 22, 23).
- Full human-readable run reports and JSON schema (issue 24).

## Design References

- DESIGN §14.1 (commands and flags), §14.2 (output channels), §14.4 (exit codes), §17.7
  (output hygiene)
