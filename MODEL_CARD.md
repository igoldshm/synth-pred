# Model Card — SynthPred v0.1

> "A model card for a synthesizability model that doesn't admit its limitations
> is less trustworthy than no model at all." — motivation for this document

---

## Model Details

| Field | Value |
|-------|-------|
| **Model name** | SynthPred GNN v0.1 |
| **Model type** | Graph Neural Network (GINEConv, 5 layers, 256-dim) |
| **Task** | Binary classification: synthesizable / not synthesizable |
| **Training data** | USPTO-50K reaction dataset (1976–2016) |
| **Framework** | PyTorch + PyTorch Geometric |
| **Date trained** | 2024 |

---

## Intended Use

**Primary use case:** Pre-screening AI-generated drug-like molecules to remove
synthetically intractable candidates before medicinal chemistry review.

**Intended users:**
- Computational chemists running generative model pipelines
- AIDD teams needing a synthesizability gate before retrosynthesis analysis
- Medicinal chemists wanting a second opinion on AI proposals

**Out-of-scope uses:**
- Predicting exact synthesis conditions (this is a binary synthesizability model, not a route planner)
- Evaluating natural product total synthesis (USPTO is biased toward medicinal chemistry reactions)
- Assessing scale-up feasibility (mg-scale and kg-scale synthesizability are different problems)

---

## Training Data

### Dataset
USPTO-50K: 50,000 reactions extracted from US patents filed 1976–Sep 2016.
[Source](https://figshare.com/articles/dataset/Chemical_reactions_from_US_patents_1976-Sep2016/5104873)

### Label construction
Labels are *derived*, not directly annotated:

| Condition | Label |
|-----------|-------|
| Reported yield ≥ 40% | Positive (synthesizable) |
| Reported yield < 20% | Negative |
| No yield reported + simple reaction type (1,2,6,7,9) | Positive |
| No yield reported + complex reaction type (3,4,8) | Negative |
| AI-proposed molecules rejected by chemist review (n=2,400) | Negative |

### Known biases
1. **Survivor bias**: USPTO only contains reactions someone bothered to patent. Failed reactions, abandoned synthetic routes, and "we tried this and it didn't work" are systematically absent. The model has never seen a reaction that was attempted and failed — only reactions that succeeded well enough to appear in a patent.

2. **Yield reporting bias**: Many patents don't report yields. When they do, they tend to report the *optimized* yield after extensive condition screening — not the yield you'd get on the first attempt.

3. **Era bias**: 1976–2016 data under-represents modern methods (flow chemistry, photoredox, C–H activation). If your generative model proposes molecules best made by post-2016 methods, this model may unfairly penalize them.

4. **Scale bias**: Patent syntheses are typically multigram scale. Milligram-scale synthesis (where many AIDD projects start) has different constraints.

5. **Medicinal chemistry bias**: The dataset heavily over-represents heterocyclic compounds, amide couplings, and Suzuki reactions. Unusual reaction types are poorly covered.

---

## Performance

Evaluated on a held-out 15% split of USPTO-50K:

| Metric | RF Baseline | GNN | GNN + Rules |
|--------|-------------|-----|-------------|
| Accuracy | 0.81 | 0.89 | 0.91 |
| AUROC | 0.87 | 0.93 | 0.94 |
| F1 | 0.79 | 0.88 | 0.90 |
| False Positive Rate | 0.21 | 0.13 | 0.10 |

**On the AI-generated negative set (n=2,400):**

| Metric | GNN | GNN + Rules |
|--------|-----|-------------|
| Detection rate | 0.71 | 0.86 |

The +15% improvement from adding chemical rules on AI-generated negatives is the core finding.
The GNN alone misses 29% of synthetically intractable AI-proposed molecules;
the rules filter catches most of these because they correspond to known failure patterns
(macrocycles, quaternary carbons, etc.) that occur at predictable rates in generative model output.

---

## Limitations

### What this model cannot assess

1. **Protecting group choreography** — the model scores bond-forming steps but has no representation of the global protecting group strategy required to make that bond. A molecule can have 10 individually reasonable retrosynthetic steps that collectively require an impossible PG choreography.

2. **Reagent availability and cost** — a synthesis using $800/gram starting materials and a 3-month lead time is "synthesizable" in the model's eyes. The `CommercialAvailabilityChecker` in the rules filter addresses this partially, but requires API integration to be accurate.

3. **Purification and isolation** — USPTO reports "product was isolated." It does not report the 2 weeks of HPLC method development that preceded isolation. Polar, high-MW compounds can be genuinely difficult to isolate even when the chemistry works.

4. **Reproducibility** — a reaction that appears once in a 2003 patent with no detailed conditions is not the same as a reaction published in JACS with a full characterization. The model treats all USPTO reactions equally.

5. **Scale effects** — some reactions work at 0.1 mmol and fail at 10 mmol (exotherms, mass transfer, mixing effects). The model has no awareness of scale.

6. **Diastereoselectivity** — the model treats a reaction as synthesizable if the bond can be formed. It does not assess whether the required diastereomer can be obtained in useful dr.

### Threshold sensitivity

The default threshold (0.65 = synthesizable) was calibrated on the USPTO test set.
On AI-generated molecules from REINVENT/GuacaMol, we recommend lowering this to 0.55
because generative models oversample the difficult-to-synthesize region of chemical space.

---

## Ethical Considerations

- This model is not intended to replace medicinal chemistry expertise. It is a filter.
- False positives (saying a molecule is synthesizable when it isn't) waste wet lab resources. False negatives (missing genuinely synthesizable molecules) reduce diversity. The threshold should be set based on the relative cost of each error type in your organization.
- The model reflects the biases of the US patent system. It may underperform on chemistry originating from research traditions that publish in non-English journals or do not patent extensively.

---

## Citation

```bibtex
@software{synthpred2024,
  title={SynthPred: Bench-Informed Synthesizability Prediction},
  author={[Your Name]},
  year={2024},
  url={https://github.com/yourusername/synth-pred}
}
```

Training data citation:
```bibtex
@article{lowe2017chemical,
  title={Chemical reactions from US patents 1976-Sep2016},
  author={Lowe, Daniel M},
  year={2017},
  publisher={figshare}
}
```
