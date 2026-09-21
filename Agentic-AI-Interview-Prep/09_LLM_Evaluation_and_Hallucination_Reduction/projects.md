# LLM Evaluation and Hallucination Reduction — Projects

## Small: LLM-as-Judge Bias Audit

Build a pairwise LLM-as-judge harness over 25-30 answer pairs for a task of your choice, then run three controlled experiments: swap answer order to measure position bias, pad one answer's length without adding content to measure verbosity bias, and use the same model family as both generator and judge vs. a different family to measure self-preference bias. Produce a short report quantifying how often each bias flips a verdict. This proves you understand judge failure modes empirically, not just by name, which is exactly what separates a definitions-level answer from a senior one in an interview.

## Medium: CI-Gated Regression Pipeline for a Prompt-Based Feature

Build a golden dataset of 50-100 examples for a concrete task (support Q&A, summarization, or classification), wire up an LLM judge with a rubric and JSON output contract, and write a `regression_gate.py`-style script that runs baseline vs. candidate prompt versions against the golden set, computes a paired delta, and exits non-zero on regression beyond a threshold. Wrap it in a GitHub Actions workflow that blocks a PR merge on failure. Calibrate the judge against 30 hand-labeled examples (Spearman's rho) before trusting it in the gate. This proves you can operationalize evaluation as an enforced CI check, not a one-off notebook metric.

## Medium: Hallucination-Reduction Comparison Harness

Build a small RAG pipeline over 20-30 documents, then implement and compare three hallucination-reduction techniques against the same 20-question eval set: baseline RAG with no additional check, self-consistency majority voting (5 samples), and a retrieval-then-verify pass that fact-checks each generated claim against retrieved context. Measure hallucination rate, cost, and latency for each approach and report the tradeoff curve. This proves you can reason quantitatively about cost/quality tradeoffs across competing mitigation techniques rather than picking one dogmatically.

## Large: Production-Style Quality Gate with Dashboard and Panel-of-Judges

Build a full release-readiness system: a frozen, versioned golden dataset; a panel of 3 diverse-model judges (PoLL-style) with majority-vote aggregation and per-dimension scoring (correctness, relevance, format); absolute gates (hard block on hallucination rate and format-validity) plus a relative gate against a stored production baseline; and a lightweight dashboard (a simple web page or notebook is fine) showing headline pass/fail status, per-dimension breakdown, and a 30-day trend line. Wire the gate into a CI pipeline so a failing run actually blocks a mock "deploy" step. This proves you can design the full evaluation-to-release pipeline a senior LLMOps interview expects, end to end, not just one piece of it in isolation.
