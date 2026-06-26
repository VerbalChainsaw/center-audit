# CENTER-AUDIT Changelog

## 2.5.1

Operability hardening follow-up. Closes gaps in the v2.5.0 release surfaced by post-publish review.

### Schema: `contradiction.kind` field

Adds an optional `kind` field to contradiction entries: `CONTRADICTION` (default, for v2.0.x back-compat) or `SELF_CORRECTION`. Single-evidence entries (K1-style) are now semantically distinct from real contradictions between two or more anchor-stable observations.

### Schema: `repair_revalidation.drift_notes` is now enforced

The v2.5.0 schema description claimed `drift_notes` was "required when `result != INVARIANT_HOLDS`" but the schema did not enforce it. v2.5.1 uses `allOf` / `if` / `then` to actually require `drift_notes` (non-empty) when `result` is `INVARIANT_DRIFTED`, `INVARIANT_REPLACED`, or `CONTRACT_REJECTED`. Empty `drift_notes` on a non-HOLDS result is now a schema violation.

### Schema: `NO_DEFECT_CONFIRMED` silent nothing-burger guard

With `falsifier` now optional in v2.5.0, an audit could be `NO_DEFECT_CONFIRMED` with empty `disproven_concerns`, no `falsifier`, and no contradictions — a clean bill of health from someone who didn't actually look. v2.5.1 uses `allOf` / `if` / `then` to require either a `case_file.falsifier` or at least one entry in `disproven_concerns` when `result == NO_DEFECT_CONFIRMED`.

### Validator: URL false-positive fix

v2.5.0's `RESOURCE_RE` negative lookbehind excluded `[A-Za-z0-9_.-]` but not `/`, so URLs like `https://example.com/references/foo.md` falsely matched as a resource reference. The current bundle did not trigger this (no URLs contain `references/`), but the bug was latent. v2.5.1 excludes `/` too. Trade-off: nested paths like `subdir/references/foo.md` are no longer validated; in an Agent Skills bundle this is acceptable (nested skill-root paths would violate the bundle structure).

### Doctrine: tiered budget decision criteria

v2.5.0 introduced the tiered pre-flight budget but did not specify when to pick which class. v2.5.1 adds a decision rule: use the highest tier whose criteria apply, escalating one tier when in doubt.

### Doctrine: `repair_revalidation` result-value guidance

v2.5.0 introduced four result values but did not say when to use which. v2.5.1 adds guidance: `INVARIANT_HOLDS` is the default; `DRIFTED` is reserved for cases where the contract edge itself shifted (not for mechanical adjustments); `REPLACED` is for the audit having been wrong; `REJECTED` is for a failed handoff.

### Documentation

- `README.md` version badge updated to 2.5.1; new "What's new in v2.5" section documents `repair_revalidation`, tiered budget, validator fix, and schema relaxations.
- `docs/methodology.md` gains a v2.5 section explaining what changed and what deliberately did not.
- `references/compact-mode.md` gains the mutual-exclusion note that v2.5.0 added only to `SKILL.md`.
- `references/multi-perspective-cascade.md` restores the hermes concrete example as an instance of the host-agnostic pattern (v2.5.0 generalized it away).
- `examples/repair-revalidation-drifted.json` adds a worked example showing `INVARIANT_DRIFTED` with non-trivial `drift_notes`.

### CI

- Bumped `actions/checkout` v4 → v5, `actions/setup-python` v5 → v6, `actions/upload-artifact` v4 → v5. Silences Node 20 deprecation warnings.
- New smoke-test step: validates that the validator catches a deliberately missing `.toml` reference in a fixture bundle.
- New backward-compat step: validates five shapes against the v2.5.1 schema (v2.0.x with falsifier, NO_DEFECT_CONFIRMED with disproof, NO_DEFECT_CONFIRMED with nothing — must reject, full v2.5.1 with `INVARIANT_HOLDS`, `INVARIANT_DRIFTED` without `drift_notes` — must reject).

### Compatibility

