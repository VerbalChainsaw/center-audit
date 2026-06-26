# CENTER-AUDIT Roadmap

Forward-looking only. Past releases live in [CHANGELOG.md](CHANGELOG.md). Items are ordered by target release, not by priority within a release. Anything here can change without notice — committing this file is a commitment to think about these problems, not a commitment to ship them on this schedule.

---

## v2.5.2 — Tightening

Target: 1–2 weeks after v2.5.1. Pure operability polish. No methodology change, no schema additions, no validator behavior change beyond closing the gaps below. Backward compatible with v2.5.1.

### Schema fixes

- **`disproof.evidence_of_absence` requires non-empty content.** A disproof entry with `evidence_of_absence: ""` and `evidence_ids: ["E1"]` currently satisfies both the `disproof` shape and the `NO_DEFECT_CONFIRMED` guard (via `disproven_concerns` array non-empty). The schema should require `minLength: 1` on `evidence_of_absence` so a one-character placeholder doesn't slip through as a "tested and held" claim. **Why:** this is the same nothing-burger problem the v2.5.1 `NO_DEFECT_CONFIRMED` guard was supposed to prevent, one level deeper. **Compat:** v2.0.x and v2.5.x disproof entries with non-empty `evidence_of_absence` remain valid.

- **`repair_revalidation` adds an `AUDIT_MOOT` result value.** The current four-value enum (`HOLDS | DRIFTED | REPLACED | REJECTED`) does not cleanly model the case where the code under audit was deleted, the feature was deprecated, or the scope no longer applies. `REJECTED` is the wrong fit because there is no contradiction to resolve — the audit is simply no longer relevant. `AUDIT_MOOT` (or `SCOPE_INVALID`) lets the repair agent declare the contract obsolete without implying the audit was wrong. **Why:** surfaced during gap review as an unmodeled case. **Compat:** additive enum value; consumers treating the schema strictly will not break.

### Documentation fixes

- **Update `references/output-format.md`** to document the `repair_revalidation` field in the final human-readable report. The field is in the JSON schema but the prose format guide still treats audit output as ending at `REPAIR CONTRACT`. Add a `REPAIR REVALIDATION:` section between `REPAIR CONTRACT` and `VERIFICATION PLAN`. **Why:** doc consistency; a reader following the prose format would not know to capture this. **Compat:** prose-only change.

- **Update `references/machine-output.md`** to call out that consumers should validate `repair_revalidation.drift_notes` non-empty when `result != INVARIANT_HOLDS` even though the schema already enforces it (defense in depth). Also document the `kind` field on contradictions. **Why:** the schema is the contract; the reference doc is the implementation guide. **Compat:** prose-only change.

### Validator hygiene

- **Fix the smoke test fixture.** The current smoke test creates a tmpdir with skill name `smoke-test-skill` but parent dir `tmpXXXX`, producing a noisy "skill name must match parent directory" error alongside the intended "missing.toml" error. Rename the fixture so the only error is the intended one. **Why:** test cleanliness; the noisy extra error makes a real regression harder to spot if it lands in the same test. **Compat:** test-only change.

- **Audit for nested-path references** in the bundle. v2.5.1's validator URL fix lost validation of nested paths like `subdir/references/foo.md`. The trade-off was documented but I did not verify that no nested paths appear in the current bundle. Run `rg -n '\bsubdir/' --type md` and `rg -n '\.\./references/' --type md` and capture the result. If zero matches, document the audit result in the changelog. If matches exist, decide whether to (a) accept the loss, (b) add a nested-path exception to the regex, or (c) fix the references. **Why:** latent regression from v2.5.1; the trade-off is acceptable only if no real references are silently unvalidated. **Compat:** depends on outcome.

### SKILL.md hygiene

- **Either trim to ≤500 lines, or document why v2.5.x is allowed to exceed the recommended threshold.** The 503-line warning has been accepted silently across v2.5.0 and v2.5.1. Either find the four lines to trim (candidates: prose condensation in repair_revalidation doctrine, tighter tier decision rule) or add a one-line note in the doctrine acknowledging the overage and explaining the rationale (substantive operability hardening, not line bloat). **Why:** the doctrine is more credible when it follows its own rules. **Compat:** no schema or validator change.

---

## v2.6 — Substantive but backward-compatible

Target: after v2.5.2 settles. Adds new functionality; existing v2.5.x outputs remain valid. No methodology change.

### Schema additions

- **`audit_id` (UUID) and `schema_version_pin` (semver range).** Lets consumers detect when an audit output is from an older schema version they may not fully understand, or when a downstream repair agent is operating against a different schema than the audit was authored against. Without this, the `repair_revalidation` field can be silently misinterpreted across schema boundaries. **Why:** the `repair_revalidation` field is the load-bearing inter-agent contract; pinning its schema version is more important than pinning the audit's own. **Compat:** additive fields; both optional; old outputs remain valid.

- **`defect.severity_rationale` (string).** Currently the `defect` schema records `severity` (CRITICAL/HIGH/MEDIUM/LOW) but not the reasoning. A consumer reading the JSON cannot distinguish a HIGH assigned because the defect actually matters from a HIGH assigned because the auditor was being conservative. The rationale captures the why. **Why:** closes a transparency gap that already shows up in the behavior rubric ("Severity: 1 = uses labels but overstates one dimension"). **Compat:** additive field; optional.

