# Virtual Hit Finding for the Adenosine A<sub>2A</sub> Receptor

**An end-to-end, reproducible ligand- and structure-based virtual screening pipeline against a 53,440-compound Enamine GPCR-targeted library.**

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Babakmamnoon/A2A-Enamine-Virtual-Hit-Finding/blob/main/A2A_Enamine_HitFinding.ipynb)
[![Python](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/)
[![RDKit](https://img.shields.io/badge/RDKit-2026.03-orange.svg)](https://www.rdkit.org/)
[![AutoDock Vina](https://img.shields.io/badge/AutoDock%20Vina-1.2-green.svg)](https://vina.scripps.edu/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Overview

This project screens an Enamine GPCR-targeted screening library for novel antagonists of the
**adenosine A<sub>2A</sub> receptor** (`ADORA2A`, UniProt [P29274](https://www.uniprot.org/uniprotkb/P29274),
ChEMBL [CHEMBL251](https://www.ebi.ac.uk/chembl/target_report_card/CHEMBL251/)) — a class-A GPCR and
validated immuno-oncology target, where tumour-derived extracellular adenosine suppresses effector
T-cell function.

The pipeline runs end to end in a single notebook: chemical curation → molecular representation →
supervised modelling with honest evaluation → AutoDock Vina docking → a transparent composite
ranking → a diversity-selected hit list.

**What distinguishes this project is its evaluation protocol rather than its model.** The modelling
is deliberately conventional. The effort went into measuring performance in a way that predicts
what actually happens when you screen a library — and the result was that a machine-learning model
*did not* beat a trivial similarity baseline once the benchmark was made realistic. That finding is
reported rather than buried.

---

## Quickstart

### Google Colab

Click the badge above. The notebook detects Colab, installs its dependencies, and runs top to
bottom. No GPU required.

### Local

```bash
git clone https://github.com/Babakmamnoon/A2A-Enamine-Virtual-Hit-Finding.git
cd A2A-Enamine-Virtual-Hit-Finding
pip install -r requirements.txt
jupyter lab A2A_Enamine_HitFinding.ipynb
```

### Inputs

| File | Required | Where to get it |
|---|:---:|---|
| Enamine GPCR library (`.csv`, `.smiles`, or `.sdf`) | yes | [enamine.net](https://enamine.net/compound-libraries/targeted-libraries/gpcr-library) |
| ChEMBL A<sub>2A</sub> bioactivities (`.csv`) | yes | Fetched automatically, or export from the [CHEMBL251 report card](https://www.ebi.ac.uk/chembl/target_report_card/CHEMBL251/) |
| `4EIY.pdb` | optional | [RCSB PDB](https://www.rcsb.org/structure/4EIY) — docking is skipped cleanly without it |

Point `CFG.library_path` at your library export and, if the ChEMBL REST API is unreachable, set
`CFG.chembl_csv_fallback` to a manual export. The loader handles the web interface's
semicolon-delimited, quoted format automatically.

**Runtime:** ~25 min for the ligand-based pipeline on 2 CPU cores; ~20 min more to dock 120 compounds.
Docking results are cached, so re-runs are free.

---

## Results

### Pipeline

| Stage | Compounds |
|---|---:|
| Enamine library, as supplied | 53,440 |
| After curation (standardisation, PAINS, de-duplication) | 52,899 |
| After lead-likeness filtering | **48,489** |
| Within the applicability domain | 38,927 |
| Docked (top slice by ligand-based rank) | 120 |
| **Prioritised hits delivered** | **50** |

Training set: 12,644 ChEMBL measurements → **3,644 labelled compounds** (3,014 active / 630 inactive).

### Model performance

Random forest on ECFP4 + 37 physicochemical descriptors, selected on validation PR-AUC across
19 model × representation combinations. All intervals are 95% percentile bootstrap over 1,000
test-set resamples.

| Metric | Scaffold-disjoint test set |
|---|---|
| ROC-AUC | **0.936** [0.914 – 0.956] |
| MCC | 0.666 [0.578 – 0.745] |
| Decision threshold | 0.535 (validation F1 optimum) |

### Structure-based validation

| | |
|---|---|
| Redocking control (ZM-241385, rigid-body) | **0.63 Å RMSD** vs crystal pose |
| Vina score, ZM-241385 | −9.94 kcal/mol |
| Library docking range | −11.3 to −7.4 kcal/mol (median −8.76) |
| Mean ligand efficiency | 0.381 kcal/mol per heavy atom |

### Positive control

Exactly one library compound is also a ChEMBL training compound. It ranked **#1 of 48,489
(top 0.0021%)** — and was then **excluded from the hit list**, because the model was trained on it
and a published compound carries no novelty. Reported as validation, delivered as nothing.

### Hit set

| | |
|---|---|
| Compounds | 50 |
| Unique Murcko scaffolds | 48 |
| Maximum pairwise Tanimoto | **0.484** (ceiling 0.50, verified exhaustively) |
| In the novelty band (T = 0.30–0.55 to nearest known active) | 44 |
| Outside the applicability domain | 0 |
| Top-200 ranking stability under ±50% weight perturbation | median 90% |

---

## What this project found

Five results, none of which were assumed at the outset:

**1. A trivial baseline beat the machine-learning model at realistic prevalence.**
The curated ChEMBL set is **82.7% active** — publication bias, since medicinal chemists publish
what works. That caps EF@1% at 1.21, and every model in the panel scored ≈1.23: pinned to the
ceiling, not tied on merit. BEDROC read 1.000 for everything. Both early-recognition metrics had
**zero resolving power**. Re-evaluating against property-matched decoys at ~2.5% prevalence
reversed the conclusion:

| PR-AUC | Model | Nearest-neighbour similarity |
|---|---|---|
| ChEMBL class balance (82.7% active) | 0.985 | 0.964 |
| Strict decoys (2.55% active) | 0.816 [0.783–0.846] | **0.967** [0.950–0.983] |
| Relaxed decoys (2.49% active) | 0.798 [0.764–0.831] | **0.940** [0.921–0.957] |

The composite score weights similarity accordingly instead of deferring to the model.

**2. The decoy benchmark is itself confounded — and says so.**
Strict decoys are *selected* for low ECFP4 similarity to actives, which is the similarity
baseline's own scoring function. Two decoy sets (0.35 and 0.60 similarity ceilings) are reported to
bracket the answer rather than pretending one number settles it.

**3. Random splitting inflates ROC-AUC by +0.051** on this dataset. Bioactivity data is built from
analogue series; under a random split, members of the same series land on both sides of the split.
Both splits are reported side by side.

**4. The library is already diversity-selected.** At the conventional Butina cut-off, **92% of
clusters are singletons** and the largest holds nine compounds. Cluster-based diversity picking is
close to a no-op here, so selection enforces a direct Tanimoto ceiling between chosen hits instead.

**5. Vendor cLogP disagrees materially with RDKit** (r = 0.72, against 0.98 for MW and 1.00 for
TPSA). Filtering on the vendor value while modelling on the RDKit value would have been a silent
inconsistency. All descriptors are recomputed from standardised structures, and the agreement is
plotted as a QC check.

---

## Method

### Phase 1 — Curation

Format-agnostic ingest (CSV / SMILES / SDF), RDKit standardisation cascade (cleanup → largest
fragment → element whitelist → isotope stripping → normalisation → neutralisation → optional
tautomer canonicalisation), PAINS filtering, InChIKey de-duplication, and lead-likeness filtering.
**Every removal is attributed to exactly one step and counted** — nothing is dropped silently.

ChEMBL bioactivity curation handles four web-export quirks, each of which corrupts a dataset
silently if unhandled — most severely, `Standard Relation` arrives as `'='` with **literal single
quotes**, so a naïve filter for `=` matches zero rows and deletes every usable measurement while
printing a perfectly normal-looking audit table.

Library and training compounds pass through the **identical** standardisation engine, without which
the InChIKey leakage audit and every cross-dataset comparison would be meaningless.

### Phase 2 — Representation and chemical space

Five fingerprints (ECFP4/6, FCFP4, RDKit topological, MACCS) via the current
`rdFingerprintGenerator` API; a 37-feature descriptor block; PCA and UMAP with both datasets
projected into **one shared embedding**; Butina cut-off calibration and MiniBatch k-means at scale;
nearest-active similarity defining the applicability domain and the novelty band.

### Phase 3 — Modelling and honest evaluation

Scaffold-disjoint splitting (with the random split reported alongside), class weighting rather than
SMOTE (the midpoint between two fingerprints is not a molecule), a 19-combination model panel
benchmarked against the no-ML similarity baseline, ROC-AUC / PR-AUC / EF / BEDROC with bootstrap
confidence intervals, property-matched decoy construction, and prospective scoring with
ensemble-variance uncertainty.

### Phase 4 — Structure-based scoring and selection

Receptor preparation from PDB 4EIY, a search box derived from the co-crystallised ZM-241385, a
**redocking control that gates the entire arm** (if Vina cannot reproduce the crystal pose, docking
scores are computed but excluded from the composite), flexible-ligand docking via Meeko, a
transparent weighted composite with sensitivity analysis, and greedy Tanimoto-ceiling diversity
selection.

Each phase ends with a **validation gate** — 19, 28, 24 and 19 hard assertions respectively — that
halts the notebook if any invariant later phases depend on is violated.

---

## Repository structure

```
├── A2A_Enamine_HitFinding.ipynb      # the complete 125-cell pipeline
├── requirements.txt
├── data/
│   └── raw/                          # ChEMBL export, 4EIY.pdb, docking cache
├── results/
│   ├── tables/
│   │   ├── FINAL_prioritised_hits.csv   # ranked hits + per-compound rationale
│   │   ├── FINAL_prioritised_hits.sdf   # same, with properties as SD tags
│   │   ├── model_panel.csv
│   │   ├── generalisation_gap.csv
│   │   └── weight_sensitivity.csv
│   ├── figures/                      # fig01–fig16
│   ├── metrics/                      # per-phase JSON + run manifest
│   └── models/final_model.pkl
└── README.md
```

Every run writes a `run_manifest.json` recording the environment, package versions, random seed,
and full configuration, so any artefact can be traced back to what produced it.

---

## Requirements

```
rdkit>=2024.03
scikit-learn>=1.3
pandas>=2.0
numpy>=1.24
scipy>=1.10
matplotlib>=3.7
umap-learn>=0.5
tqdm
chembl_webresource_client
vina>=1.2          # optional — structure-based arm
meeko>=0.5         # optional — flexible-ligand PDBQT
gemmi              # optional — required by meeko
```

The docking dependencies are optional by design. Without them the notebook reports the
structure-based arm as inactive, renormalises the composite weights over the remaining components,
and completes normally.

---

## Limitations

Stated plainly, because a screening result without its caveats is not a result:

- **No experimental validation.** Every number here is computational. The output is a testable
  hypothesis set, not confirmed binders.
- **Decoys are presumed inactive, not measured inactive.** Some fraction will be genuine binders,
  making the reported enrichments mild under-estimates.
- **Decoy property matching is imperfect** — median decoy MW sits ~50 Da below median active MW,
  because a lead-like library cannot supply matches for the larger ChEMBL actives. This biases
  slightly toward the model.
- **One conformer per ligand.** Standard for a screening funnel, but not a substitute for careful
  pose refinement on compounds you actually order.
- **The redocking control is rigid-body.** It validates receptor preparation, box placement and the
  scoring function — not conformer search.
- **The hit set is flatter than the library** (median Fsp³ 0.17 vs ~0.36). Consistent with the
  planar A<sub>2A</sub> antagonist pharmacophore, but flat aromatic compounds carry solubility and
  developability risk.

---

## Next steps

1. Order the top 20–30 hits and run a radioligand binding assay — the only step that converts this
   into evidence.
2. For any confirmed hit, use its `AnalogsFromREAL` URL to enumerate the surrounding REAL-space
   neighbourhood for rapid SAR expansion — the reason for screening a REAL-backed library.
3. Re-run the docking arm with a properly protonated receptor and confirm the control passes before
   trusting any pose.
4. Fold experimental actives back into the training set and re-run Phases 3–4; the notebook is
   built to be re-run end to end.

---

## Data sources and citation

- **Compound library** — Enamine GPCR-targeted library, 53,440 plated compounds
- **Bioactivity data** — ChEMBL, target CHEMBL251 ([Zdrazil *et al.*, *Nucleic Acids Res.* 2024](https://doi.org/10.1093/nar/gkad1004))
- **Structure** — PDB [4EIY](https://www.rcsb.org/structure/4EIY), A<sub>2A</sub>AR–BRIL with ZM-241385 at 1.8 Å ([Liu *et al.*, *Science* 2012](https://doi.org/10.1126/science.1219218))
- **Docking** — AutoDock Vina 1.2 ([Eberhardt *et al.*, *J. Chem. Inf. Model.* 2021](https://doi.org/10.1021/acs.jcim.1c00203))
- **Cheminformatics** — [RDKit](https://www.rdkit.org/)

```bibtex
@software{mamnoon_a2a_hit_finding,
  author  = {Mamnoon, Babak},
  title   = {Virtual Hit Finding for the Adenosine A2A Receptor from an
             Enamine GPCR-Targeted Library},
  year    = {2026},
  url     = {https://github.com/Babakmamnoon/A2A-Enamine-Virtual-Hit-Finding}
}
```

---

## Author

**Babak Mamnoon, Ph.D.** — Computational Chemistry · Cheminformatics · AI/ML for Drug Discovery

[GitHub](https://github.com/Babakmamnoon)

## License

MIT