- v2.0.x and v2.5.0 JSON outputs remain valid against the v2.5.1 schema.
- New constraints (`drift_notes` required when applicable; `NO_DEFECT_CONFIRMED` guard) tighten the schema; outputs that passed v2.5.0 by being incomplete will now fail v2.5.1. This is intentional.

## 2.5.0

Operability hardening. Methodology preserved; the audit/repair separation, evidence-gated expansion, calibrated fusion, and clean-audit principle are unchanged. This release closes operational gaps surfaced during real-world use and makes the doctrine more honest about its own contract.

### Output schema: `repair_revalidation` field (additive, optional)

Adds a top-level `repair_revalidation` object captured by the repair agent after applying the fix. Four result values: `INVARIANT_HOLDS`, `INVARIANT_DRIFTED`, `INVARIANT_REPLACED`, `CONTRACT_REJECTED`. Closes the operational enforcement gap on repair phase independence.

### Output schema: `falsifier` is now optional

`case_file.falsifier` is no longer required. Required for `DEFECT_CONFIRMED` and `INCONCLUSIVE`; optional for `NO_DEFECT_CONFIRMED` (disproof ledger captures what was tested).

### Output schema: `contradiction.evidence_ids` minimum lowered from 2 to 1

Allows K1-style single-evidence self-corrections. **Behavior change to note:** single-evidence contradictions that previously *failed* validation in v2.0.x now *pass* in v2.5.0+. This is intentional — see the v2.5.1 entry for the semantic distinction (`kind: CONTRADICTION` vs `kind: SELF_CORRECTION`).

### Validator: extension-whitelist replaced with path-only match

`RESOURCE_RE` no longer restricts matching to `.md|.py|.json|.yaml|.yml|.sh|.js|.ts`. Any file referenced via `references/`, `scripts/`, `assets/`, or `evals/` path is checked for existence on disk. Catches missing `.toml`, `.proto`, `.sql`, `.prisma`, `.env.example`, `.txt`. **Stricter than v2.0.x** — bundles that previously passed by accident will now fail.

### Doctrine: tiered pre-flight budget by complexity class

The flat 3/5 cap is replaced with: single defect 3/5, layered surface 5/8, distributed/AI 6/10. Selected by surface shape, not user urgency.

### Doctrine: `compact-mode.md` mutual exclusion

`references/compact-mode.md` is loaded **instead of** `SKILL.md`, not in addition to.

### Generalization: multi-perspective dispatch notes

Hermes-specific `<$text>` pitfall generalized to host-agnostic guidance. The hermes instance is preserved in v2.5.1 as a concrete example.

### Compatibility

v2.0.x JSON outputs (with or without `falsifier`, with or without `repair_revalidation`, with 2+ evidence contradictions) remain valid against the v2.5.0 schema. The validator change is the only breaking strictness change.

## 2.0.1

Maintenance release. No methodology changes — only packaging and integration polish.

### Tightened frontmatter

- Trimmed the `description` field from a 13-clause trigger list to a tight one-sentence trigger shape. Easier for agent triggers to match.
- Bumped skill metadata version and JSON Schema `schema_version` to `2.0.1`.

### Schema tightening

- `case_file.pivots` items are now structured objects (`prior_claim`, `disproof_evidence_ids`, `new_claim`, `new_center`, `selection_reason`) instead of free-form strings. The pivot entry becomes machine-checkable and matches the formats.md pivot template.
- Pivot `disproof_evidence_ids` references the same `E#` evidence IDs as the trajectory, so a pivot cannot be invented without backing evidence.

### Added: repair phase independence

- New SKILL.md section "Repair phase independence" makes explicit that the audit produces a repair contract but the repair phase is a separate operation. The repair agent must re-validate the invariant before editing, not treat the contract as truth.

### Compatibility

- v2.0.0 JSON outputs that omit pivot entries remain valid (pivots is still optional content with `maxItems: 2`).
- v2.0.0 consumers reading the schema by `$id` will read `const: "2.0.1"` and may need to refresh their cached schema. Backwards-compatible otherwise.

