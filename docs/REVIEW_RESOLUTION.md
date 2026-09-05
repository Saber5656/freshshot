# Review resolution contract

This addendum is a documentation-only acceptance contract for PR #1. It records the resolution required for each existing review thread. It does not claim that product implementation or tests have been completed. The existing Bot review is the sole Bot input for this PR and will not be retriggered.

## PRRT_kwDOTNkDa86QCxoN — redirect targets are blocked before network access

Finding: Validating only the initial navigation target does not protect against an untrusted redirect hop.

Normative resolution:
- Validate the target before issuing the request/navigation.
- For every redirect hop, inspect and validate the next URL before allowing the browser or HTTP client to contact it.
- Main-frame navigation policy must be evaluated separately from subresource policy, and a rejected target must produce NAV_BLOCKED without a network request to that target.

Focused verification before resolving this thread:
- Use a local redirect chain whose first hop points to a forbidden target and assert the forbidden server receives zero requests.
- Test direct forbidden targets, allowed targets, and multiple redirect hops with a per-hop decision trace.
- Verify a subresource rejection cannot be confused with a main-frame navigation decision.

## PRRT_kwDOTNkDa86QCxoO — HOOK_FAILED remains distinct from STEP_FAILED

Finding: An attribution wrapper must preserve hook failure semantics instead of relabeling every wrapped failure as a step failure.

Normative resolution:
- Preserve HOOK_FAILED as the terminal classification when the hook itself fails.
- Only the documented NAV_BLOCKED exception may be translated by the navigation attribution boundary; unrelated hook failures must pass through unchanged.
- The result must retain the original step/hook identity and failure details without weakening the exit-code or JSON contract.

Focused verification before resolving this thread:
- Trigger a hook failure, a step failure, and NAV_BLOCKED independently and assert three expected classifications.
- Confirm the wrapper does not rewrite HOOK_FAILED to STEP_FAILED and does not rewrite unrelated errors to NAV_BLOCKED.
- Check human and JSON output expose the same classification.

## PRRT_kwDOTNkDa86QCxoP — scanner errors fail a requested coverage gate

Finding: A coverage gate that fails only for broken items can pass while the scanner itself errored and produced incomplete coverage.

Normative resolution:
- When fail-on coverage is requested, any scanner error is a gate failure, regardless of whether the partial result contains broken items.
- The report must distinguish scanner error, broken coverage, and successful complete scan.
- Without the fail-on option, scanner errors may be reported according to the CLI contract but must not be silently presented as complete coverage.

Focused verification before resolving this thread:
- Inject scanner errors with zero, partial, and apparently clean findings and assert the requested gate fails in all cases.
- Assert broken items and scanner errors have distinct report/exit semantics.
- Confirm the success path requires a complete scan with no scanner error.

## PRRT_kwDOTNkDa86QCxoQ — hook extensions are validated by the config loader

Finding: Validating extensions only inside loadHooks leaves invalid configuration accepted until a later execution phase.

Normative resolution:
- The configuration loader must validate hook paths/extensions as part of schema/config loading and accept only the documented .mjs or .js forms.
- Unsupported extensions, missing paths, and malformed hook entries must fail before execution starts.
- loadHooks may repeat defense-in-depth checks but is not the first or sole validation boundary.

Focused verification before resolving this thread:
- Load valid .mjs and .js configurations and assert they pass schema validation.
- Load unsupported extensions, missing files, and malformed entries and assert failure occurs during config loading with actionable diagnostics.
- Verify execution is not started after configuration validation fails.

## PRRT_kwDOTNkDa86QCxoR — release rename sweep covers all release surfaces

Finding: A name-independent design must include release metadata and tooling, not only source references.

Normative resolution:
- The rename checklist must sweep SECURITY.md, CHANGELOG, RELEASING, workflow files, scripts, root .gitignore, package metadata, documentation, examples, and generated release configuration.
- Search must cover tracked files and relevant history/automation inputs as defined by the release policy.
- Any stale product name, package name, executable, URL, or secret name must be classified as fixed, intentionally retained, or a release blocker; no silent substitution is allowed.

Focused verification before resolving this thread:
- Run the release rename inventory over the complete tracked-file set and the explicitly listed release surfaces.
- Add a fixture containing each surface and assert the inventory finds every stale reference.
- Confirm the final report records intentional legacy references separately from unresolved release blockers.

## Scope and review boundary

This file is a design/acceptance contract only. It is not evidence that the implementation or focused checks have already passed. After the relevant implementation and validation evidence exists, each mapped existing thread may be replied to and resolved individually. No Bot review will be triggered again.