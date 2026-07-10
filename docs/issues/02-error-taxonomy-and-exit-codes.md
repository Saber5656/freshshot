# Title

Core error taxonomy and exit-code contract

## Summary

Implement `src/core/errors.ts`: the `FreshshotError` class, the closed `ErrorCode` union from
DESIGN §16, the error→exit-code mapping from DESIGN §14.4, and a formatter used by every later
module. This is the single place error semantics live.

## Context

Every module raises typed errors; the CLI maps them to the process exit-code contract
(0 success / 1 policy / 2 usage-config / 3 environment). Implementing this first means later
issues never invent ad-hoc error shapes.

## Scope

- `src/core/errors.ts` and `tests/unit/errors.test.ts` only.

## Detailed Requirements

1. `export type ErrorCode =` exactly these members (DESIGN §16):
   `"USAGE" | "CONFIG_NOT_FOUND" | "CONFIG_INVALID" | "CONFIG_PATH_ESCAPE" |
   "SERVER_START_TIMEOUT" | "SERVER_EXITED_EARLY" | "SERVER_UNREACHABLE" |
   "BROWSER_NOT_INSTALLED" | "NAV_BLOCKED_ORIGIN" | "STEP_FAILED" | "HOOK_FAILED" |
   "CAPTURE_FAILED" | "WRITE_FAILED" | "DOCS_PARSE_FAILED" | "BASELINE_DECODE_FAILED"`.
2. `export class FreshshotError extends Error` with fields
   `readonly code: ErrorCode`, `readonly hint?: string`, `override cause?: unknown`;
   constructor `(code, message, opts?: { hint?: string; cause?: unknown })`;
   `name` set to `"FreshshotError"`.
3. `export function isFreshshotError(e: unknown): e is FreshshotError` (instanceof + duck-type
   fallback on `name`+`code` for cross-realm safety).
4. `export function exitCodeForError(e: unknown): 1 | 2 | 3` — mapping (DESIGN §16 table):
   - 2: `USAGE`, `CONFIG_NOT_FOUND`, `CONFIG_INVALID`, `CONFIG_PATH_ESCAPE`
   - 3: `SERVER_START_TIMEOUT`, `SERVER_EXITED_EARLY`, `SERVER_UNREACHABLE`,
        `BROWSER_NOT_INSTALLED`, `WRITE_FAILED`
   - 1: `NAV_BLOCKED_ORIGIN`, `STEP_FAILED`, `HOOK_FAILED`, `CAPTURE_FAILED`,
        `DOCS_PARSE_FAILED`, `BASELINE_DECODE_FAILED` (these normally surface as per-shot /
        per-file statuses, but if one escapes to top level it is a policy failure)
   - Non-`FreshshotError` values: 3 (unexpected runtime failure).
5. `export function formatError(e: unknown, opts: { verbose: boolean }): string`:
   - For `FreshshotError`: `error[CODE]: message` on line 1; `  hint: …` line when present;
     `cause` summarized (its `message` only) when present; full stack appended only when
     `verbose`.
   - For unknown values: `error[UNEXPECTED]: <String(e)>`, stack when verbose and available.
   - Strips ANSI escape and C0 control characters (except `\n`, `\t`) from all interpolated
     message/hint/cause text (DESIGN §17.7). Provide the shared helper
     `export function sanitizeText(s: string): string` here — later issues reuse it.
6. No other exports. No I/O, no process.exit here.

## Acceptance Criteria

- [ ] The `ErrorCode` union compiles to exactly the 15 listed members (a test asserts the
      runtime list used by the mapping covers all codes — e.g. an exhaustive `switch` with
      `never` check).
- [ ] `exitCodeForError` returns the exact table values for every code; unknown errors → 3.
- [ ] `formatError` output matches the specified shapes (snapshot tests) and never emits ANSI
      codes present in the input.
- [ ] `sanitizeText("[31mred[0m")` → `"red"`.
- [ ] 100 % line coverage of `errors.ts`.

## Validation

- `npm run test` green; review snapshots for the three formatting shapes
  (with hint / with cause / verbose stack).

## Dependencies

- 01 (toolchain).

## Non-goals

- Exit code 0 handling and process termination (CLI, issue 05).
- Reporter presentation (issue 24) — this file only formats single errors.

## Design References

- DESIGN §16 (taxonomy), §14.4 (exit codes), §17.7 (output hygiene)