## 2.0.0

This release preserves the original courtroom doctrine, goalpost ladder, delta recording, evidence-gated expansion, clean-audit principle, and strict separation between audit and repair. It modernizes the protocol where current codebases and coding agents make line-only, file-count-only reasoning brittle.

### Stronger center derivation

- Separates the failure observation point from the causal center.
- Supports LINE, SYMBOL, EDGE, STATE, EVENT, GENERATED, and PAIRED centers.
- Treats a user-supplied code reference as high-value evidence that must be verified, not automatic proof.
- Replaces "topmost application frame is the center" with the first frame or edge where the contract can actually be violated.
- Adds a falsifiable defect claim before expansion.
- Limits center pivots to prevent search thrash.

### Revision and baseline integrity

- Captures repository root, target revision, baseline, dirty state, package/service, and environment.
- Uses merge-base-aware branch/PR comparison instead of blindly using `HEAD~1`.
- Supports non-Git builds and artifacts with hashes, build IDs, or release versions.
- Uses stable revision + path + symbol + line anchors.

### Modern causal navigation

- Keeps the original 0-10 ladder while preferring AST/LSP semantic scopes over raw line radii when available.
- Replaces rigid neighbor file counts with bounded causal hops.
- Adds edge types: VALUE, CONTROL, STATE, CONTRACT, TEMPORAL, CONFIG, PROVENANCE, TEST, and RUNTIME.
- Adds edge bundles for dependency injection, registries, generated clients, schemas, and other indirection.
- Adds search ledgers so a grep miss cannot masquerade as evidence of absence.
- Adds loop guards, a visited frontier, and a two-pivot limit.

### Evidence calibration and convergence

- Adds evidence IDs, channels, A-D strength grades, and independence groups.
- Separates mutation likelihood, impact severity, confidence, and reproducibility.
- Adds contradiction handling rather than forcing premature resolution.
- Adds a bounded independent-witness protocol for high-impact, disputed, distributed, or AI-related claims.
- Explicitly rejects majority vote as causal proof.

### Autonomous and CI operation

- Adds INTERACTIVE, AUTONOMOUS, and CI modes.
- Replaces mandatory questioning with conservative decision forks in unattended runs.
- Returns INCONCLUSIVE instead of inventing intent or crossing a safety boundary.

### Modern system coverage

Conditional guidance now covers:

- monorepos and polyglot packages
- generated/transpiled/minified code and source maps
- dependency injection, reflection, plugins, and registries
- async, queued, and distributed systems
- databases, migrations, and data pipelines
- concurrency, caching, ordering, retries, and cancellation
- frontend reactive state, SSR, and hydration
- configuration, feature flags, policy, and infrastructure
- dependency and lockfile edges
- AI/LLM, RAG, tool, memory, parser, and multi-agent systems

### Safer agent execution

- Treats repository and tool content as untrusted data, not instructions.
- Adds prompt-injection, secret-exposure, network, MCP, test, build, database, and workspace safeguards.
- Requires inspection of unfamiliar commands before execution.
- Prefers disposable worktrees, sandboxes, or containers for potentially mutating verification.

### Better verification and handoff

- Adds oracle independence, fixture validity, isolation, and side-effect checks.
- Supports property, characterization, differential, trace replay, static, screenshot, log, and manual verification when direct reproduction is infeasible.
- Adds nondeterministic sample/envelope requirements for AI and race-like behavior.
- Upgrades "smallest sufficient safe change" into a repair contract with allowed scope, forbidden scope, compatibility invariants, and reversibility.
- Adds a machine-readable JSON Schema for orchestrators and downstream repair agents.

### Packaging and maintainability

- Packages the skill inside one standards-compliant top-level `center-audit/` directory.
- Uses portable forward-slash ZIP paths.
- Adds validation and packaging scripts.
- Adds trigger evals and a behavior rubric.
- Keeps `SKILL.md` below the recommended 500-line progressive-disclosure threshold.