### Behavioral eval harness

- **Build a `behavior-eval/` harness** that runs the skill against a fixed suite of fixtures and grades outputs against `evals/behavior-rubric.md`. Today the rubric is documentation; the harness turns it into a regression test. Fixtures would include: clean audit (must produce NO_DEFECT_CONFIRMED with disproof), confirmed defect (must produce DEFECT_CONFIRMED with trajectory), contradictory evidence (must log K# entry), self-correction (must log SELF_CORRECTION kind), insufficient evidence (must return INCONCLUSIVE not guess). **Why:** the rubric has been a load-bearing artifact on paper; it has never been machine-verified. This is the right time to make it real. **Compat:** additive; new directory.

### Reference: cascade dispatch under low-bandwidth conditions

- **`references/multi-perspective-cascade.md` adds a "low-bandwidth cascade" pattern.** The current cascade assumes the lead auditor can dispatch 4-6 subagents in parallel with full prompts. On resource-constrained hosts or tight time budgets, dispatching six subagents is not viable. Add guidance for: (a) which lens to drop first (the spec-drift lens, usually), (b) how to compress remaining lenses into 2-3 short ones, (c) when to fall back to a single-lens deep audit instead of a multi-lens shallow one. **Why:** the cascade is gated on the user asking for "several perspectives" but the protocol doesn't degrade gracefully. **Compat:** prose-only addition.

---

## v3.0 — Methodology-level (breaking)

Target: only if there is a real reason. v3.0 is reserved for changes that affect the audit/repair contract shape or the calibration axes. v2.x is reserved for everything else.

### Candidates (none currently scheduled)

- **Replace the 0–10 goalpost ladder with semantic-step-based expansion** that does not depend on line radii even as a fallback. The current ladder uses line windows (`±2`, `±4`, `±8`, `±20`) when LSP/AST semantic scopes are unavailable. In practice, line windows are wrong often enough that the fallback is itself a source of errors. v3.0 could require semantic scopes (LSP/AST) as the primary and only treat line windows as a last-resort warning, not a default. **Why:** the doctrine already says "prefer LSP/AST semantic scopes; line windows are deterministic fallbacks, not sacred geometry" but the practice is the opposite. v3.0 could enforce the preference.

- **Add `time-to-detect` as a fifth fusion dimension** alongside likelihood, impact, confidence, reproducibility. Today a defect that took 30 minutes to find is graded identically to one that took 6 hours. **Why:** repair prioritization could be informed by detectability — a defect that took hours to root-cause is more likely to recur in production. **Status:** speculative; needs real-world data to justify the dimension.

- **Machine-actionable patch spec** in the repair contract. Today the repair contract is prose (objective, allowed/forbidden scope, allowed/forbidden commands). v3.0 could add a structured `patch_spec` with file path, line range, expected diff shape, and verification hook — letting a downstream repair agent consume the contract without re-deriving intent from prose. **Why:** the audit/repair handoff is currently prose-bridged; the v2.5.x `repair_revalidation` field is the first step toward structured repair contracts. v3.0 could close the loop. **Status:** would need a parser/interpreter design first.

---

## Long-term (no version target)

### Companion skills

- **`fix-with-sketchpad`** — referenced in `SKILL.md` ("skip the sketchpad for single-fix work"). Should exist as a stand-alone skill that the repair agent can invoke for multi-step repair work.
- **`consensus-scan`** — referenced in `references/multi-perspective-cascade.md`. Should exist as the companion that handles the lens-prompt generation when the user asks for cascade but does not name lenses.
- **`audit-handoff`** — a thin skill that wraps the audit → repair cycle: takes an audit JSON, runs `repair_revalidation` against it, applies the fix, emits a post-repair audit cycle. This is what every consumer will want to write; better to provide it.

### Behavioral adoption

- **Submit CENTER-AUDIT as a reference implementation** to the Agent Skills spec maintainers. The skill is a candidate for a "case study" or "example bundle" entry — it shows what a serious skills bundle looks like (validator, schema, evals, behavior rubric, examples, methodology doc, CHANGELOG, ROADMAP, CI).
- **Publish a "writing a CENTER-AUDIT-compatible skill" guide** for other skill authors who want to adopt the evidence-discipline pattern. Not all skills need a full courtroom doctrine, but the JSON schema + behavior rubric + progressive disclosure pattern generalizes well.

---

## What this roadmap deliberately does not promise

- **A release cadence.** Versions ship when they are ready, not on a calendar. v2.5.2 may ship next week or in a month.
- **That every item here will ship.** Items get cut when they turn out to be wrong, or when better solutions emerge.
- **Backward compatibility across major versions.** v3.0 will break v2.x contracts by definition; consumers should pin.
- **That roadmap items are bug-free designs.** Each item still needs a design pass before implementation; the prose here is the *problem statement*, not the *solution*.

---

*Last updated: v2.5.1 ship date. Revisit when v2.5.2 lands or when v3.0 thinking starts in earnest.*