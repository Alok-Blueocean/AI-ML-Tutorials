# MLOps and Model Deployment — Exercises

Ordered easy to hard. Mix of hands-on tasks and conceptual questions.

1. **Track an experiment.** Take any small model (e.g., scikit-learn classifier on a tabular dataset) and log parameters, metrics, and the trained model artifact to MLflow. Compare 3 runs with different hyperparameters in the MLflow UI.

2. **Version data with DVC.** Set up a small project where a CSV dataset is tracked with DVC instead of Git, push it to a remote (even local-disk remote is fine), modify the data, and show `dvc repro` re-running only the affected pipeline stage.

3. **Conceptual: why k-fold CV can lie.** Explain why standard random k-fold cross-validation is inappropriate for time-ordered production data (e.g., daily transaction data), and what you'd use instead (e.g., time-based / rolling-origin splits).

4. **Containerize a model.** Wrap a trained model in a REST API (FastAPI/Flask) and Dockerize it. Confirm it runs identically on a different machine, and measure cold-start latency.

5. **Simulate data drift.** Take a trained classifier and a test set; artificially shift one feature's distribution (e.g., add a constant offset or resample from a different range) and measure how accuracy degrades. Then implement a simple drift detector (e.g., Kolmogorov-Smirnov test comparing training vs "production" feature distributions) that flags the shift.

6. **Conceptual: data drift vs concept drift.** Describe a real scenario for each: one where input distribution changes but the input-output relationship is stable (data drift), and one where the input-output relationship itself changes even with stable inputs (concept drift). Explain why detecting the second one usually requires labels.

7. **Build a CI gate for model quality.** Set up a CI pipeline (GitHub Actions or similar) that fails a pull request if a newly trained model's accuracy on a fixed holdout set is worse than the currently deployed model's recorded metric.

8. **Shadow deployment simulation.** Deploy two versions of a model behind one API: log predictions from both on every real request, but only return the "production" model's prediction to the caller. Compare their outputs offline afterward.

9. **Design an A/B test.** For a hypothetical new recommendation model, define: the metric you'd use as the primary success criterion, the minimum detectable effect, an estimate of the sample size/duration needed, and how you'd guard against peeking-related false positives.

10. **Quantize a model.** Take a trained neural network (e.g., a small PyTorch model) and apply post-training quantization (FP32 -> INT8). Measure the change in model size, inference latency, and accuracy.

11. **Export to ONNX.** Export the same model to ONNX format and run inference with ONNX Runtime. Compare latency against the native framework's inference.

12. **Design a feature store schema.** For a fraud-detection use case, sketch (on paper or in a doc) which features would live in an offline store vs an online store, and how you'd guarantee both compute "transactions in last 10 minutes" identically.

13. **Build a monitoring dashboard.** Using Evidently (or a similar open-source tool) or a hand-rolled script, build a report that tracks prediction distribution and top-feature drift over time for a deployed (or simulated) model, and define alert thresholds.

14. **Conceptual: rollback trigger design.** Design 3 concrete automatic rollback triggers for a production model (not "accuracy dropped" — you don't have labels in real time) using only signals available within minutes of deployment.

15. **End-to-end pipeline critique.** Given a described ML system that trains monthly on a fixed schedule with no drift monitoring and no A/B testing before full rollout, propose a prioritized set of MLOps improvements and justify the order given limited engineering time.
