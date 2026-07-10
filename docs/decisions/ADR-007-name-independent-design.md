# ADR-007: Name-independent design; rename decision gated before publication

- Status: accepted (2026-07-10, confirmed with product owner)

## Context

Verified 2026-07-10: the npm package name `freshshot` is owned by an abandoned project
(mphinance/freshshot v0.1.0, MIT, 0 stars) whose concept overlaps ours. Publishing under a
scoped name (`@saber5656/freshshot`) would cause permanent confusion with the unscoped package.
The product owner chose to keep designing under the working name and decide the final name
before publication.

## Decision

1. `freshshot` is a **working name**. All design docs, code identifiers, binary name, and config
   filename use it consistently so that a rename is a mechanical sweep.
2. Publication to npm is **blocked** by a naming gate (issue 29): candidate generation,
   availability verification (npm, GitHub, common registries), human decision, then a single
   rename sweep (package name, bin, config filename `\<name\>.config.yaml`, docs).
3. Nothing may hardcode the name in ways a sweep cannot reach (e.g., no name-derived magic
   strings in user data; `.freshshot/` directory name derives from the final name at rename time).

## Consequences

- Design and implementation proceed without waiting on branding.
- The rename sweep is one focused, testable change (grep-clean acceptance criterion).
- GitHub repo rename (redirects preserved by GitHub) happens at the same gate.

## Alternatives considered

- Scoped npm name keeping `freshshot` — rejected: permanent collision with an unscoped
  same-concept package; discoverability loss.
- Rename immediately — rejected: naming deserves a considered decision; it must not block design.
