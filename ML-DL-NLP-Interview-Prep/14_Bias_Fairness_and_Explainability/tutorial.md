# Bias, Fairness & Explainability

Brief primer — increasingly asked in senior/lead interviews as a judgment check, not usually a deep technical deep-dive.

## Where bias comes from

- **Sampling bias**: training data doesn't represent the real population (e.g. a resume-screening model trained mostly on past hires from one demographic).
- **Historical/label bias**: the label itself encodes past discrimination (e.g. "was promoted" as a target, when promotion decisions were historically biased).
- **Proxy discrimination**: a protected attribute (gender, race) isn't in the feature set, but a correlated proxy is — zip code, name, or school can carry nearly the same signal. Removing the protected attribute directly does **not** make a model fair; this is one of the most common interview traps on this topic.

## Fairness metrics (and why you can't have them all)

- **Demographic parity**: equal positive-prediction rate across groups.
- **Equalized odds**: equal true-positive and false-positive rates across groups.
- **Predictive parity**: equal precision across groups.

When base rates differ between groups, it's mathematically proven you generally cannot satisfy demographic parity and equalized odds simultaneously (Kleinberg et al.) — picking a fairness definition is a deliberate, documented business/ethics tradeoff, not a default setting.

## Explainability

- **SHAP** (Shapley values): a game-theoretic way to attribute a prediction's outcome fairly across its input features, with guaranteed consistency properties. Used both globally (which features matter most overall) and locally (why did *this* applicant get denied).
- **LIME**: perturbs an input locally and fits a simple, interpretable surrogate model around one prediction — cheaper than SHAP but without the same consistency guarantees.
- **Interpretable-by-design models** (linear/logistic regression, shallow trees, GAMs) are sometimes chosen over a more accurate black-box model specifically because regulated domains (credit, healthcare, criminal justice) require exact, auditable explanations rather than approximate post-hoc ones — e.g. US lending law (ECOA/Regulation B) requires specific, actionable adverse-action reasons for a loan denial, which a raw deep-learning score can't provide on its own.

## A real scenario worth knowing cold

A bank's credit model doesn't use gender or race as a feature and still shows a large approval-rate gap between demographic groups. The fix isn't "we didn't use the protected attribute, so we're compliant" — it's auditing outcome disparities directly (not just feature inclusion), checking for proxy variables, and using SHAP/LIME to inspect whether the same features are driving denials disproportionately for one group.

## Quick self-check

Can you explain, in one sentence each, why "we removed the protected attribute" doesn't guarantee fairness, and why SHAP and LIME can give different explanations for the same prediction? If yes, this topic is covered for interview purposes.
