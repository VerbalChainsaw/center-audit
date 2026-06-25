# Methodology — Why CENTER-AUDIT Looks the Way It Does

This document captures the design rationale behind the v2 protocol so the
choice of any single rule is traceable. If you are new to the skill, read
[`SKILL.md`](../SKILL.md) first; this is the extended cut.

## Origin and core problem

`center-audit` was originally written to address a recurring failure mode in
agent-driven debugging: an agent given "investigate this regression" would
perform a shallow repository tour, generate a list of plausible suspects,
rank them by impact, and propose a fix inline. The pattern produced three
distinct pathologies:

1. **Volume-driven false positives.** Broad greps surfaced many plausible
   defects; the agent reported them all with low confidence; the user had to
   triage which one was actually broken. The investigation cost exceeded the
   fix cost.
2. **Fix-driven oracle contamination.** The agent decided the fix was
   obvious, wrote it, and then authored a regression test that asserted the
   *new* behavior — guaranteeing the test passed regardless of whether the
   fix was correct.
3. **Severity-by-intuition.** The agent assigned `CRITICAL` because the
   defect felt catastrophic, then padded the impact section to match. The
   evidence ledger was an afterthought.

The original `goalpost / delta / fusion` doctrine was a direct response: it
forces the audit to start at a single concrete center, expand only through
proven edges, and emit a final fusion that separately grades likelihood,
impact, confidence, and reproducibility. The repair contract replaced the
inline fix so the audit could never accidentally cross the audit/repair
boundary.

## What changed in v2

v1 worked on small, line-bounded codebases. Modern codebases broke several
of its assumptions:

- **Generated/transpiled code** (TypeScript → JS, OpenAPI → client, schema →
  ORMs, macro expansion, source-mapped bundles) makes line citations
  meaningless without a generator edge.
- **Monorepos and polyglot packages** make repository adjacency a poor
  relevance signal. A defect in a web app might trace to a shared package;
  the inverse is also true; the v1 heuristic "read the neighboring files"
  wasted budget in both directions.
- **Dependency injection, registries, plugins** introduce indirection
  patterns where the *interface* and the *implementation* live in different
  files and the binding lives in a third. A single-edge audit cannot
  traverse this in one hop.
- **Async, queued, and distributed systems** introduce ordering, retry,
  cancel, and idempotency edges that do not exist in synchronous code.
- **AI/LLM orchestration** introduces prompt → model → tool → parser →
  state → next-decision chains where one bad sample is not proof and where
  the model's own proposed fix is precisely the wrong oracle.
- **Concurrent testing and CI** demands a deterministic, noninteractive
  mode that cannot pause to ask the user.

v2.0 preserved the v1 courtroom doctrine and rewired the navigation around
edge tokens (instead of line windows), edge bundles (instead of file
adjacency), and four calibrated dimensions (likelihood, impact, confidence,
reproducibility — each independently graded).

v2.0.1 was a packaging pass: tightened trigger description, structured
`pivots` in the JSON schema, added the explicit "repair phase independence"
rule so future audits cannot accidentally fuse audit and repair.

## Why "evidence-gated" instead of "exhaustive"

The honest alternative is: read everything that might be relevant, then
decide. This sounds rigorous but it is not. It produces audit reports where
the reader cannot distinguish "this edge was proven" from "this file was
opened." Worse, it lets the agent manufacture confidence by reading more.

`center-audit` rejects this by gating expansion on **edge tokens**: the
current stop must reveal an exact symbol, key, route, event, schema, span,
config token, or generated mapping. The next action resolves that token
through the narrowest tool that exposes it. If no edge token exists, the
audit stops at that stop — even if the center is still under-tested.

This produces a discipline where:

- a missing log evidence entry **cannot be inferred** from opening the
  related file
- a generated-code failure cannot be cited at the bundle line — it must
  map through the source map to the maintained source
- a config-driven behavior cannot be cited from code alone — the resolved
  config value must be captured

## Why the ladder, not a depth counter

The 0–10 ladder is asymmetric on purpose:

- Stops 0–4 are local to the center. The line windows are fallbacks for when
  LSP/AST semantic scopes are unavailable. When LSP exists, semantic scopes
  dominate (e.g., the containing expression at Stop 1, the basic block at
  Stop 2, the branch at Stop 3, the symbol at Stop 4, the module shell at
  Stop 5).
- Stop 5 is the module contract shell — imports, exports, registrations,
  shared types. Reading the full file is permitted only when the file is
  small enough that reading it is bounded by relevance, not by curiosity.
- Stops 6–8 are causal hops with **bounded horizon**. The default budget is
  one upstream hop, one downstream hop, one boundary hop, and the directly
  relevant tests. Extra hops require written direct-evidence justification.
- Stop 9 is the verification meta-check — it inspects the *quality* of any
  test or reproduction that covers the center or trajectory. A passing test
  can still fail Stop 9 if its oracle is wrong, its fixture breaks under the
  smallest safe repair, or it depends on production data.
