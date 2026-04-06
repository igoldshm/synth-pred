# Contributing to SynthPred

Contributions are welcome — especially new chemical rules.

## Adding a New Chemical Rule

Chemical rules are the most valuable contribution you can make. A good rule is:

1. **Grounded in real bench experience** — not a heuristic from a paper, but a failure mode you or a colleague have actually encountered
2. **Specific** — "macrocycles are hard" is too broad; "macrocycles >12 atoms where the ring-closure step must be last are unreliable due to oligomerization" is specific
3. **Testable** — comes with at least two pytest tests: one passing and one failing example

### Rule Template

```python
# src/synth_pred/filters/chemical_rules.py

class YourRuleName(ChemicalRule):
    """
    One-line summary.

    Bench reality: [Describe the actual synthesis failure mode in 2-4 sentences.
    What goes wrong? Why doesn't the ML model catch it? What's the workaround?]
    """

    name = "your_rule_name"
    description = "Short description"

    def __init__(self, penalty: float = -0.10):
        self.penalty = penalty

    def apply(self, mol: Mol) -> RuleResult:
        # Your SMARTS pattern or descriptor logic
        ...
        return RuleResult(
            rule_name=self.name,
            passed=True/False,
            penalty=0.0 or self.penalty,
            warning="Human-readable explanation of what's wrong and why.",
            severity="info" | "warning" | "critical"
        )
```

Then add it to `ChemicalRulesFilter.__init__()` and write tests in `tests/test_filters.py`.

## Running Tests

```bash
pytest tests/ -v
```

## Code Style

```bash
pip install black ruff
black src/ tests/ scripts/
ruff check src/ tests/ scripts/
```

## What We're NOT Looking For

- Rules that simply restate existing SA score / SCS score thresholds
- Rules that aren't validated against actual synthesis outcomes
- Over-broad rules that penalize large classes of reasonable molecules
