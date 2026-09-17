# Deloitte Interview Handbook 2 --- Machine Learning + Statistics

## AI / ML Engineer II \| Fundamentals + Production Reasoning

> **Goal:** Explain classical ML and statistics from first principles,
> choose models/metrics appropriately, diagnose failures, and connect
> decisions to production systems.

------------------------------------------------------------------------

# 1. ML lifecycle

``` text
Business problem → Problem formulation → Data → Validation → EDA
→ Features → Train/validation/test → Training → Evaluation → Error analysis
→ Deployment → Monitoring → Retraining
```

An Engineer-II answer should go beyond "model + accuracy."

# 2. Supervised vs unsupervised

**Supervised:** labels exist; classification and regression are common.

**Unsupervised:** no target; clustering, dimensionality reduction and
some anomaly detection approaches are common.

# 3. Classification

Examples: fraud detection, document classification, default prediction.

Common models:

-   Logistic Regression
-   Decision Tree
-   Random Forest
-   Gradient Boosting/XGBoost
-   SVM
-   neural networks

# 4. Logistic Regression

It models a probability using the sigmoid:

``` text
P(y=1|x) = 1 / (1 + e^-(wᵀx+b))
```

The probability is converted to a class using a threshold. The threshold
does not have to be 0.5.

**Production connection:** threshold choice should consider
false-positive/false-negative costs, operational review capacity and
business risk.

# 5. Linear Regression

``` text
y = wᵀx + b
```

Common losses:

``` text
MSE = average((y - ŷ)²)
MAE = average(|y - ŷ|)
RMSE = sqrt(MSE)
```

RMSE penalizes large errors more strongly; MAE is more robust to
outliers.

# 6. Decision Trees

Trees recursively split data. Classification commonly uses Gini impurity
or entropy/information gain.

Gini:

``` text
1 - Σ p_k²
```

Control overfitting with max depth, minimum samples and pruning-related
parameters.

# 7. Random Forest

Random Forest averages/aggregates many randomized decision trees,
reducing variance.

Know `n_estimators`, `max_depth`, `max_features`, and
`min_samples_leaf`.

**Production connection:** a useful nonlinear baseline for tabular data
without the operational complexity of a large neural model.

# 8. Gradient Boosting

Models are added sequentially to improve errors made by earlier models.

Strengths:

-   excellent tabular performance
-   nonlinear behavior
-   interactions

Trade-offs:

-   tuning complexity
-   training cost
-   overfitting risk
-   serving/maintenance complexity

# 9. Bias and variance

High bias → model too simple → underfitting.

High variance → model too sensitive → overfitting.

Typical diagnosis:

  Training    Validation   Interpretation
  ----------- ------------ ---------------------
  Bad         Bad          Underfitting
  Good        Good         Good generalization
  Very good   Much worse   Overfitting

# 10. Regularization

L1 adds `λ Σ|w|` and can create sparse weights. L2 adds `λ Σw²` and
shrinks weights.

Purpose: control complexity and improve generalization.

# 11. Train/validation/test

``` text
Train → fit
Validation → select/tune
Test → final estimate
```

Do not repeatedly tune against the test set.

For temporal problems, use time-aware splits such as:

``` text
Train: Jan–Sep → Validation: Oct → Test: Nov
```

# 12. Cross-validation

K-fold CV rotates validation folds and averages performance. It is
useful when data is limited and observations are suitably independent.

Do not use ordinary random K-fold blindly for time series or
grouped/dependent observations.

# 13. Classification metrics

Confusion matrix:

``` text
                 Predicted
               Positive Negative
Actual Positive    TP       FN
Actual Negative    FP       TN
```

Accuracy = `(TP+TN)/total`

Precision = `TP/(TP+FP)`

Recall = `TP/(TP+FN)`

F1 = harmonic mean of precision and recall.

**Production:** metric selection depends on class balance and business
cost.

# 14. ROC-AUC vs PR-AUC

ROC-AUC summarizes ranking across thresholds using TPR/FPR. PR-AUC
emphasizes precision/recall and can be more informative when the
positive class is rare.

Good interview answer:

> "I would select metrics based on class distribution and the relative
> business cost of false positives and false negatives."

# 15. Regression metrics

-   MAE --- interpretable, less sensitive to outliers
-   MSE --- strongly penalizes large errors
-   RMSE --- same units as target, still emphasizes large errors
-   R² --- variance explained relative to a baseline

High R² does not automatically mean a useful production model.

# 16. Feature engineering

Examples:

-   log transforms
-   ratios
-   counts
-   rolling statistics
-   categorical encoding
-   date features
-   interactions

**Production:** training and inference must use consistent
transformation logic.

# 17. Scaling

Standardization:

``` text
z = (x - mean) / std
```

Important for distance/gradient/coefficient-sensitive models such as
KNN, SVM, logistic/linear regression and neural networks. Usually less
important for trees.

# 18. Missing values

Options include deletion, mean/median, model-based methods, explicit
"unknown" categories and missingness indicators.

Understand *why* data is missing before choosing an approach.

# 19. Class imbalance

A dataset with 99.5% normal and 0.5% fraud can produce 99.5% accuracy
with a useless always-normal classifier.

Approaches:

-   class weights
-   over/undersampling
-   threshold tuning
-   anomaly detection
-   suitable metrics

Validate whether the method actually improves the real objective.

# 20. Hyperparameter tuning

Know grid search, random search and Bayesian optimization. Random search
can be efficient when only a few dimensions strongly affect performance.

Track experiments and validation results for reproducibility.

# 21. Data leakage

