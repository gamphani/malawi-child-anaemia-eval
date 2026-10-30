# Evaluation Protocol: Childhood Anaemia Risk Prediction, Malawi DHS 2024

**Version:** 1.0
**Date:** 2026-10-30
**Status:** Written before data access. Changes after this date are recorded in Section 13 only.

---

## 1. Objective

Predict whether a child aged 6-59 months in Malawi is anaemic (haemoglobin < 11.0 g/dL, altitude-adjusted), using only information obtainable by a community health worker (CHW) without a blood test. Intended use: triage, ranking children by risk to prioritise haemoglobin testing and referral.

## 2. Data

- Source: Malawi Demographic and Health Survey 2024 (MDHS 2024), Children's Recode.
- Unit of analysis: the child (one row per child).
- Eligibility: children aged 6-59 months with a valid haemoglobin measurement.
- Raw data are stored locally in `data/` and are not committed to version control.
- All variable names used in this protocol are verified against the official data dictionary before use. Corrections are recorded in Section 13.

## 3. Target

- `y = 1` if altitude-adjusted haemoglobin < 11.0 g/dL, otherwise `y = 0`.
- `y` is derived from the continuous haemoglobin variable. The DHS-provided anaemia-level variable is used only to check the derivation and is never used as a feature.
- Children with missing haemoglobin are excluded from modelling.
- Before exclusion, the number of children with missing haemoglobin is tabulated, and their age, region, wealth quintile and residence are compared with those of retained children.

## 4. Allowed features

| Group | Features |
|---|---|
| Child | age in months, sex, birth order, currently breastfed |
| Recent illness (caregiver report) | fever in last 2 weeks, diarrhoea in last 2 weeks, cough |
| Preventive care (caregiver report) | slept under a bed net last night, received iron supplements, received vitamin A |
| Mother | age, highest education level |
| Household | wealth quintile, household size, main drinking-water source, toilet type, main cooking fuel |
| Geography | residence type (urban/rural), region |

Any feature added or removed is recorded in Section 13.

## 5. Excluded features

The following are never used as model inputs:

1. Haemoglobin value (any version) and anaemia-level categories derived from it.
2. Malaria test results (rapid test or microscopy).
3. Mother's measured haemoglobin or anaemia status.
4. Cluster, household and line identifiers, interview dates and other administrative variables.
5. Survey weight and design variables (used only for the sensitivity analysis in Section 12).
6. Anthropometric variables (height, weight, stunting, wasting and underweight z-scores) in the primary model.

A secondary exploratory model adds anthropometric variables to the primary feature set. It is reported separately and labelled as requiring additional measurement.

If any single feature yields a test AUROC above 0.85 on its own, or the full model's performance is implausibly high, modelling stops and the pipeline is audited for leakage.

## 6. Data splitting

- Splits are grouped by DHS cluster (primary sampling unit). All children from a cluster are assigned to the same split.
- 80% of clusters form the development set; 20% form the test set. The split is drawn once.
- Model selection and tuning within the development set use grouped 5-fold cross-validation, grouped by cluster.
- The test set is used once, after all modelling decisions are frozen.
- Random seed: 20261030, used in every random operation.
- Sample size and anaemia prevalence are reported for each split. The split is not redrawn.
- The assignment of clusters to splits is saved to disk.

## 7. Preprocessing

All data-dependent steps (imputation, scaling, encoding, feature selection) are fitted on training data only and applied unchanged to validation and test data. These steps are implemented inside a scikit-learn `Pipeline` so that they are refitted within each cross-validation fold.

## 8. Models

| ID | Model |
|---|---|
| B0 | Predicts the training-set anaemia prevalence for every child |
| B1 | Logistic regression on age band only (6-11, 12-23, 24-35, 36-47, 48-59 months) |
| M1 | Regularised logistic regression on all allowed features |
| M2 | Gradient-boosted trees (`HistGradientBoostingClassifier`) on all allowed features |

M1 and M2 are compared on development-set cross-validated AUROC. The better of the two is the final model.

**Tuning.** M1: regularisation strength C chosen from {0.01, 0.1, 1, 10, 100}.
M2: 30 random configurations over learning rate, maximum leaf nodes, minimum
samples per leaf and L2 regularisation. Both are tuned using grouped 5-fold
cross-validation AUROC on the development set only.

**Model choice.** M2 is selected as the final model only if its mean
cross-validated AUROC exceeds that of M1 by at least 0.01. Otherwise M1 is the
final model.

## 9. Metrics

| Role | Metric |
|---|---|
| Primary | AUROC on the test set |
| Secondary | Brier score |
| Secondary | Calibration slope and intercept, from `logit(P(y=1)) = a + b * logit(p_hat)` |
| Secondary | Reliability plot |
| Secondary | Sensitivity, specificity and positive predictive value at pre-specified risk thresholds |

Accuracy is not reported as a performance metric.

Recalibration (Platt scaling or isotonic regression) may be applied to the final model. It is fitted only on cross-validated predictions from the development set. Calibration is reported before and after recalibration.

**Threshold metrics.** Sensitivity, specificity and positive predictive value
are reported when the highest-risk 20%, 30% and 50% of children (ranked by
predicted risk) are flagged.

## 10. Confidence intervals

- 95% percentile intervals from a cluster bootstrap of the test set, 2,000 resamples.
- Each resample draws test clusters with replacement until the original number of clusters is reached, retains all children in each drawn cluster, and recomputes every metric.
- Intervals are reported for AUROC, Brier score, calibration slope and calibration intercept. Point estimates come from the original test set.
- Model comparisons use paired resampling: the same resampled clusters for both models, with the difference in each metric recorded per resample.

## 11. Success criterion

The final model beats the baseline only if both conditions hold on the test set:

1. The 95% paired-bootstrap confidence interval of AUROC(final) − AUROC(B1) lies entirely above zero.
2. The point estimate of AUROC(final) − AUROC(B1) is at least 0.02.

The comparison of the final model with B1 is the only primary comparison. The comparisons M2 vs M1 and exploratory-anthropometry model vs primary model are reported descriptively. If the criterion is not met, the result is reported as: the model did not demonstrably beat the baseline.

## 12. Sensitivity analyses and stated limitations

**Sensitivity analyses**

- Survey weights: primary training and metrics are unweighted. A survey-weighted test-set AUROC is reported as a sensitivity analysis.
- AUROC is reported by region and by urban/rural residence. These subgroup results are exploratory.

**Limitations stated in the README**

1. Evaluation uses held-out clusters from a single survey and is not external validation.
2. Exclusion of children with missing haemoglobin may introduce selection bias.
3. The data are cross-sectional. Associations are predictive, not causal.
4. The model predicts measured haemoglobin status, not the cause of anaemia.
5. Unweighted estimates describe the analysed sample, not the national child population.

## 13. Deviation log

| Date | Section | Change | Reason |
|---|---|---|---|
| 2026-10-30 | All | Initial version | |
