# LLM Evaluation and Hallucination Reduction — Condensed Study Notes

## LLM-as-a-Judge: The Core Pattern

- Use an LLM (often a stronger or differently-sourced model than the one under test) to score or compare free-text outputs where there is no single correct string to diff against — support answers, summaries, RAG responses, agent plans.

- A weak judge prompt ("rate this 1-5") produces inconsistent, unauditable scores. A strong judge prompt has four ingredients: an explicit rubric (what each score level concretely means), chain-of-thought-first reasoning (force the model to reason before it commits to a number), temperature 0 (reproducibility), and separated scoring dimensions (correctness, relevance, format scored independently, not blended into one number).

```
judge_prompt = f"""
Score the CANDIDATE ANSWER on CORRECTNESS ONLY (ignore style/tone).
Rubric: 5=fully correct, 4=correct but incomplete, 3=partially correct,
2=mostly incorrect, 1=wrong/fabricated.
Reason step by step first, then output JSON:
{{"reasoning": "...", "score": <1-5>, "rubric_clause": "..."}}
QUESTION: {question}
CANDIDATE ANSWER: {answer}
"""
# always temperature=0 for judge calls
```

- Real-world example: a support-bot team ships five prompt-template changes a week and cannot afford to have humans re-review 500 eval examples per change, so an LLM judge that correlates well enough with human raters turns a two-day manual review into a two-minute automated one — the entire reason fast iteration on LLM systems is possible at all.

## Known Judge Biases and Mitigations

| Bias | What happens | Mitigation |
|---|---|---|
| **Position bias** | In pairwise (A vs. B) comparisons, the judge favors whichever answer appears first (or second), independent of content — reordering alone can flip a verdict | Randomize/counter-balance presentation order, average the score across both orderings, or prefer pointwise (direct) scoring over pairwise comparison |
| **Verbosity bias** (length bias) | Longer, more elaborate answers score higher regardless of actual quality | Add an explicit rubric clause telling the judge length is not a criterion; hold response length roughly constant when comparing two variants |
| **Self-preference bias** (self-enhancement bias) | A judge rates outputs resembling its own model family's style more favorably, plausibly because such text gets lower perplexity from the judge | Use a different model family as judge than the system under test; never let a model grade its own output in a release-gating check |

- Zheng et al. ("Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena," NeurIPS 2023) found a strong LLM judge can reach 80%+ agreement with human preference — comparable to human-human agreement — while also formally naming position, verbosity, and self-enhancement bias as persistent limitations even in strong judges.

- Wang et al. ("Large Language Models are not Fair Evaluators," ACL 2024) showed position bias can be severe enough to flip a benchmark's outcome: simply reordering candidate responses let a weaker model "beat" a stronger one on the majority of tested queries when GPT-4 was the judge.

- Real-world example: a team notices their judge prefers GPT-4-generated answers over Claude-generated answers by a suspiciously consistent margin across many unrelated prompts — swapping in a judge from a third model family (not GPT, not Claude) shows the gap nearly disappears, confirming self-preference bias rather than a genuine quality difference.

## Panels of Judges (PoLL) and Judge Ensembling

- A single judge model is a single point of failure with a single, correlated set of blind spots — recalibration adjusts the threshold, it doesn't remove the underlying skew.

