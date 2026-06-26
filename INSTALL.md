# Install

CENTER-AUDIT is a standard [Agent Skills](https://agentskills.io) bundle. Install takes 30 seconds. Verification takes another 30.

## Pick your host

| Host | Install location |
|---|---|
| Claude Code | `~/.claude/skills/center-audit/` |
| Codex | `~/.codex/skills/center-audit/` |
| opencode | `~/.config/opencode/skills/center-audit/` |

## Option 1 — git clone (recommended)

```bash
git clone https://github.com/VerbalChainsaw/center-audit \
    ~/.claude/skills/center-audit
```

Then update with `git pull` from inside that directory.

## Option 2 — Download a release ZIP

Grab the latest from https://github.com/VerbalChainsaw/center-audit/releases, then:

```bash
unzip ~/Downloads/center-audit.zip -d ~/.claude/skills/
```

The ZIP contains a single `center-audit/` folder at the top level. Unzip into any of the install locations above.

## Verify it loaded

The 30-second verification: ask the agent to investigate a known defect in a repo you have access to, and watch for the trajectory structure. If the skill loaded, the agent will:

1. State a concrete center anchor with a falsifier
2. Use evidence IDs (E1, E2, ...) for claims
3. Read only what's needed to resolve the next edge token
4. Emit a repair contract with allowed/forbidden scope
5. Stop at the audit boundary and hand off, NOT implement the fix

Example prompt:

> Use the center-audit skill to investigate why saved drafts disappear after restart in [your repo]. Prove the root cause and blast radius.

If the agent starts reading the whole repository, jumps to a fix without a contract, or asserts severity without a calibration axis, the skill didn't load. Check that the bundle is at the right path and that your host picks up skills from there.

## Verify the bundle itself

Run the validator on the installed bundle:

```bash
python ~/.claude/skills/center-audit/scripts/validate_skill.py \
    ~/.claude/skills/center-audit
```

Expected: `Skill validation passed: ...`

## Build a portable ZIP

```bash
python ~/.claude/skills/center-audit/scripts/package_skill.py \
    ~/.claude/skills/center-audit \
    --output /tmp/center-audit.zip
```

The ZIP has one top-level `center-audit/` folder, forward-slash paths, and an embedded SHA256. Distribute it the same way you'd distribute any Agent Skills bundle.

## Try the example audit

The bundle ships with two schema-valid example outputs:

- `examples/sample-audit-output.json` — a `DEFECT_CONFIRMED` with `INVARIANT_HOLDS` re-validation
- `examples/repair-revalidation-drifted.json` — a `DEFECT_CONFIRMED` with `INVARIANT_DRIFTED` re-validation (source drifted, repair proceeded with adjustments)

Use these to:

- **Test your schema validator.** If your consumer validates against `assets/center-audit-output.schema.json`, both examples should pass.
- **Train your repair agent.** Show it the `repair_revalidation` shape. It should fill in the field after applying a fix.
- **Calibrate your eval harness.** The behavior rubric in `evals/behavior-rubric.md` describes how to grade a run.

## Run the trigger evals

`evals/trigger-cases.json` has 20 cases — 10 positive (should trigger) and 10 negative (should NOT). Use them to verify your host's skill-activation logic:

- Positive cases should make the agent enter the audit protocol (produce a center anchor, evidence ledger, etc.)
- Negative cases should make the agent respond normally without invoking the audit machinery

If your host triggers on a negative case (e.g., "review this whole repository for code quality"), it is mis-calibrated and the trigger description in `SKILL.md` frontmatter may need adjustment.

## Update

```bash
cd ~/.claude/skills/center-audit
git pull
```

CI runs on every push to main. Check https://github.com/VerbalChainsaw/center-audit/actions for the current green/red status before updating.

## Uninstall

```bash
rm -rf ~/.claude/skills/center-audit
```

The skill bundle is self-contained. Removing the directory removes the skill. No config files, no global state, no daemons.

## Troubleshooting

**Skill doesn't load after install.** Verify the path matches what your host expects. Claude Code looks at `~/.claude/skills/`, Codex at `~/.codex/skills/`, opencode at `~/.config/opencode/skills/`. The directory name MUST be exactly `center-audit` — the skill name in `SKILL.md` frontmatter must match the directory name (the validator enforces this).

**Agent triggers on every "find the bug" prompt including ones that should be skipped.** This means the trigger description in `SKILL.md` is too broad for your host. Open an issue with the false-positive prompt and the agent's response.

**Agent doesn't trigger on a clear positive case.** Either the trigger description doesn't match your host's matching logic, or the case isn't in `evals/trigger-cases.json`. Open an issue with the missed prompt.

**Validator fails on a known-good bundle.** Check that no reference doc was edited to point at a file that doesn't exist. Run `python scripts/validate_skill.py .` from the bundle root to see the exact error.

**JSON output doesn't validate against the schema.** Check that `schema_version` matches the `const` value in your consumer's schema. v2.5.1 consumers expect `"2.5.1"`. v2.0.x consumers will accept v2.5.x outputs (backward compat) but not the reverse.

## Compatibility matrix

| Your bundle version | Accepts these audit outputs |
|---|---|
| v2.5.1 | v2.0.x, v2.5.0, v2.5.1 |
| v2.5.0 | v2.0.x, v2.5.0 |
| v2.0.x | v2.0.x only |

The validator and schema are forward-compatible within the v2.x line. A v3.x consumer may or may not accept v2.x outputs — see ROADMAP.md for the v3.0 plan.