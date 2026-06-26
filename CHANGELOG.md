# CENTER-AUDIT Changelog

## 2.5.0

Operability hardening. Methodology preserved; the audit/repair separation, evidence-gated expansion, calibrated fusion, and clean-audit principle are unchanged. This release closes operational gaps surfaced during real-world use and makes the doctrine more honest about its own contract.

### Output schema: `repair_revalidation` field (additive, optional)

The JSON output schema adds a top-level `repair_revalidation` object captured by the repair agent after applying the fix:

```json
"repair_revalidation": {
  "revalidated": true,
  "method": "Re-ran E1, E2, E3 against HEAD post-fix...",
  "evidence_ids": ["E1", "E2", "E3"],
  "result": "INVARIANT_HOLDS" | "INVARIANT_DRIFTED" | "INVARIANT_REPLACED" | "CONTRACT_REJECTED",
  "drift_notes": ""
}
```

This closes the operational enforcement gap on repair phase independence: re-validation is no longer a host-wiring hope, it is a contract field in the audit/repair handoff. Field is optional so v2.0.x audits remain valid; v2.5+ repair agents are expected to fill it.

### Output schema: `falsifier` is now optional

`case_file.falsifier` is no longer required by the schema. Required for `DEFECT_CONFIRMED` and `INCONCLUSIVE` audits (the claim must be falsifiable). Optional for `NO_DEFECT_CONFIRMED` audits — the disproof ledger already captures what was tested. Existing v2.0.x outputs with falsifier remain valid; new clean audits may omit it.

### Output schema: `contradiction.evidence_ids` minimum lowered from 2 to 1

Allows K1-style single-evidence self-corrections in the contradiction ledger (e.g., "E7 was originally cited as spec §E-1 but the actual webhook spec lives at §3.1"). The original `minItems: 2` forced these honest self-corrections into awkward 2-ID entries that didn't reflect the actual contradiction.

### Validator: extension-whitelist regex replaced with path-only match

`scripts/validate_skill.py` `RESOURCE_RE` no longer restricts matching to `.md|.py|.json|.yaml|.yml|.sh|.js|.ts`. Any file referenced via `references/`, `scripts/`, `assets/`, or `evals/` path is now checked for existence on disk. This fixes a silent validation gap: a missing `.toml`, `.proto`, `.sql`, `.prisma`, `.env.example`, `.txt`, or other non-whitelisted file referenced in a `.md` would previously pass validation. Now it fails loudly.

### Doctrine: tiered pre-flight budget by complexity class

The flat "3 ops norm, 5 ops hard cap" pre-flight budget is replaced with a three-tier budget selected by surface shape:

| Complexity class | Norm | Hard cap | Profile |
|---|---|---|---|
| Single defect, well-localized | 3 | 5 | One file, one symbol, one invariant |
| Layered surface, multi-hop | 5 | 8 | GUI + IPC bridge + plugin + HTTP server |
| Distributed or AI orchestration | 6 | 10 | Async/queued/cross-process state, or AI agent loops |

The cap was too tight for layered surfaces where 1 boundary + 1 upstream + 1 downstream + 1 verification = 4 ops is the minimum before investigation. The new budget is selected by surface shape, not by user urgency.

### Doctrine: `compact-mode.md` mutual exclusion

`SKILL.md` progressive disclosure now explicitly states that `references/compact-mode.md` is loaded **instead of** `SKILL.md`, not in addition to. Loading both wastes tokens; `compact-mode.md` is the compact stand-in, not a supplement.

### Generalization: multi-perspective dispatch notes

`references/multi-perspective-cascade.md` replaces the hermes-specific `<$text>foo</$text>` flat-string pitfall with host-agnostic guidance ("some hosts accept structured goal/context fields; others require flat strings; check the host's API before writing lens prompts"). The hermes requirement is one instance of a broader pattern.

### Compatibility

- v2.0.x JSON outputs without `repair_revalidation` remain valid.
- v2.0.x JSON outputs with `falsifier` remain valid (no migration needed).
- v2.0.x JSON outputs with single-evidence contradictions remain valid (they were already accepted by the validator; the schema is now also permissive).
- The validator change is stricter: bundles that previously passed because a missing `.toml`/`.proto`/etc. was whitelisted will now fail. This is intentional — it surfaces real bugs in bundle construction.

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
- Replaces “topmost application frame is the center” with the first frame or edge where the contract can actually be violated.
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
- Upgrades “smallest sufficient safe change” into a repair contract with allowed scope, forbidden scope, compatibility invariants, and reversibility.
- Adds a machine-readable JSON Schema for orchestrators and downstream repair agents.

### Packaging and maintainability

- Packages the skill inside one standards-compliant top-level `center-audit/` directory.
- Uses portable forward-slash ZIP paths.
- Adds validation and packaging scripts.
- Adds trigger evals and a behavior rubric.
- Keeps `SKILL.md` below the recommended 500-line progressive-disclosure threshold.
