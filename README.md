Smart-Contract Vulnerability Detection — Reproduction, Audit & Improvement
Final-project repository for the critical re-analysis of
> Bui Tong Nha & Nguyen Dinh Thuan (2022). *Methodology Interaction by Machine Learning Model to Detect
> Vulnerability in Smart Contract of Blockchain.* 2022 RIVF International Conference on Computing and
> Communication Technologies (RIVF), IEEE, pp. 112–117.
The paper detects "harmful" points with unsupervised K-means (k=5) and a tanh autoencoder.
This repository (1) audits the paper's reported numbers, (2) re-implements both methods,
and (3) proposes and tests a supervised, leakage-controlled, imbalance-aware alternative on real data.
Key findings (all numbers produced by the code in this repo)
#	Experiment	Data	Main result
0	Audit of paper's Table 1	Paper's Tables 2–3	Table 1 is reproduced exactly with clusters {2,4} = harmful. Hidden consequence: precision of the harmful class = 0.0032 % on test, i.e. 31,106 false alarms per true detection.
2	Function-level detection	Real — SolidiFI-benchmark (500 unique functions, 46 vulnerable)	Proposed TF-IDF + LR: PR-AUC 0.940 ± 0.080, F1 0.847, MCC 0.838 vs. paper's K-means PR-AUC 0.190 and Autoencoder 0.352 (Nadeau–Bengio corrected p < 0.001).
2b	Leakage demo	Real	Raw identifiers + no de-duplication → PR-AUC 1.000: an artefact of label-revealing variable names, not learning.
2c	Unseen vulnerability type (leave-one-type-out)	Real	Proposed keeps ROC-AUC 0.972 / precision 0.802, but recall drops 0.855 → 0.527; keyword rules recall 0.794 (precision 0.297); K-means 0.609, AE 0.513 ROC-AUC.
1	Extreme imbalance (100 / 100,100)	SYNTHETIC — illustrative only	K-means reproduces the paper's pathology (balanced acc 0.79, harmful precision 0.24 %). Hybrid model: best PR-AUC (0.440) and Recall@FPR1 % (0.74 vs 0.40), but unstable F1 because ~20 positives are too few to tune a threshold.
Limits of the claim: the proposed model beats the paper's methods and our baselines under this protocol on
SolidiFI. SolidiFI bugs are injected from templates, so real-world performance will be lower; no comparison
with published deep-learning detectors (e.g. GNN, Zhuang et al., IJCAI 2020) or with Slither run on the same functions was made.
Repository structure
```
sc-vuln-detection/
├── README.md
├── requirements.txt
├── LICENSE
├── scripts/download_data.sh          # clones SolidiFI-benchmark into data/
├── src/
│   ├── data_solidifi.py              # function extraction, labels from BugLog, identifier normalisation, de-dup
│   ├── data_synthetic.py             # SYNTHETIC extreme-imbalance generator (assumptions documented)
│   ├── models.py                     # K-means (paper), Autoencoder (paper), Isolation Forest, Hybrid (proposed)
│   └── evaluation.py                 # PR-AUC, F1/MCC of positive class, Precision@k, thresholds, tests
├── experiments/
│   ├── run_exp0_paper_audit.py       # recompute paper's Table 1 from Tables 2–3
│   ├── run_exp1_synthetic.py         # synthetic replication of the paper's setting
│   ├── run_exp2_solidifi.py          # real-data experiment + leakage demo + leave-one-type-out
│   ├── stats_tests.py                # Wilcoxon + Nadeau–Bengio corrected resampled t-test
│   └── make_figures.py               # figures from results/*.csv
├── notebooks/colab_demo.ipynb        # one-click Google Colab walkthrough
├── results/                          # CSV outputs + figures produced by the runs reported above
└── tests/test_core.py                # unit tests (incl. exact reproduction of the paper's Table 1)
```
Quick start (Google Colab or local, CPU only)
```bash
pip install -r requirements.txt
python -m experiments.run_exp0_paper_audit            # seconds, no data needed
bash scripts/download_data.sh                         # ~440 MB
python -W ignore -m experiments.run_exp2_solidifi     # ~20-25 min (use --repeats 1 for ~3 min)
python -W ignore -m experiments.run_exp1_synthetic    # ~6-10 min
python -m experiments.stats_tests
python -m experiments.make_figures
python -m tests.test_core
```
Method summary
Paper re-implementation. K-means (k = 5, mini-batch) on standardised features; clusters whose share of
harmful points exceeds the base rate are named harmful (the paper merged clusters 2 and 4 the same way).
Autoencoder: under-complete tanh MLP trained on non-harmful training points; score = log reconstruction error;
threshold chosen on an inner validation split (the paper chose it from a precision–recall plot).
Proposed pipeline.
Function-level unit, label = overlap with injected-bug lines (SolidiFI BugLog).
Normalisation: comments removed, Solidity keywords and security-relevant APIs kept, user identifiers → `ID`,
literals → `NUM`/`STR` (SolidiFI names injected variables e.g. `callme_re_ent7`, which would leak the label).
Global de-duplication of normalised bodies (no identical function in train and test).
Models: TF-IDF token 1–3-grams + class-weighted logistic regression; Hybrid = SVD(64) + Isolation-Forest and
autoencoder scores → class-weighted HistGradientBoosting.
Evaluation: repeated stratified 5-fold CV (5 × 5), thresholds chosen inside the training fold, positive-class
metrics (PR-AUC, F1, MCC), Nadeau–Bengio corrected t-test.
Reproducibility
Seeds are fixed (`random_state=42` for CV splits, fold index for models). Results in `results/` were produced
with Python 3.13, numpy 2.5, pandas 3.0, scikit-learn 1.9, scipy 1.18 on a 2-vCPU / 7 GB machine.
Small numeric differences are expected with other library versions.
Data & licences
SolidiFI-benchmark — Ghaleb & Pattabiraman, ISSTA 2020: https://github.com/DependableSystemsLab/SolidiFI-benchmark
(not redistributed here; downloaded by the script).
The synthetic data are generated by `src/data_synthetic.py` and must not be reported as empirical findings.
References
Durieux, T., Ferreira, J. F., Abreu, R., & Cruz, P. (2020). Empirical review of automated analysis tools on 47,587 Ethereum smart contracts. ICSE 2020. https://arxiv.org/abs/1910.10601
Ghaleb, A., & Pattabiraman, K. (2020). How effective are smart contract analysis tools? Evaluating smart contract static analysis tools using bug injection. ISSTA 2020. https://arxiv.org/abs/2005.11613
Liu, F. T., Ting, K. M., & Zhou, Z.-H. (2008). Isolation forest. ICDM 2008.
Nadeau, C., & Bengio, Y. (2003). Inference for the generalization error. Machine Learning, 52, 239–281.
Saito, T., & Rehmsmeier, M. (2015). The precision-recall plot is more informative than the ROC plot when evaluating binary classifiers on imbalanced datasets. PLoS ONE, 10(3), e0118432.
Zhuang, Y., et al. (2020). Smart contract vulnerability detection using graph neural network. IJCAI 2020. https://www.ijcai.org/proceedings/2020/454
License
MIT (code). Datasets keep their original licences.
