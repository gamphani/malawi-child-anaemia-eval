# Evaluation Protocol (written before data access)

## 1. Question
Predict childhood anaemia (Hb < 11.0 g/dL, altitude-adjusted) in children
aged 6-59 months in Malawi, from information obtainable without a blood test.

## 2. Data
Malawi DHS 2024, Children's Recode. Unit of analysis: [child / mother / cluster?]

## 3. Target
Binary: 1 = anaemic, 0 = not. Children with missing Hb are [dropped / handled how?
Why? What bias might dropping introduce?]

## 4. Allowed features
[List the feature groups you will allow, e.g. child age and sex, mother's
education, wealth, residence, region, water/sanitation, ...]

## 5. Excluded features (leakage audit)
- Haemoglobin value and anything derived from it
- Malaria test result
- [Add two more and say why each is excluded]
- Open question: is anthropometry (height/weight) allowed? [Decide and justify,
  considering what a community health worker could realistically measure.]

## 6. Splitting
Grouped by cluster ID. Proportions: [train / validation / test?].
Test set is touched exactly once, at the very end.
Random seed: [pick one and fix it].

## 7. Baselines
- B0: predict the training-set prevalence for everyone
- B1: logistic regression using child age only

## 8. Metrics (fixed in advance)
- Primary: AUROC on the test set
- Secondary: Brier score, calibration slope and intercept, reliability plot
- Uncertainty: 95% CIs from a cluster bootstrap, [how many resamples?]

## 9. Success criterion
The final model "beats" the baseline only if [state the exact rule, e.g. the 95%
CI of the AUROC difference excludes zero].

## 10. Known limitations
Survey weights are [used / not used] because [why]. State what that means for
how far the results generalise.