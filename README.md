# Noise-Robust Quantum Machine Learning for Building Energy Performance

A reproducible, five-seed study of how classical regressors and a shallow quantum neural network (QNN) hold up when their inputs are corrupted, using the ENB2012 building energy-efficiency benchmark to predict **Heating Load** and **Cooling Load**.

The goal is an honest robustness comparison, not a quantum-advantage claim. Every result is reported per seed and then summarised as mean ± sample SD with a two-sided 95% Student-t confidence interval across seeds.

---

## Research questions

1. How much does clean-test performance degrade under six controlled input corruptions?
2. Do degradation patterns differ by target (heating vs. cooling) and by model?
3. Does training on matched noise improve robustness?
4. On identical small subsets, how does a shallow four-qubit QNN compare with classical baselines?
5. Can warm-start SPSA fine-tuning recover QNN performance under corruption?

---

## Repository contents

| File | What it is |
|---|---|
| `Noise_Robust_QML_Building_Energy_Paper_FULL_5Seeds.ipynb` | The full experiment, from data loading to findings, with saved outputs |
| `findings.txt` | Headline results written automatically by the notebook's final cells |
| `reproducibility.json` | Seeds, seed roles, protocol choices, selected hyperparameters and package versions from the reported run |

The dataset is **not** included; see [Data](#data).

---

## Method at a glance

### Data and splits
- **Dataset:** ENB2012 (768 simulated buildings, 8 features, 2 targets).
- **Features:** Relative Compactness, Surface Area, Wall Area, Roof Area, Overall Height, Orientation, Glazing Area, Glazing Area Distribution.
- **Seeds:** `11, 23, 42, 71, 101`. Each seed gets its own 80/20 split, and derived seeds control the quantum subset, model initialisation, train/test corruption streams and SPSA (see `reproducibility.json`).
- **Leakage safety:** imputation, scaling, PCA, target scaling, noise magnitudes and clipping bounds are all fitted on that seed's training data only.

### Models
| Model | Setup |
|---|---|
| Ridge | Median imputation → standard scaling → Ridge (α = 1.0) |
| Random Forest | Median imputation → 200 trees, `min_samples_leaf=2` |
| RBF SVR | Median imputation → standard scaling → SVR (C = 20, ε = 0.1) |
| QNN | Median imputation → scaling → PCA to 4 components → angle scaling to [−π, π] → `ZZFeatureMap` (4 qubits) + `RealAmplitudes` (linear entanglement), trained with COBYLA on a noiseless statevector simulator |

The QNN is trained on Heating Load only (96 training / 64 test rows per seed) because simulation is costly. The classical models are refitted on the same rows for a fair subset comparison.

**Circuit selection:** a training-only validation ablation over ansatz repetitions {1, 2} and COBYLA budgets {40, 80, 120}, run on three seeds. The smallest configuration within 0.10 RMSE of the best is kept. The reported run selected **2 repetitions, budget 120**.

### Six input-noise families
Severities `0.0, 0.1, 0.2, 0.3, 0.4`. Continuous features are clipped back to the training range after corruption.

| Noise | Definition |
|---|---|
| Gaussian | Additive zero-mean noise, SD = severity × training SD |
| Multiplicative | Relative noise, factor `1 + N(0, severity)` |
| Systematic bias | One signed shift per feature and seed, magnitude = severity × training SD |
| Missingness | Each of the 8 inputs masked with probability = severity |
| Rounding | Quantisation step = severity × training range / 3 |
| Outlier | With probability = severity, a ±3 training-SD jump |

Within each seed and noise family, the same random stream is reused across severities, so the severity curves are paired.

### Experiments
1. **Classical baselines** on both targets, all five seeds.
2. **Classical robustness curves**: clean-trained models evaluated on corrupted test sets.
3. **Matched-noise training** (Heating Load): Gaussian, missingness and outlier at severity 0.3.
4. **Five-seed QNN** with paired robustness curves and a same-rows classical comparison.
5. **Warm-start SPSA** (30 iterations) from each seed's learned QNN weights, under Gaussian and missingness noise at severity 0.3.
6. **Damage AUC**: area under the relative-RMSE-increase curve, integrated within each seed before summarising (signed and positive-only versions).
7. **Single-feature sensitivity**: corrupt one feature at a time with a Random Forest on Heating Load.

---

## Key results (reported run)

**Clean classical baselines (full data, test RMSE, mean ± SD over 5 seeds)**

| Target | Ridge | Random Forest | RBF SVR |
|---|---|---|---|
| Heating Load | 2.93 ± 0.18 | **0.51 ± 0.09** | 2.10 ± 0.25 |
| Cooling Load | 3.12 ± 0.24 | **1.73 ± 0.15** | 2.42 ± 0.19 |

**Headline findings** (from `findings.txt`)

- **Most damaging corruption:** multiplicative noise on the Random Forest, for both targets. Positive-only damage AUC is 4.541 (95% CI 3.712–5.369) for Heating Load and 1.049 (95% CI 0.815–1.284) for Cooling Load. The Random Forest's very low clean error makes its *relative* degradation large.
- **QNN clean performance:** RMSE 11.739 ± 0.869 SD (95% CI 10.660–12.818), with mean R² ≈ −0.43. On this reduced, PCA-compressed setup the QNN does worse than predicting the mean and well behind every classical baseline.
- **Warm-start SPSA:** small mean recoveries whose confidence intervals include zero. Gaussian: 0.667 RMSE (95% CI −0.171 to 1.505). Missingness: 0.338 (95% CI −0.157 to 0.833). In both cases 4 of 5 seeds improved.
- **Most sensitive feature:** Relative Compactness (mean RMSE increase 1.807 when corrupted alone).

> **Interpretation boundary:** this is a simulated, PCA-compressed, noiseless-statevector study. It does **not** establish quantum advantage.

---

## Getting started

### Requirements
Versions used in the reported run (Python 3.13):

```
numpy==2.2.3
pandas==2.2.3
scipy==1.17.1
scikit-learn==1.8.0
seaborn==0.13.2
matplotlib
openpyxl
qiskit==2.3.0
qiskit-machine-learning==0.9.0
```

```bash
pip install numpy pandas scipy scikit-learn matplotlib seaborn openpyxl \
            qiskit qiskit-machine-learning jupyter
```

### Data
Download the **Energy Efficiency** dataset (Tsanas & Xifara, 2012) from the [UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/242/energy+efficiency) and save `ENB2012_data.xlsx` next to the notebook. The notebook also checks your working directory and `~/Downloads`. Alternatively, set `DATASET_PATH` in the configuration cell.

### Run
```bash
jupyter notebook Noise_Robust_QML_Building_Energy_Paper_FULL_5Seeds.ipynb
```
Then **Run All**. The notebook is locked to full five-seed mode. QNN training takes about a minute per seed on a laptop, plus the circuit ablation and SPSA runs. To skip the quantum sections, set `RUN_QUANTUM = False`. They are also skipped automatically if Qiskit cannot be imported.

### Outputs
Everything is written to `results_paper_full_5seeds/`:

```
results_paper_full_5seeds/
├── tables/          # per-seed CSVs and cross-seed summaries for every experiment
├── plots/           # damage AUC, feature sensitivity, circuit ablation, QNN robustness
├── findings.txt
└── reproducibility.json
```

Every summary table has a matching `*_by_seed.csv`, so the raw seed-level results are always available.

---

## Limitations

- ENB2012 is a small simulated-building benchmark, not deployment data.
- The six corruptions are controlled stress tests, not learned field-error distributions.
- Five seeds capture variability only coarsely, so confidence intervals are wide.
- Model/noise/target comparisons are exploratory; no unadjusted p-values are used to declare winners.
- The QNN uses four PCA components, shallow circuits and noiseless simulation. The ablation and optimiser budgets do not prove convergence.
- Runtime numbers depend on hardware and software.

---

## Citation

If you use this work, please cite the repository:

```bibtex
@misc{kapoor_noise_robust_qml_building_energy,
  author       = {Kapoor, Pratham},
  title        = {Noise-Robust Quantum Machine Learning for Building Energy Performance},
  howpublished = {\url{https://github.com/PrathamKapoor/Noise_Robust_QML_Building_Energy}}
}
```

Please also cite the dataset:

> A. Tsanas and A. Xifara, "Accurate quantitative estimation of energy performance of residential buildings using statistical machine learning tools," *Energy and Buildings*, 49, 560–567, 2012.

## License

No license file is currently included. Add one (e.g. MIT) to make reuse terms clear.
