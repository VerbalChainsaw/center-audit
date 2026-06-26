# Contributing to CENTER-AUDIT

CENTER-AUDIT is a serious skill with a serious audience. The bar is "an adopter can verify every claim in the README against a code path in the bundle, and a reviewer can audit the audit." Everything below serves that bar.

## Principles

1. **Doctrine is load-bearing.** Changes to `SKILL.md` affect what every agent does when triggered. If you change the doctrine, you change the audit shape. Every doctrine change needs a `CHANGELOG.md` entry and a `metadata.version` bump.
2. **Schema is contract.** The JSON schema at `assets/center-audit-output.schema.json` is the public contract between audit producers and repair consumers. Schema additions are additive; relaxations are backward-compatible; breaking changes require a major version bump.
3. **Validator is the gate.** Every PR runs `scripts/validate_skill.py`. If the validator fails, the bundle cannot ship. If your change requires new validation rules, add them to the validator and the smoke test.
4. **Evals are regression tests.** Trigger cases gate activation; behavior cases gate the doctrine. If you add a new failure mode to the skill, add a behavior case. If you add a new exclusion trigger, add a negative trigger case.
5. **Examples are documentation that runs.** Both `examples/*.json` files validate against the schema. If you change the schema, update the examples in the same commit.

## Before opening a PR

Run locally:

```bash
python scripts/validate_skill.py .
python scripts/package_skill.py . --output /tmp/test.zip
python -c "import json, jsonschema; schema = json.load(open('assets/center-audit-output.schema.json')); [jsonschema.validate(json.load(open(p)), schema) for p in __import__('pathlib').Path('examples').glob('*.json')]"
```

All three should pass. If the validator complains about a missing reference, your change introduced a `references/foo.md` citation that doesn't resolve — either add the file or remove the citation.

## Pull request checklist

- [ ] `SKILL.md` updated (if doctrine changed)
- [ ] `assets/center-audit-output.schema.json` updated (if JSON contract changed)
- [ ] `metadata.version` in SKILL.md frontmatter bumped (if anything above changed)
- [ ] `CHANGELOG.md` entry added under the new version header
- [ ] `examples/*.json` updated and still schema-valid (if schema changed)
- [ ] `evals/trigger-cases.json` updated (if activation behavior changed)
- [ ] `evals/behavior-cases.json` updated (if doctrine changed)
- [ ] `scripts/validate_skill.py` updated (if bundle structure changed)
- [ ] `.github/workflows/validate.yml` updated (if CI surface changed)
- [ ] `python scripts/validate_skill.py .` passes locally
- [ ] CI is green on the PR

## Adding a new reference doc

1. Write `references/<name>.md` following the style of existing references (compact, doctrine-flavored, with worked examples).
2. Add a line to the "Progressive disclosure" section in `SKILL.md` that says when to load it.
3. If the doc introduces a new term, add it to `references/prime-tags.md`.
4. Run the validator — it will catch the missing-file citation.

## Adding a new trigger case

1. Edit `evals/trigger-cases.json`.
2. Add both a positive (should trigger) and a negative (should NOT trigger) case for the new pattern.
3. The skill bundle should have at least 8 cases in each direction; the goal is roughly balanced.

## Adding a new behavior case

1. Edit `evals/behavior-cases.json`.
2. Each case has a `prompt`, `must` (list of things the agent MUST do), and `must_not` (list of things it MUST NOT do).
3. The must/must_not pairs encode the doctrine boundary. Keep them concrete (e.g., "use merge-base-aware baseline" not "be careful with git").

## Bumping the version

| Change type | Bump to | Example |
|---|---|---|
| Patch: bug fix, doc clarification, validator tightening | `x.y.Z+1` | `2.5.1` → `2.5.2` |
| Minor: additive schema field, new reference doc, doctrine refinement | `x.Y+1.0` | `2.5.1` → `2.6.0` |
| Major: schema breaking change, methodology shift, repair-contract shape change | `X+1.0.0` | `2.5.1` → `3.0.0` |

The validator enforces that `metadata.version` in SKILL.md matches `schema_version`'s `const` value in the output schema. Both must be bumped together.

## What to NOT change

- **The clean-audit-is-correct principle.** A clean audit with strong disproof evidence is valuable output. Resist pressures to "find something" when the evidence is clean.
- **The audit/repair separation.** The repair contract is the handoff. The audit agent must not implement. The repair agent must re-validate. This is the load-bearing design choice; relaxing it cascades into every other rule.
- **The evidence-gated expansion.** Expansion only through edge tokens. No repository tours. No "while I'm here" reads. The discipline is what makes the audit evidence-backed.
- **The goalpost ladder's 0–10 shape.** Stops 0–4 are local; 5 is module; 6–8 are causal hops; 9 is verification; 10 is handoff. Changing this shape is a methodology-level change, which is what `v3.0` is for.

## Communication

- **Bug reports**: open an issue with a minimal reproduction. Include the schema_version, the result the audit produced, and what you expected.
- **Feature proposals**: open an issue first. CENTER-AUDIT changes affect every adopter. Discuss the design before the implementation.
- **Pull requests**: link to the issue if one exists. Reference the relevant CHANGELOG entry if your PR is part of a planned release.

## Design philosophy in one sentence

> The audit is the courtroom. The repair agent is the bailiff. The schema is the law. Code is the evidence. The doctrine is the procedure. The validator is the bailiff's badge. Every claim must rest on evidence IDs; every trajectory arrow must resolve to a real edge token; the only output that may contain a fix is a repair contract with allowed/forbidden scope, compatibility invariants, reversibility, and failing-before/passing-after tests.

When in doubt: read `docs/methodology.md`. If the question is "is this in scope for v2.x?" the answer is almost always yes if it doesn't change the doctrine, and no if it does.