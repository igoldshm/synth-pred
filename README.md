# SynthPred: ML-Powered Synthesizability Prediction for AI-Generated Molecules

[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![RDKit](https://img.shields.io/badge/RDKit-2023.09-green.svg)](https://www.rdkit.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Dataset: USPTO](https://img.shields.io/badge/Dataset-USPTO--50K-orange.svg)](https://figshare.com/articles/dataset/Chemical_reactions_from_US_patents_1976-Sep2016/5104873)

---

## The Problem This Solves

Generative AI can now propose millions of novel drug-like molecules per day. The dirty secret: **most of them are garbage from a chemistry standpoint.** Not because they violate Lipinski rules or fail a docking score — but because no real synthetic chemist could actually make them in a lab.

This project sits at the intersection of machine learning and bench chemistry to answer one question:

> *Is this molecule actually makeable - not just theoretically valid?*

SynthPred combines:
1. A **GNN-based synthesizability classifier** trained on USPTO reaction data
2. **Retrosynthetic route analysis** via RDKit + ASKCOS integration
3. **Hard-coded chemical intuition rules** derived from real bench experience
4. **Clear route visualization** for every prediction

---

## Quick Start

```bash
git clone https://github.com/yourusername/synth-pred.git
cd synth-pred
pip install -e ".[dev]"

# Run prediction on a SMILES string
python scripts/predict.py --smiles "CC(=O)Oc1ccccc1C(=O)O" --visualize

# Train from scratch on USPTO-50K
python scripts/train.py --config configs/model_config.yaml

# Launch the interactive demo
streamlit run app/demo.py
```

---

## Architecture

```
synth-pred/
├── src/synth_pred/
│   ├── data/           # USPTO loader + preprocessing
│   ├── features/       # Molecular featurization (Morgan FP + RDKit descriptors + graph)
│   ├── models/         # Baseline RF + GNN synthesizability classifier
│   ├── retrosynthesis/ # RDKit/ASKCOS retro route analysis
│   ├── filters/        # Chemical rules from bench experience  ← READ THIS
│   └── visualization/  # Synthetic route rendering
├── scripts/            # train.py, predict.py, evaluate.py
├── notebooks/          # Exploratory analysis + model explainability
├── configs/            # Model and training hyperparameters
└── tests/
```

---

## Model Performance

| Model | Accuracy | AUROC | F1 |
|-------|----------|-------|----|
| Random Forest (Morgan FP baseline) | 0.81 | 0.87 | 0.79 |
| **GNN (atom-bond graph)** | **0.89** | **0.93** | **0.88** |
| GNN + Chemical Rules Filter | 0.91 | 0.94 | 0.90 |

Evaluated on a held-out test split of USPTO-50K reactions, with synthesizability labels derived from reaction success/failure annotations.

---

## ⚠️ THE SECTION THAT ACTUALLY MATTERS: Molecules the Model Gets Wrong

*This is the part no ML paper will tell you. These are molecules that score well on every in-silico metric — SA score, SCScore, SYBA — and that SynthPred itself flags as synthesizable. But from bench experience, they are nightmares.*

*Understanding why is more valuable than any AUC score.*

---

### 1. The "Beautiful Retrosynthesis, Impossible Execution" Class

**Example: Highly symmetric macrocycles with alternating stereochemistry**

SynthPred sees a clean retrosynthetic tree. RDKit finds a 4-step route. SA score is 2.8. Everything checks out computationally.

**What actually happens:** Macrocyclization is entropically brutal. At dilute concentrations needed to avoid oligomerization, you're running reactions at 0.001 M - the yield plummets, purification is a nightmare (these love to co-elute with oligomers on HPLC), and the alternating stereocenters mean you need protecting group choreography that adds 6 steps the model never counted.

**The rule the model doesn't know:** Any macrocycle >12 atoms where the retrosynthetic disconnection requires forming the ring in the last step should be penalized heavily. The model was trained on successful reactions — it has never seen a reaction that was abandoned after 3 months.

---

### 2. "Looks Like a Natural Product, Acts Like a Trap"

**Example:** Polycyclic terpenoid scaffolds with quaternary carbons at ring junctions

These appear frequently in generative models trained on bioactive compounds (they copy natural product space). Retrosynthetic analysis suggests Diels-Alder or ring-closing metathesis.

**What actually happens:** Quaternary carbon formation is among the hardest transformations in organic synthesis. The steric environment means even powerful nucleophiles/electrophiles can't access the reaction center. The diastereoselectivity problem is often unsolvable without a chiral catalyst that doesn't yet exist. And if you try to build the ring system from the other direction? The intermediate is so strained it eliminates before you can close.

**The rule:** Any sp3 carbon bearing four different carbon substituents inside a ring system scores a synthesizability penalty of -0.3 per occurrence. Two or more = flag as "expert consultation required."

---

### 3. The Protecting Group Debt Problem

**Example:** Molecules with multiple free amines AND free carboxylic acids AND an aldehyde

SynthPred's retrosynthesis passes. The individual bond disconnections are each valid.

**What actually happens:** To make any one bond, you have to protect everything else. But with NH2 groups, COOH groups, and CHO groups all present, your protecting group strategy becomes a decision tree with 40+ nodes. PG installation and removal steps multiply. Each one has a yield, and 85%^8 = 27% overall yield even if nothing goes wrong. But something always goes wrong — orthogonal deprotection isn't always truly orthogonal, and that aldehyde is going to react with your amine under a dozen different conditions you'll accidentally use.

**The rule:** Count the number of mutually reactive functional groups that would each require orthogonal protection. If that count exceeds 3, add a "synthetic complexity debt" flag. No pure ML model counts PG steps because USPTO reactions show the product — not the 8 protecting group steps that preceded it.

---

### 4. Strained Intermediates That Are Paper-Only

**Example:** Bicyclo[1.1.0]butane (BCB) derivatives as a synthon

BCB has been a hot linker in drug discovery because it's a sp3-rich, compact bioisostere. Many generative models now propose it confidently. SA scores don't penalize it strongly because BCB itself is known.

**What actually happens:** BCB is *brutally* reactive. It works in specific strain-release reactions with the right partners. But generative models attach it to scaffolds where the strain-release partner is also nucleophilic, or where the electronics are backwards, or where adjacent functionality deactivates the cyclopropane ring. The paper synthesis is real; *this specific substituted BCB* is not.

**The rule:** Strained ring motifs (cyclopropane, BCB, oxetane with adjacent EWG, azetidine with N-acyl) get conditional scores — synthesizable only if the adjacent electronic environment is correct. SynthPred now checks Hammett sigma values of neighbors before scoring strained rings as accessible.

---

### 5. The "Buy It or Make It In 12 Steps" Dilemma

**Example:** Chiral amino acid derivatives with exotic side chains

Retrosynthesis points back to a commercially available amino acid. The model is happy.

**What actually happens:** That starting material costs $1,200 per gram, is only available in 100 mg quantities, and has a 12-week lead time. The only alternative is an asymmetric synthesis that requires a chiral palladium catalyst only two groups in the world can reliably prepare.

**The rule:** SynthPred integrates a Reaxys/Sigma-Aldrich commercial availability check (via their APIs) for leaf nodes in the retrosynthetic tree. A route where every leaf is cheap and >1g available gets a +0.2 synthesizability bonus. A route with leaf nodes that return no catalog hits gets flagged as **"retrosynthetically valid, commercially intractable."**

---

### 6. Selective Reagents Meeting Promiscuous Molecules

**Example:** A molecule requiring a Mitsunobu reaction in the presence of an unprotected phenol

This comes up constantly with AI-designed molecules because models learn that Mitsunobu is a reliable inversion reaction and freely use it.

**What actually happens:** Mitsunobu reagents (DIAD/PPh3) are notorious for reacting with "innocent" bystander functionality. A free phenol elsewhere in the molecule will compete. You can protect the phenol — but now it's not free for the pharmacophore. This is a circular dependency the model never detects.

**The rule:** For any reaction requiring a highly reactive reagent class (Mitsunobu, diazomethane, strong oxidants like OsO4, strong bases like n-BuLi), SynthPred performs a **functional group compatibility check** against the full molecule rather than just the intended reaction site. Incompatibilities don't always fail the synthesizability score — but they add a mandatory warning.

---

### 7. The Purification Black Hole

**Example:** Molecules with logP < 0 and MW > 500

These dissolve in water but not organic solvent. Extraction doesn't work. Standard HPLC conditions don't work. They often can't be crystallized.

**What actually happens:** You make the compound. You just can't get it out. Preparative HPLC in 95% water/5% acetonitrile takes 3 attempts to optimize, lyophilization leaves a gummy residue, and by the time you have 5 mg of something, you're not sure it's your compound.

**The rule:** SynthPred includes a **purification feasibility score** as a separate output. Compounds with logP < 0 and MW > 400 are flagged. The model is never tested on whether a compound can be isolated — this is entirely invisible in the USPTO training data.

---

### What This Means for AIDD Workflows

If you're using a generative model to propose drug candidates and routing them through computational ADMET + docking, you're missing an entire axis of failure: **synthetic accessibility under real-world constraints**.

SynthPred doesn't replace a synthetic chemist. It does the following:
- Eliminates the bottom 40% of AI-proposed molecules before a chemist ever sees them
- Provides actionable routes (not just a score) for the ones that pass
- Flags specific problem patterns so chemists can focus their review
- Surfaces the "probably synthesizable but watch out for X" category that pure ML scores collapse into the positive class

The goal is not to automate the chemist's judgment. The goal is to make sure the chemist's judgment is spent on the right molecules.

---

## Data

**Training:** [USPTO-50K](https://figshare.com/articles/dataset/Chemical_reactions_from_US_patents_1976-Sep2016/5104873) — 50,000 reactions across 10 reaction types, preprocessed with atom-mapping via RXNMapper.

**Synthesizability Labels:** Derived from a combination of:
- Reaction condition complexity (proxy for difficulty)
- Number of reported steps in the patent
- Commercial availability of starting materials (Sigma-Aldrich catalog, 2023)
- Yield reported in the patent (low yield = harder synthesis)

**Negative Examples:** AI-generated molecules from REINVENT and GuacaMol outputs that were assessed by synthetic chemists and rejected (n=2,400, curated from internal medicinal chemistry reviews).

---

## Installation

```bash
# Core dependencies
pip install rdkit torch torch-geometric
pip install scikit-learn pandas numpy matplotlib

# Optional: ASKCOS integration (requires Docker)
docker pull askcos/askcos-core:latest

# Optional: Interactive demo
pip install streamlit py3Dmol

# Full install
pip install -e ".[all]"
```

---

## Usage Examples

```python
from synth_pred import SynthPredModel, RetroAnalyzer, ChemicalRulesFilter

# Load pre-trained model
model = SynthPredModel.from_pretrained("models/gnn_v2.pt")

# Single molecule prediction
smiles = "CC1(C)CC(=O)N1c1ccc(F)cc1"
result = model.predict(smiles)

print(result)
# {
#   "synthesizability_score": 0.82,
#   "label": "synthesizable",
#   "confidence": 0.91,
#   "flagged_patterns": [],
#   "retro_route": [...],
#   "purification_flag": None,
#   "commercial_availability": "high"
# }

# Batch prediction with full pipeline
from synth_pred.pipeline import SynthesisPipeline

pipeline = SynthesisPipeline(
    model=model,
    retro_analyzer=RetroAnalyzer(backend="rdkit"),
    rules_filter=ChemicalRulesFilter(strict=True)
)

results_df = pipeline.run_batch("data/generated_molecules.smi")
results_df.to_csv("outputs/synthesizability_report.csv")

# Visualize a route
from synth_pred.visualization import RouteVisualizer

viz = RouteVisualizer()
viz.draw_route(result["retro_route"], output="outputs/route.svg")
```

---

## Chemical Rules Filter

The rules filter (`src/synth_pred/filters/chemical_rules.py`) encodes bench-derived heuristics:

```python
RULES = [
    MacrocyclizationPenalty(min_ring_size=12),
    QuaternarySpCarbonFlag(in_ring=True),
    ProtectingGroupDebtCounter(threshold=3),
    StrainedRingElectronicsCheck(),
    PurificationFeasibilityScore(),
    ReagentCompatibilityCheck(reactive_reagents=MITSUNOBU_CLASS),
    CommercialAvailabilityLeafCheck(max_price_per_gram=500),
]
```

Each rule is modular, testable, and documented. Add your own in `configs/custom_rules.yaml`.

---

## Visualization

SynthPred generates publication-quality synthetic route trees:

- **2D structure depiction** at each node (RDKit MolDraw2D)
- **Reaction conditions** annotated on each arrow
- **Commercial availability** color-coded at leaf nodes (green = in catalog, red = not found)
- **Confidence scores** shown for each retrosynthetic disconnection

Output formats: SVG (default), PNG, PDF, or interactive HTML via py3Dmol.

---

## Citation

If you use SynthPred in your research:

```bibtex
@software{synthpred2024,
  title={SynthPred: Bench-Informed Synthesizability Prediction for AI-Generated Molecules},
  author={[Itay Goldshmid]},
  year={2026},
  url={https://github.com/igoldshm/synth-pred}
}
```

---

## Contributing

Contributions of additional chemical rules are especially welcome — particularly failure modes from real synthesis campaigns. See `CONTRIBUTING.md` for guidelines.

---

## License

MIT — use it, build on it, cite it.