- Verga et al. ("Replacing Judges with Juries," 2024) introduced PoLL (Panel of LLM evaluators): several smaller, diverse-family judges voting/averaging outperforms one large frontier-model judge on agreement with humans, has less intra-model bias (no single family's quirk dominates), and costs over 7x less.

- Real-world example: a release gate that used a single GPT-4-class judge for months quietly inherited that judge's self-preference bias against a competitor-model-generated candidate; switching to a 3-model panel (spanning different labs) and voting removed the systematic skew without materially increasing latency, since the three judge calls run in parallel.

- Use ensembling for high-stakes, release-blocking gates. Skip it for cheap, high-volume, exploratory scoring during prompt iteration — a single well-calibrated judge is faster and cheaper there, and the marginal bias reduction isn't worth the extra cost on every loop iteration.

## Calibrating a Judge Against Human Raters

- A judge is not trustworthy just because it produces a number — it must be shown to agree with human judgment on a shared sample before it's allowed to gate anything.

```python
from scipy import stats

human_scores = [5, 3, 4, 2, 5, 1, 4, 3, 5, 2]
judge_scores = [5, 3, 4, 3, 4, 1, 4, 2, 5, 2]
rho, p_value = stats.spearmanr(human_scores, judge_scores)
# rho >= 0.75 -> safe to gate CI/CD on; rho < 0.75 -> refine rubric, don't deploy
```

- Use Spearman's rank correlation for ordinal/continuous Likert scores (it only asks whether the two raters agree on relative ranking, not exact numeric alignment). Use Cohen's kappa for binary/categorical outputs (pass/fail), since raw agreement and Spearman's ρ can both be inflated by chance agreement on a small number of categories.

- Real-world example: a judge that was calibrated at ρ=0.81 six months ago silently degrades to ρ=0.68 after the provider updates the underlying model behind the same API name — nobody notices until a monthly recalibration job (run against a fixed 80-example human-labeled set) flags the drop and freezes the judge from gating further releases until the rubric is re-anchored.

## Regression Testing: Golden Datasets and CI-Gated Eval

- A "golden dataset" is a frozen, versioned set of input examples (ideally with reference answers or acceptance criteria) used identically across every comparison — the same discipline as pinning a dataset for reproducible model training, applied to evaluation.

- Any prompt, model, or RAG-corpus change should re-run against the golden set before merge, with the run gated in CI the same way a code change is gated by unit tests.

```python
# regression_gate.py — simplified CI gate comparing two prompt versions
def run_regression(golden_set, baseline_prompt, candidate_prompt, judge):
    baseline_scores = [judge(ex, baseline_prompt) for ex in golden_set]
    candidate_scores = [judge(ex, candidate_prompt) for ex in golden_set]
    delta = mean(candidate_scores) - mean(baseline_scores)
    if delta < -0.02:            # candidate regressed correctness by >2%
        sys.exit(1)              # fail the build, block the merge
    print(f"delta={delta:+.3f} — safe to promote")
```

- Real-world example: an engineer tweaks a system prompt's wording to fix one customer complaint, and the CI gate catches that the same change silently broke 3 of 20 golden-set examples that depended on the original phrasing — a defect that would have shipped invisibly without a frozen regression set to diff against.

- Freeze the eval set explicitly (never let it silently drift between two comparisons) and version it alongside the prompt/model version, so "eval_set_v7" always means the same 200 examples six months from now.

## Quality Benchmarking: Task-Specific Metrics vs. General Benchmarks

- General public benchmarks (MMLU, HumanEval, GSM8K, HellaSwag) measure broad capability and are useful for picking a base model, but they rarely predict how well a model performs on your specific task, tone, and domain — a model that tops a leaderboard can still fail your customer-support use case.

- Task-specific metrics (RAG faithfulness, format-validity rate, refund-policy accuracy, refusal-rate on off-topic questions) are what should actually gate a release, because they measure the thing your product is graded on.

- Real-world example: a legal-summarization startup picks its base model using MMLU and HumanEval leaderboard rank, then discovers its actual failure mode — citing the wrong statute section — isn't measured by either benchmark at all; they build a 200-example task-specific "citation accuracy" eval set and use that, not the leaderboard, to decide which model version to ship.

## Benchmark Contamination Risk

- Public benchmark data can leak into a model's pretraining corpus (scraped from the web, GitHub, or benchmark leaderboard writeups), inflating reported scores without a corresponding real capability gain.

- A 2026 systematic review across 55 studies found every popular static benchmark contaminated to some degree, with identical model weights scoring 10-20 percentage points apart depending on the evaluation harness, and concluded no single detection method (string-matching, likelihood-based membership inference, LLM-prompted detection, benchmark auditing) is reliable across every contamination tier.

- Mitigations: prefer your own held-out, task-specific eval set that a vendor's pretraining run could not plausibly have seen; treat public leaderboard numbers as a rough capability signal, not a promise of production performance; watch for benchmarks with a known "canary string" or contamination-resistant design; re-evaluate periodically since new model releases can silently be trained on last year's benchmark.

- Real-world example: a team picks a base model partly because it leads a public reasoning benchmark, then finds its own task-specific eval set (built from real production queries) tells a completely different story — the benchmark-leading model performs no better than a cheaper alternative on the task that actually matters, consistent with contamination inflating the public number.

## Hallucination Reduction: RAG Grounding and Citation Requirements

- Ground generation in retrieved, source-of-truth text and instruct the model to answer only from that context, saying "I don't know" when the context doesn't support an answer.

- Requiring inline citations (quote or point to the specific retrieved chunk backing each claim) makes ungrounded claims visible and auditable, and gives the model an incentive structure that discourages inventing unsupported content.

- Real-world example: a medical-info chatbot adds a hard rule — never answer a dosage question unless the answer is a direct quote from a retrieved, approved source document — because a fluent but wrong dosage is a liability, not just an inconvenience.

## Self-Consistency and Majority Voting

- Sample multiple independent reasoning paths (via nonzero temperature) for the same question, then take the majority answer instead of trusting a single greedy decode — Wang et al. ("Self-Consistency Improves Chain of Thought Reasoning," 2022) showed this boosts accuracy on reasoning benchmarks by double-digit percentage points in some cases (e.g., +17.9% on GSM8K) with no extra training.

```python
from collections import Counter

def self_consistency_answer(question, llm, n_samples=5):
    answers = [llm.generate(question, temperature=0.7) for _ in range(n_samples)]
    final_answers = [extract_answer(a) for a in answers]
    majority, count = Counter(final_answers).most_common(1)[0]
    confidence = count / n_samples          # cheap proxy for how "sure" the ensemble is
    return majority, confidence
```

- Real-world example: a math-tutoring feature samples 5 independent solution paths per question and takes the majority final answer instead of the first generation — a wrong answer that appears in only 1 of 5 samples gets outvoted, catching a class of arithmetic slips a single greedy decode would have shipped silently.

- Tradeoff: 5x the inference cost and latency for the majority-vote call. Reserve it for high-stakes or high-ambiguity questions, not every request.

## Confidence Calibration

- A model's stated confidence (or a computed probability) should track its actual correctness rate — a well-calibrated model that says "90% confident" should be right about 90% of the time across many such answers.

- Kadavath et al. ("Language Models (Mostly) Know What They Know," 2022) showed larger models are reasonably well-calibrated on multiple-choice framings and can self-evaluate the probability their own free-text answer is correct ("P(True)") with useful, scaling accuracy.

- Real-world example: a fraud-triage assistant routes any answer where the model's self-reported confidence falls below a set threshold to a human reviewer instead of auto-resolving it — this only works because the confidence score was validated (via a held-out calibration set) to actually track accuracy, not just calibrated-sounding language.

## Retrieval-then-Verify Patterns

- Generate first, then independently retrieve evidence and check each claim in the draft against it, revising or removing unsupported claims — decouples "write something plausible" from "confirm it's actually true," which is a different failure surface than grounding-at-generation-time alone.

- RARR (Gao et al., 2023) implements exactly this: post-hoc research and revision of any generated text to fix unsupported claims while preserving the rest of the output. Chain-of-Verification (Dhuliawala et al., 2023) has the model draft an answer, generate independent verification questions, answer them separately (so the answers aren't anchored to the original draft), then produce a final revised response.

```
draft = llm.generate(question)
claims = extract_claims(draft)
verification_questions = [llm.generate(f"Verify: {c}") for c in claims]
verified_answers = [llm.generate(q, context=None) for q in verification_questions]  # independent, unbiased by draft
final = llm.revise(draft, verified_answers)
```

- Real-world example: a research-assistant feature drafts a summary citing three statistics, then runs a separate verification pass that independently re-derives each statistic from the source documents — one of the three turns out to be a plausible-sounding but fabricated number, and the verification step catches and removes it before the summary reaches the user.

## Guardrail Implementation: Structured-Output Validation

- Constrain the model's output to a schema (JSON schema, Pydantic model, regex) and validate every response against it before it reaches downstream code or a user — malformed output should be a hard block, not a soft warning.

```python
from pydantic import BaseModel, ValidationError

class SupportResponse(BaseModel):
    answer: str
    confidence: float
    source_chunk_ids: list[str]

def validate_output(raw_json: str) -> SupportResponse | None:
    try:
        return SupportResponse.model_validate_json(raw_json)
    except ValidationError:
        return None   # trigger retry or fallback, never pass malformed output downstream
```

- Real-world example: an order-lookup agent's tool-calling output must always include a valid order ID matching a known regex; a guardrail rejects and retries (bounded to 2 attempts, then escalates to human) any response where the model hallucinates a plausible-looking but non-existent order ID format.

## Guardrail Implementation: Fact-Checking Layers

- A post-generation fact-checking layer (often a smaller/cheaper judge or a deterministic rule) scores whether each claim in the output is supported by retrieved context, independent of the generation step — this is the "faithfulness" metric in RAG evaluation, distinct from "did the answer address the question" (relevancy).

- Real-world example: a customer-facing summarizer runs a faithfulness check (does every sentence in the summary trace back to a retrieved source chunk) as a hard gate before display; a summary that references a policy detail not present in any retrieved chunk is blocked and regenerated rather than shown, even though the summary read fluently and confidently.

## Quick Gotchas Worth Naming in an Interview

- LLM-as-judge is a measurement instrument with known systematic error, not an oracle — treat it like you'd treat any biased human reviewer: characterize the bias, build guards, and validate before trusting it unattended.

- A composite judge score can look healthy while one dimension (say, correctness) is badly broken and another (format) is compensating for it in the average — always check per-dimension thresholds, not just the composite.

- Statistical significance (p < 0.05 on a paired test) is necessary but not sufficient for promoting a prompt/model change — pair it with a practical-significance floor (e.g., minimum 3% lift), or you'll promote changes that are "real" but operationally meaningless.

- Self-consistency and judge ensembling both trade cost/latency for reliability — neither is free, and neither belongs in every code path; reserve them for where being wrong is expensive.

- A model that passes benchmark leaderboards can still fail your specific task; a model that fails leaderboards can still be the right choice for your task. Task-specific eval beats general benchmark rank for a production ship decision, every time.

- Grounding (RAG) reduces hallucination but does not eliminate it — a model can still ignore correctly retrieved context and answer from its own memorized knowledge instead, which is exactly what a faithfulness check is designed to catch.

- "The judge said it's fine" is not the same as "a human confirmed the judge is calibrated." Recalibrate on a fixed cadence; provider-side silent model updates behind the same API name can quietly break a judge's calibration without any code change on your end.
