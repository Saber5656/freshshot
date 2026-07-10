# Title

SECURITY.md and security hardening audit

## Summary

Publish the project's security policy (SECURITY.md) and perform a verifying audit of every
DESIGN §17 requirement against the implemented code and tests, recording evidence in
`docs/security-audit-v1.md`. Any unmet requirement becomes a new blocking issue — this is the
gate that makes the security model real, not aspirational.

## Context

DESIGN §17 distributed security acceptance criteria across issues 03, 06, 11, 13, 20, 24, 28.
Before release, one pass must confirm nothing was skipped or weakened, and the public repo needs
a disclosure policy. This issue is verification + documentation; it changes no product code
itself.

## Scope

- `SECURITY.md` (repo root), `docs/security-audit-v1.md` (evidence checklist), new
  `docs/issues/NN-*.md` drafts for any gaps found (following ISSUE_PLAN §8 discovery protocol).

## Detailed Requirements

1. `SECURITY.md` contents (English):
   - Supported versions: latest minor only.
   - Reporting: GitHub private vulnerability reporting (link to the repo's advisories page);
     acknowledgement within 7 days, fix-or-mitigation target 90 days; no bounty.
   - Trust model summary for users (from ADR-003 / DESIGN §17.2): config and hooks are code;
     review them like CI workflow changes; `update` in CI should run with least-privilege
     tokens; freshshot makes no network calls beyond the configured app and origins.
   - Enabling GitHub private vulnerability reporting itself is a **human maintainer step** —
     list it as a checklist item for the owner, do not attempt via API.
2. `docs/security-audit-v1.md`: a table with one row per §17 requirement
   (17.3 confinement — outputs/hooks/static/cwd/globs/static-server requests; 17.4 origin
   policy pre-nav + post-redirect; 17.5 no-secrets posture incl. docs warnings; 17.6 parser
   limits (2 MiB, alias cap, PNG containment); 17.7 sanitization in errors/reporter/server
   logs; 17.8 loopback binding + nosniff + no-listing + method limits; 17.9 lockfile, `npm ci`,
   Dependabot, SHA-pinned actions, provenance flag in the release workflow; 17.10 defaults —
   deny-origins, report-only coverage, strict schema, no telemetry). Columns:
   requirement / where implemented (file) / evidence (test file::test name or config line) /
   status (pass | gap).
3. Audit procedure: for each row, locate the test or config line and RUN the relevant test
   selection; paste the vitest selector used. A row without executable evidence is a `gap`.
4. Gaps: for each `gap`, create a `docs/issues/NN-*.md` draft (next free number, standard
   sections) and list it in ISSUE_PLAN §2 + §8; the audit doc links them. The audit is complete
   with gaps **documented**, but v1 release (issue 28's publish gate) requires zero open gaps.
5. Abuse-case spot checks (exploratory, recorded in the audit doc):
   - config with `output: ../../evil.png` (expect `CONFIG_PATH_ESCAPE`);
   - static server request `GET /%2e%2e/%2e%2e/etc/hosts` (expect 404);
   - `goto` to a live non-allow-listed local port (expect `NAV_BLOCKED_ORIGIN`, zero requests);
   - hooks path outside root (expect `CONFIG_PATH_ESCAPE`);
   - crafted shot id with ANSI escape in a failing message (expect sanitized output).
   Each with the exact command/config used and observed result.

## Acceptance Criteria

- [ ] `SECURITY.md` exists with the four content blocks; maintainer-action checklist explicit.
- [ ] `docs/security-audit-v1.md` covers every §17.3–§17.10 requirement with file + executable
      evidence; zero rows left blank.
- [ ] All five abuse-case spot checks recorded with observed results matching expectations.
- [ ] Any `gap` rows have corresponding issue drafts and ISSUE_PLAN updates.
- [ ] No product code changed by this issue (audit-only; fixes belong to the gap issues).

## Validation

- Reviewer re-runs two randomly chosen evidence commands and one abuse case; results match the
  audit doc.

## Dependencies

- 25 (feature-complete product to audit).

## Non-goals

- New hardening features (v2: sub-resource blocking etc.). Penetration testing beyond the
  listed abuse cases. Fixing gaps inside this issue.

## Design References

- DESIGN §17 (entire), ISSUE_PLAN §5 (security coverage mapping), §8 (discovery protocol),
  ADR-003
