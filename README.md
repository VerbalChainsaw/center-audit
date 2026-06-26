# center-audit

> Evidence-gated, read-only, courtroom-disciplined root-cause audit for coding agents.

[![Validate skill bundle](https://github.com/VerbalChainsaw/center-audit/actions/workflows/validate.yml/badge.svg)](.github/workflows/validate.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version: 2.5.1](https://img.shields.io/badge/version-2.5.1-success.svg)](CHANGELOG.md)

`center-audit` is an [Agent Skills](https://agentskills.io)-compatible skill that
turns a coding agent into a forensic root-cause auditor. It performs a
**center-out, evidence-gated, read-only investigation** of a single suspected
defect, regression, persistence failure, contract mismatch, AI/agent loop, or
unsafe behavior change — and returns a calibrated finding, blast radius,
**repair contract** (not an implementation), and verification plan.

It does not edit code. It produces a contract the next agent can independently
re-validate before touching the file.

```text
CENTER-AUDIT v2: READ-ONLY EVIDENCE-GATED GOALPOST / DELTA / FUSION. DO NOT IMPLEMENT.
```

---

## Why this exists

Most "investigate this bug" requests produce one of two failure modes:

1. **Volume-driven audits.** The agent finds 17 plausible issues, ranks them,
   and the human has to triage which one is real.
2. **Fix-driven audits.** The agent decides the fix is obvious and writes it
   inline, contaminating the oracle before verification is even possible.

`center-audit` rejects both. It treats the audit as a courtroom: every claim
must rest on evidence IDs, every trajectory arrow must resolve to a real edge
token, and the only output that may contain a fix is a **repair contract** with
allowed/forbidden scope, compatibility invariants, reversibility, and
failing-before/passing-after tests.

A clean audit (`NO_DEFECT_CONFIRMED`) is correct output when the evidence is
clean.

---

## What's new in v2.5

### `repair_revalidation` field (v2.5.0)

The output schema adds an optional top-level field captured by the **repair agent** (not the audit agent) after applying the fix. It records whether the audit's `required_invariant` was independently re-validated against current source before editing, the method, and one of four results:

- `INVARIANT_HOLDS` — required_invariant still reproduces. Proceed with the contract.
- `INVARIANT_DRIFTED` — anchor or contract drifted; repair proceeded with adjustments in `drift_notes`.
- `INVARIANT_REPLACED` — original invariant was wrong; repair replaced it.
- `CONTRACT_REJECTED` — re-validation failed; repair halted; audit reopens.

This closes the operational enforcement gap on repair phase independence: re-validation is no longer a host-wiring hope, it is a contract field in the audit/repair handoff.

### Tiered pre-flight budget (v2.5.0)

The flat 3/5 pre-flight budget is now tiered by complexity class:

| Complexity class | Norm | Hard cap |
|---|---|---|
| Single defect, well-localized | 3 ops | 5 ops |
| Layered surface, multi-hop | 5 ops | 8 ops |
| Distributed or AI orchestration | 6 ops | 10 ops |

Pick by surface shape (layer count, async/distributed boundaries, AI orchestration involvement), not by user urgency.

### Validator: extension-whitelist replaced with path-only match (v2.5.0)

`scripts/validate_skill.py` now catches missing `.toml`, `.proto`, `.sql`, `.prisma`, `.env.example`, `.txt`, or any other file referenced via `references/`, `scripts/`, `assets/`, or `evals/` path. v2.0.x silently passed through these; v2.5 fails loudly. v2.5.1 also fixes a URL false-positive (URLs containing `/references/foo.md` no longer falsely match).

### Schema relaxations (v2.5.0, backward compatible)

- `case_file.falsifier` is optional — required for `DEFECT_CONFIRMED` and `INCONCLUSIVE`; optional for `NO_DEFECT_CONFIRMED` (the disproof ledger captures it). A `NO_DEFECT_CONFIRMED` audit must now have either a `falsifier` or at least one disproof entry — silent nothing-burgers are no longer schema-valid.
- `contradiction.evidence_ids` `minItems` lowered from 2 to 1, allowing K1-style single-evidence self-corrections. New optional `kind` field on contradictions: `CONTRADICTION | SELF_CORRECTION` (default `CONTRADICTION`).

### Multi-perspective cascade dispatch (v2.5.0)

`references/multi-perspective-cascade.md` dispatch mechanics were generalized from a hermes-specific pitfall to host-agnostic guidance. The hermes `<$text>` flat-string requirement is preserved as a concrete instance of the broader pattern.

---

## When it triggers

Use `center-audit` for a **specific suspected defect or risky behavior change**:

- regression or root-cause isolation against a known-good baseline
- persistence, serialization, migration, or data-integrity failures
- API, CLI, UI, event, schema, or boundary contract mismatches
- state propagation, caching, ordering, race, retry, or idempotency defects
- generated/transpiled/minified code failures that must map to source
- configuration, feature flag, dependency, or environment-specific behavior
- AI/LLM orchestration, tool-schema, retrieval, parser, memory, or
  model-boundary failures
- unclear blast radius before a repair
- **layered surfaces** (GUI + IPC bridge + plugin + HTTP server) where one
  center anchor cannot cover the cross-layer defects

Do **not** use it for: broad code review, greenfield design, formatting,
cosmetic edits, naming, comments, routine test-only changes, or implementing
an already-proven fix.

See [`evals/trigger-cases.json`](evals/trigger-cases.json) for 10 positive and
10 negative trigger cases that gate the skill's activation behavior.

---

## Install

`center-audit` is a standard Agent Skills bundle. Drop the directory into your
agent's skills location:

```bash
# Claude Code
git clone https://github.com/VerbalChainsaw/center-audit \
    ~/.claude/skills/center-audit

# Codex
git clone https://github.com/VerbalChainsaw/center-audit \
    ~/.codex/skills/center-audit

# opencode
git clone https://github.com/VerbalChainsaw/center-audit \
    ~/.config/opencode/skills/center-audit

# Or grab the portable ZIP from the latest release and unzip it into any of
# the above directories. It contains exactly one top-level folder:
#   center-audit/
#     SKILL.md
#     assets/  evals/  references/  scripts/
```

The bundle is standards-compliant: exactly one `SKILL.md` at the bundle root,
frontmatter under the Agent Skills size limits, all referenced resources
present, no portable-path violations. See
[`scripts/validate_skill.py`](scripts/validate_skill.py) for the validator.

---

## How it works

### Operating modes

| Mode | When to use |
|---|---|
| `INTERACTIVE` | Default. A human is available; ask only when product intent or missing runtime facts truly block the case. |
| `AUTONOMOUS` | Told to proceed without interruption. Replaces pauses with explicit decision forks and conservative defaults; stops only when continuing would be unsafe. |
| `CI` | Non-interactive and deterministic. Never asks. Returns `INCONCLUSIVE` with the exact missing evidence when blocked. |

### The goalpost ladder

A 0–10 ladder that prefers LSP/AST semantic scopes over raw line radii:

| Stop | Scope | Question |
|---|---|---|
| 0 | Center anchor | Does the claim hold at the exact line/symbol/edge/state/event? |
| 1 | ±2 lines / enclosing expression | Are operands, guards, and immediate values what the claim assumes? |
| 2 | ±4 lines / basic block | What local control/data flow changes the center's meaning? |
| 3 | ±8 lines / branch or handler | Do branches, catches, lifecycle, or early exits alter the contract? |
| 4 | ±20 lines / containing semantic unit | What does the function promise, mutate, call, and return? |
| 5 | Module contract shell | Imports, exports, registrations, connected siblings. |
| 6 | Upstream causal hop | What produces/validates/configures/schedules the input? |
| 7 | Downstream causal hop | What consumes/persists/renders/republishes the output? |
| 8 | Boundary & runtime reality | API/CLI/UI/DB/FS/queue/network/config/model boundary. |
| 9 | Verification reality | Reproduction, oracle quality, fixtures, isolation, post-repair checks. |
| 10 | Stop & handoff | Fuse evidence, bound blast radius, define repair contract, state non-scope. |

### Evidence-bearing edges

Expansion is permitted **only** when the current stop exposes an edge token of
one of these types — and the next action resolves that exact token:

- `VALUE` — argument, return value, serialized field, payload
- `CONTROL` — call, callback, branch, exception, lifecycle, cancellation
- `STATE` — store field, table, file, cache, lock, shared mutable object
- `CONTRACT` — type, schema, API, CLI, UI, event, tool schema
- `TEMPORAL` — ordering, retry, timeout, debounce, queue, scheduler, race
- `CONFIG` — flag, env value, dependency binding, build option, policy
- `PROVENANCE` — generated source, source map, migration, lockfile
- `TEST` — fixture, assertion, mock, helper, coverage edge
- `RUNTIME` — trace/span, log correlation, metric exemplar, query plan, capture

Modern indirection may require a small **edge bundle** — at most three tightly
coupled artifacts proving one hop (interface + DI binding + implementation,
generated client + schema + server handler, etc.).

Default horizon: one upstream hop, one downstream hop, one boundary hop,
directly relevant tests. Each extra hop requires written direct-evidence
justification.

### Fusion

Every confirmed trajectory arrow needs:

- edge type
- evidence ID (`E#`)
- exact stable anchor (revision + path + symbol + line range, or trace/span,
  or schema version, or command + output hash)
- mechanism proving the hop

Unproven links end the confirmed trajectory. One unproven link forbids
`CERTAIN`. Two sequential unproven links make the path a hypothesis, not a
finding.

Final calibration records:

- `likelihood`: `CERTAIN | LIKELY | POSSIBLE | UNLIKELY`
- `impact`: `CATASTROPHIC | HIGH | MEDIUM | LOW`
- `confidence`: `HIGH | MEDIUM | LOW`
- `reproducibility`: `DETERMINISTIC | INTERMITTENT | NOT_REPRODUCED | NOT_APPLICABLE`

---

## Output formats

### Human-readable report

`SKILL.md` and `references/output-format.md` define the final report shape:

```text
CENTER-AUDIT RESULT: DEFECT_CONFIRMED | NO_DEFECT_CONFIRMED | INCONCLUSIVE

CASE FILE:
  Mode / Repo / Target revision / Baseline / Workspace state / Package / Env

CLAIM:
  Observation / Expected invariant / Audited claim / Center kind / Center
  anchor / Falsifier / Pivots

FUSION:
  Likelihood / Impact / Confidence / Reproducibility / Root cause / Terminal impact

EVIDENCE LEDGER:
  E1 [CHANNEL/strength] [stable anchor] — Method / Result / Relation / Group
  ...

DISPROVEN CONCERNS: ...
CONTRADICTIONS: ...
SAFETY STOPS: ...
REPAIR CONTRACT:
  Objective / Required invariant / Allowed scope / Forbidden scope /
  Compatibility / Reversibility
VERIFICATION PLAN:
  Failing-before mechanism / Passing-after requirements / TEST META-CHECK
NON-SCOPE: ...
STOP REASON: ...
```

### Machine-readable JSON

When the host wants structured output (downstream repair agents, CI
gateways, audit dashboards), emit JSON validated against
[`assets/center-audit-output.schema.json`](assets/center-audit-output.schema.json).
The schema is `draft/2020-12`, versioned at `2.0.1`, with strict
`additionalProperties: false` everywhere — any drift is a contract violation.

See [`examples/sample-audit-output.json`](examples/sample-audit-output.json) for
a minimal schema-valid example.

---

## What ships in the bundle

```text
center-audit/
├── SKILL.md                              # primary instructions (<500 lines)
├── CHANGELOG.md                          # v2.0.1 → v2.0.0 → v1 history
├── ROADMAP.md                            # forward-looking release plan
├── LICENSE                               # MIT
├── README.md                             # this file
├── .gitignore
├── .github/
│   └── workflows/
│       └── validate.yml                  # CI: validator + ZIP build
├── assets/
│   └── center-audit-output.schema.json   # machine-readable output schema
├── evals/
│   ├── trigger-cases.json                # 20 trigger activation cases
│   ├── behavior-cases.json               # 10 must/must-not behavior gates
│   └── behavior-rubric.md                # 10-dimension scoring rubric
├── references/
│   ├── compact-mode.md                   # ultra-compact doctrine (low context)
│   ├── formats.md                        # stop, pivot, decision fork templates
│   ├── evidence-model.md                 # A–D strength, channels, independence
│   ├── execution-safety.md               # trust frame, read-only discipline
│   ├── modern-systems.md                 # monorepos, generated, async, AI edges
│   ├── multi-perspective-audit.md        # when to fan out lenses
│   ├── multi-perspective-cascade.md      # N CENTERs + subagent dispatch
│   ├── output-format.md                  # final report shape
│   ├── prime-tags.md                     # finding taxonomy
│   ├── severity-rubric.md                # CRITICAL/HIGH/MEDIUM/LOW rules
│   └── worked-example-autogoal-bridge-webhook.md  # full end-to-end run
├── scripts/
│   ├── validate_skill.py                 # standards-compliance validator
│   └── package_skill.py                  # portable ZIP builder
├── docs/
│   └── methodology.md                    # extended rationale + design notes
└── examples/
    └── sample-audit-output.json          # schema-valid minimal example
```

`SKILL.md` stays under 500 lines via progressive disclosure — load a reference
only when the case requires it.

---

## Validate

The bundle is self-validating:

```bash
python scripts/validate_skill.py .
# Skill validation passed: .

python scripts/package_skill.py . --output release/center-audit.zip
# Packaged: release/center-audit.zip
# SHA256:   ...
```

The validator enforces:

- exactly one `SKILL.md` at the bundle root, name matches the directory
- frontmatter under Agent Skills size limits
- every `references/`, `scripts/`, `assets/`, `evals/` reference in any `.md`
  resolves to a real file
- every `.json` file parses
- trigger-cases.json is a JSON array with both positive and negative cases
- output-schema `schema_version` matches skill metadata `version`
- no backslash paths anywhere

The ZIP enforces a single top-level `center-audit/` folder with forward-slash
paths so it installs cleanly on Windows, macOS, and Linux.

---

## Repair phase independence

The audit produces a contract; the repair is a separate operation. The repair
agent must **re-validate the invariant** before editing and treat the contract
as a lead, not as truth. A contract that fails re-validation is a contradiction,
not a blocker to bypass.

This means:

- `failing-before` tests are authored or confirmed against the audit's
  invariant, not against the proposed fix's diff
- the oracle is independent of the proposed implementation
- if the trajectory no longer reproduces against current source, the repair
  halts and reopens the audit

---

## Evals

- [`evals/trigger-cases.json`](evals/trigger-cases.json) — 20 cases: 10
  positive (should trigger), 10 negative (should not). Covers regressions,
  distributed queues, AI agent loops, generated code, feature flags, and the
  explicit refusal cases (broad review, refactor, feature design, dependency
  upgrade, performance tour, etc.).
- [`evals/behavior-cases.json`](evals/behavior-cases.json) — 10 must/must-not
  behavior gates: top-frame trap, merge-base baseline, clean-audit discipline,
  generated source mapping, DI edge bundle, distributed-queue correlation, AI
  agent loop envelope, autonomous intent gap, unsafe test script, and
  independent-witness requirement.
- [`evals/behavior-rubric.md`](evals/behavior-rubric.md) — 10-dimension
  scoring rubric (0–2 per dimension, 18–20 = release grade).

---

## Contributing

1. Edit `SKILL.md` or a `references/*.md` file.
2. If you change methodology, also update `CHANGELOG.md` and bump
   `metadata.version` in the `SKILL.md` frontmatter.
3. If you change the JSON contract, also update
   `assets/center-audit-output.schema.json` (`schema_version`) and add or
   update an example in `examples/`.
4. Run `python scripts/validate_skill.py .` locally before pushing.
5. CI will run the validator and build the portable ZIP on every PR.
6. Forward-looking plans live in [ROADMAP.md](ROADMAP.md). Items move through
   versions; the file is updated when scope changes, not on a calendar.

---

## License

[MIT](LICENSE) © Gabriel Zero (VerbalChainsaw).