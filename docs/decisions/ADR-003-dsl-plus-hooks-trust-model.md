# ADR-003: Declarative step DSL plus an explicit JS hooks escape hatch

- Status: accepted (2026-07-10, confirmed with product owner)

## Context

Recipes need to drive real apps into specific states. A purely declarative DSL hits an
expressiveness ceiling (drag-drop, app-specific setup, network stubbing). Raw script files have
maximal power but no reviewable structure and a high entry barrier. Inline JS strings inside
YAML (`evaluate: "..."`) hide code execution inside data.

## Decision

1. The primary surface is a small declarative DSL (DESIGN §7): `goto`, `click`, `hover`, `fill`,
   `press`, `select`, `waitFor`, `wait`, `scroll`, `hook`.
2. The escape hatch is **one repo-local ESM hooks module** referenced by path in config and
   invoked by name via the `hook` step (DESIGN §8). No inline JS in YAML, ever.
3. Trust model, stated in docs and SECURITY.md: **config and hooks are code**. Executing
   `freshshot` runs `server.command` and hook functions with the user's privileges — identical
   trust to `package.json` scripts. freshshot never fetches or generates code.

## Consequences

- Code execution is always visible as a file in review (`freshshot.hooks.mjs`), greppable and
  lintable; YAML remains pure data.
- The DSL can stay small: anything exotic goes to a hook instead of growing the schema.
- Security review of a PR touching screenshots = review config + hooks file, same bar as CI
  workflow changes.

## Alternatives considered

- DSL-only (no escape hatch) — rejected: real apps need arbitrary setup; users would fork.
- Inline `evaluate` step — rejected: hides code in data; encourages unreviewable one-liners.
- Sandboxing hooks (vm/isolates) — rejected for v1: false sense of security; hooks legitimately
  need the Playwright Page object, which is already an arbitrary-execution surface.
