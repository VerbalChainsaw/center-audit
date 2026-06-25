# CENTER-AUDIT Changelog

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