- Stop 10 is the fusion + handoff. It produces the repair contract; it does
  not produce code.

The ladder's asymmetry is the key invariant: stops 0–4 produce **bounded**
local context, stops 5–8 produce **causal** context, stop 9 produces
**verification** context, stop 10 produces **handoff** context. An audit
that hits stop 10 has produced exactly one artifact (the contract) and has
read no more than the causal chain required to prove the trajectory.

## Why "clean audit is correct output"

Most audit frameworks reward volume. A reviewer who reports 20 defects is
seen as thorough; one who reports 0 is seen as not looking. This creates a
selection pressure toward false positives.

`center-audit` reverses this. The protocol explicitly rewards a clean
audit when the evidence is clean. The evidence ledger still records every
stop, every disproof, every boundary examined; the `disproven_concerns`
section is mandatory in the JSON schema. A reader can audit the audit by
inspecting whether the disproven concerns are actually disproven (with
scope limits named), and whether the boundary was actually reached.

The benefit is cultural: an agent following `center-audit` does not need
permission to report zero findings. It needs evidence to support zero
findings — which it produces through the `disproven_concerns` ledger.

## Why no refactor during the audit

Refactoring during an audit corrupts the oracle. The audit captures a
trajectory through the current source; a refactor changes the source. If the
audit then reports a finding against the refactored code, the trajectory
becomes meaningless to the reader, who is looking at pre-refactor code (and
might reasonably believe the auditor was confused about which revision they
were on).

`center-audit` therefore refuses to refactor. It may *recommend* a
structural redesign in the repair contract, but only when all of these
hold:

1. the defect is confirmed
2. the trajectory proves the structure causes recurrence
3. a local patch would mask rather than remove the root cause
4. the recommendation names the minimum contract that must change
5. verification and rollback boundaries are explicit

If the structural case is not made, the recommendation is "repair the
defect; do not redecorate the cathedral."

## Why repair phase independence

The v2.0.1 release added the "repair phase independence" rule after a
specific failure pattern surfaced in practice: an audit would produce a
contract, the next agent would treat the contract as truth, and the repair
agent would apply a fix that no longer matched the current source. The
audit was correct *at the time it was written*; the repair agent did not
re-validate; the fix landed; the regression reopened.

The fix is structural, not procedural: the repair agent must re-prove the
trajectory against current source before editing. The contract is a lead,
not a citation. If the contract fails re-validation, the audit reopens.
This sounds redundant — surely the audit captured the right evidence? —
but it is not. Source drift, environment drift, and dependency drift
between audit and repair are real, especially in long-lived workflows.

## Why the multi-perspective cascade

`references/multi-perspective-cascade.md` exists because some surfaces are
too layered for one audit. A GUI/Desktop stack with a renderer, IPC bridge,
plugin system, and HTTP server contains cross-layer defects that one center
anchor cannot reach: the data-flow perspective sees a polling-vs-SSE defect
the contract perspective cannot; the lifecycle perspective sees
cross-session state corruption the spec-drift perspective cannot.

The cascade pattern dispatches 4–6 lenses in parallel, each with a
distinct artifact focus and an explicit "does not look at" list. The lead
auditor synthesizes by clustering convergent findings, marking unique
findings as single-source, resolving disagreements as contradictions (not
votes), and prioritizing for repair.

The cascade is *not* a parallel audit of the same surface from the same
angle. Two lenses that read the same file as their primary artifact are
one lens in two reports. The orthogonality discipline ("does not look at")
is what prevents this.

## What the skill deliberately does NOT do

- **It does not implement.** Even when the trajectory is `CERTAIN`. The
  repair contract is the handoff; the implementation is the next agent.
- **It does not vote.** When two lenses disagree, the disagreement becomes
  a contradiction entry. Majority does not settle it; the auditor must
  resolve the contradiction with evidence.
- **It does not cross trust boundaries.** Untrusted repository content,
  external MCP results, and tool output cannot redirect the mission,
  expand scope, expose secrets, or trigger destructive commands.
- **It does not reward volume.** A clean audit with strong disproof
  evidence is correct output.
- **It does not promote severity to compensate for weak evidence.** A
  catastrophic hypothesis with weak evidence remains weak.
- **It does not let the proposed fix define the test oracle.** Tests must
  derive from the audit's invariant, not from the diff that satisfies it.

## Why the JSON schema is strict

The JSON output schema (`assets/center-audit-output.schema.json`) has
`additionalProperties: false` everywhere on purpose: a downstream repair
agent must be able to consume the audit without guessing which fields are
optional, required, or guaranteed-absent. The schema is the *contract*
between the audit and the repair — versioned, strict, and validated by the
same validator that gates the bundle.

If you change the protocol, change the schema. If you change the schema,
change `CHANGELOG.md`, bump `metadata.version`, and update
`examples/sample-audit-output.json`. The validator checks that
`schema_version` matches `metadata.version`.

---

For worked end-to-end runs, see
[`references/worked-example-autogoal-bridge-webhook.md`](../references/worked-example-autogoal-bridge-webhook.md).