Leakage occurs when information unavailable at prediction time enters
features or training.

Example: predicting loan default at application time while using
post-approval repayment information.

Prevention:

-   define prediction timestamp
-   define feature availability timestamp
-   time-aware splits
-   feature lineage review

# 22. Drift

**Data drift:** `P(X)` changes.

**Concept drift:** `P(Y|X)` changes.

Monitor input distributions, prediction distributions and labeled
performance when labels arrive.

Do not automatically retrain just because a drift alert fired;
investigate impact first.

# 23. Statistics --- central tendency and spread

Mean is sensitive to outliers. Median is more robust. Variance measures
spread and standard deviation is its square root.

# 24. Probability and Bayes

``` text
P(A|B) = P(A∩B)/P(B)
P(A|B) = P(B|A)P(A)/P(B)
```

Understand the intuition, not only the formula.

# 25. Correlation vs causation

Correlation means variables move together; it does not establish that
one causes the other.

# 26. Normal distribution

Symmetric bell-shaped distribution. In a normal distribution mean,
median and mode coincide.

Know standard normal intuition: mean 0 and standard deviation 1.

# 27. Central Limit Theorem

Under appropriate conditions, the sampling distribution of a sample mean
approaches a normal distribution as sample size increases. This
underpins many inference methods.

# 28. Hypothesis testing

Start with null and alternative hypotheses. A p-value is the
probability, under the null model, of obtaining data at least as extreme
as observed according to the test statistic.

It is **not** the probability that the null hypothesis is true.

# 29. Confidence intervals

A confidence interval is a range produced by a procedure with a
specified long-run coverage under repeated sampling. It communicates
uncertainty around an estimate.

# 30. Type I and Type II errors

Type I: reject a true null → false positive.

Type II: fail to reject a false null → false negative.

Business interpretation depends on the use case.

# 31. Statistical vs practical significance

A tiny improvement may be statistically significant with huge data but
operationally irrelevant. Ask both:

``` text
Is the effect statistically reliable?
Is it practically meaningful?
```

# 32. Sampling

Know random, stratified and systematic sampling, plus sampling bias.
Stratification can preserve class proportions in splits for imbalanced
classification.

# 33. PCA

PCA finds directions of high variance and projects data to fewer
dimensions.

Uses: dimensionality reduction, visualization and sometimes noise
reduction.

Trade-off: less dimensionality can mean less interpretability.

# 34. Clustering

K-Means iteratively assigns points to centroids and recomputes
centroids. It is sensitive to scaling, initialization, K and outliers.

# 35. Anomaly detection

Examples: Isolation Forest, One-Class SVM, statistical thresholds and
autoencoders.

Production pattern:

``` text
Event → features → anomaly score → threshold → alert/review
```

Thresholds should reflect investigation capacity and cost.

# 36. Explainability

Know feature importance, permutation importance and SHAP at a conceptual
level. Explainability is not causality. In financial/regulated settings,
explainability can also support governance and review.

# 37. Model selection framework

Never answer "XGBoost is best." Reason through:

``` text
Data type → sample size → nonlinearity → interpretability
→ latency → training cost → business/governance constraints
```

# 38. Production ML architecture

``` text
Data sources
 ↓
Data validation
 ↓
Feature pipeline
 ↓
Training
 ↓
Experiment tracking / registry
 ↓
Deployment
 ↓
Inference
 ↓
Monitoring
 ↓
Retraining
```

Monitor system metrics, data quality/drift, model quality and business
KPIs.

# 39. Batch vs online inference

**Batch:** periodic scoring of large datasets.

**Online:** request → prediction → response.

Online inference has stricter latency/availability/concurrency
requirements.

# 40. Model monitoring

Monitor:

-   latency
-   throughput
-   errors
-   CPU/GPU/memory
-   feature missingness
-   schema changes
-   drift
-   model performance when labels arrive
-   prediction distribution

# 41. Retraining strategy

Retraining may be scheduled, drift-triggered, performance-triggered or
data-triggered.

A robust flow is:

``` text
Drift → verify → assess business/model impact → diagnose → retrain if justified
```

# 42. High-yield interview questions

### Overfitting?

The model captures training-specific patterns and generalizes poorly.
Mitigate with regularization, simpler models, more data, early stopping,
CV and feature selection.

### Precision vs recall?

Precision asks how many predicted positives are correct; recall asks how
many actual positives are found.

### Why accuracy can mislead?

Class imbalance.

### How prevent leakage?

Define prediction and feature availability times.

### How handle drift?

Monitor distributions and performance, investigate impact, then decide
intervention/retraining.

### Why trees don't need scaling?

Their splits are based on feature thresholds/order, not distances or
coefficient magnitudes.

### What is regularization?

Penalty/constraint that controls model complexity and improves
generalization.

# 43. Case study --- financial document classification

``` text
PDF → OCR/text → cleaning → BERT representation → classifier → class
```

Evaluate accuracy plus per-class precision/recall/F1 and confusion
matrix where appropriate.

Production monitoring: OCR failures, class distribution, confidence
distribution, latency, per-class quality and drift across
markets/templates.

# 44. Case study --- fraud/risk

``` text
Transactions → SQL features → validation → model → risk score → threshold → action/review
```

Key concepts: imbalance, precision/recall, thresholding, calibration,
temporal validation, leakage, drift, explainability and auditability.

# 45. Final answer framework

For theoretical ML questions:

``` text
Definition → intuition → math if needed → advantages → limitations
→ when to use → production consideration
```

That turns textbook knowledge into an Engineer-II answer.
