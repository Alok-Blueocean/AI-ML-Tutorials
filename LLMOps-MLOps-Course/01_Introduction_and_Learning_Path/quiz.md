# Module 01 — Quiz

15 questions covering the terminology, the DevOps/MLOps/LLMOps distinction, the Sculley et al. argument, the course roadmap rationale, and the reference architecture. A mix of multiple-choice and short-answer. Answer key is at the end — try to answer everything first without looking.

---

**Q1 (Multiple Choice).** Which of the following best describes the relationship between LLMOps and MLOps as this course frames it?

A. LLMOps replaces MLOps entirely once a system uses an LLM.
B. LLMOps and MLOps are unrelated, parallel disciplines that happen to share some tools.
C. LLMOps is an extension layer on top of MLOps — it reuses MLOps's versioning/CI-CD/monitoring foundation and adds LLM-specific concerns.
D. MLOps is a subset of LLMOps that only applies to classical, non-generative models.

---

**Q2 (Short Answer).** Name the three things MLOps adds on top of standard DevOps practices, and explain in one sentence *why* each is necessary (i.e., what would go wrong without it).

---

**Q3 (Multiple Choice).** According to Sculley et al.'s *Hidden Technical Debt in Machine Learning Systems*, what does "CACE" stand for, and what problem does it describe?

A. "Code And Configuration Entropy" — the tendency for config files to grow unmanageably over time.
B. "Changing Anything Changes Everything" — because ML systems learn statistical relationships from data, changing one input, feature, or hyperparameter can silently change the whole system's behavior.
C. "Continuous Automated Compliance Enforcement" — an audit requirement for regulated ML systems.
D. "Cache Alignment Cost Estimation" — a performance-tuning concept for model serving.

---

**Q4 (Short Answer).** A teammate says: "Unit tests passed, code review passed, so the model is fine to deploy." Explain, in 2–3 sentences, why this reasoning is insufficient specifically for ML systems (as opposed to ordinary software).

---

**Q5 (Multiple Choice).** Which of the following is a genuinely new failure mode that LLMOps must account for, with no direct equivalent in classical MLOps?

A. Data drift.
B. Hallucination — a fluent, confident, well-formatted, entirely wrong answer.
C. Model versioning.
D. Continuous Training.

---

**Q6 (Short Answer).** Define Continuous Training (CT) in one sentence, and explain why traditional DevOps has no equivalent concept.

---

**Q7 (Multiple Choice).** A team calls a hosted LLM API with a fixed prompt, no fine-tuning, no RAG, and no agentic behavior. According to this module, do they have an LLMOps surface to manage?

A. No — without training or fine-tuning, there is no MLOps or LLMOps concern at all.
B. No — LLMOps only applies once RAG or agentic tool-calling is involved.
C. Yes — they still have a versioned prompt artifact, non-deterministic open-ended output, real cost/latency dynamics, and hallucination risk.
D. Yes, but only if the API provider changes the underlying model version.

---

**Q8 (Short Answer).** What is the "governance gap" as described in this module, and name one concrete production question a team without it cannot cheaply answer.

---

**Q9 (Multiple Choice).** In the course's 26-module roadmap, why are the evaluation methodology modules (M09–M12) sequenced *before* RAG and agentic systems (M15–M18)?

A. Evaluation is alphabetically prior to retrieval in most textbooks.
B. Without evaluation discipline in place, teams building elaborate RAG/agent systems cannot tell whether a change made the system better or worse.
C. RAG and agent frameworks require the MLflow tooling taught in M13, which itself requires M09–M12.
D. There is no specific reason; the modules could be reordered without consequence.

---

**Q10 (Short Answer).** Explain why security/governance (M23) and cost optimization (M24) are taught near the *end* of the course, while the tutorial simultaneously argues they should NOT be deferred to the end in a real project's *build* order. Are these two claims in tension? Why or why not?

---

**Q11 (Multiple Choice).** In the full reference architecture (`architecture.md` Section 1), what closes the loop back to the top of the diagram (the CI pipeline), and what is this loop called?

A. The gateway/API layer; it's called the "auth loop."
B. The observability and feedback/drift loop; it's called Continuous Training/Continuous Improvement.
C. The registry layer; it's called the "versioning loop."
D. The serving layer; it's called the "autoscaling loop."

---

**Q12 (Short Answer).** In the sequence diagram of a single production request (`architecture.md` Section 2), what three pieces of information must be tagged on every span for a team to later answer "which prompt version, which retrieved documents, and which index snapshot produced this bad output"?

---

**Q13 (Multiple Choice).** Per the decision tree in `architecture.md` Section 3, a system that calls an LLM but does NOT decide at runtime which tools/actions to invoke (i.e., it's a single prompt-in/response-out call) should:

A. Still add the full agentic tracing framework (M17/M18) as a precaution.
B. Skip M17/M18's agent-specific tracing complexity — a single-call LLM pattern is sufficient.
C. Be treated as plain DevOps with no LLMOps concerns.
D. Require a vector database regardless of whether RAG is used.

---

**Q14 (Short Answer).** List the five axes this module uses to compare DevOps, MLOps, and LLMOps (from the Section 3.1 comparison table), and for ONE of them, briefly state how the answer differs across all three.

---

**Q15 (Multiple Choice).** Which statement best reflects this module's guidance on when to apply full MLOps/LLMOps rigor vs. when to avoid over-engineering?

A. Always build the full registry, CI/CD gate, and drift-monitoring stack from day one, regardless of project size.
B. Never build MLOps machinery for a prototype; wait until the system has been in production for at least a year.
C. Apply full rigor to anything touching real users or real money, especially if iterated on repeatedly; for prototypes, avoid heavy machinery but don't make decisions (like hardcoding prompts) that block adding it later.
D. MLOps rigor should be applied uniformly across an entire company regardless of individual project risk or maturity.

---
---

## Answer Key

**Q1: C.** LLMOps is explicitly framed as an extension layer on MLOps, not a replacement, a rival, or a superset that subsumes MLOps.

**Q2:** Any three of: (1) **Data/model versioning and a registry** — because a trained model binary and the data behind it have no source-code equivalent and can't be tracked by git alone; without it you lose reproducibility and rollback ability. (2) **Continuous Training (CT)** — because ML systems can degrade with zero code changes as real-world data shifts, so there needs to be a trigger to retrain without a human writing new code. (3) **Statistical/behavioral evaluation gates** — because pass/fail unit tests can't verify a model's learned statistical behavior is still accurate; you need evaluation against held-out data. (4) **Drift monitoring** — because production monitoring in DevOps (latency/error rate) won't catch silent accuracy degradation from a shifting input distribution.

**Q3: B.** CACE = "Changing Anything Changes Everything," the entanglement problem: because relationships are learned from data rather than hand-coded, any change anywhere can silently ripple through the whole system's behavior.

**Q4:** Unit tests and code review verify that the *code* executes its intended logic correctly — they cannot verify that a *model's* learned statistical behavior is still accurate, because "correct" for an ML model is defined against a real-world data distribution, not against a fixed specification the code review can check. A model can pass every test and still be silently wrong on real-world inputs; you need offline evaluation against held-out/gold data (and for LLMs, potentially LLM-as-judge scoring) to actually validate it.

**Q5: B.** Hallucination — a fluent, confident, well-formatted, entirely wrong answer — has no equivalent in classical ML, where a classifier is simply right or wrong and doesn't fabricate plausible-sounding invented reasoning. (Data drift and Continuous Training are classical-MLOps concepts; model versioning predates LLMOps entirely.)

**Q6:** CT is the automated retraining and redeployment of a model triggered by new data or detected performance drift, without a human writing new application code. DevOps has no equivalent because ordinary deployed code doesn't spontaneously start behaving differently just because the world (e.g., user behavior) shifted underneath it — but a static deployed model can silently degrade that way, so ML needs a mechanism to detect and respond to that.

**Q7: C.** Even with zero fine-tuning, RAG, or agentic behavior, calling a hosted LLM API still has a versioned prompt/system-message artifact, real per-request cost and latency dynamics, non-deterministic open-ended output requiring evaluation beyond exact-match, and hallucination risk — a full LLMOps surface.

**Q8:** The governance gap is the observation that LLMOps deployment/serving tooling (gateways, scaling, serving engines) matured quickly, while lineage/governance tooling — tracing which prompt version, retrieved documents, or vector-index state produced a specific output — lags roughly where classical-MLOps lineage tooling was around 2018. A concrete question such teams can't cheaply answer: "which prompt version, which retrieved documents, and which vector index snapshot produced this specific bad output six weeks ago?"

**Q9: B.** Teams that build elaborate agentic systems without evaluation discipline in place cannot tell whether a change made the system better or worse — they are flying blind, which they typically discover expensively in production.

**Q10:** They are not in tension because the module explicitly distinguishes *teaching order* from *build order*. The course teaches security/cost last because they are cross-cutting concerns that are easier to explain once the full architecture (serving, RAG, agents) already exists to point at — but the tutorial explicitly warns that in a *real project*, these are cheapest to build in from the start and expensive to retrofit, so they should not actually be deferred to the end when building a real system. Teaching sequence and engineering practice sequence are different axes.

**Q11:** (1) the prompt/system-message version, (2) the retrieved documents (and by extension the retriever/query used), and (3) the vector index snapshot ID — plus the model/version identifier — all tagged on the span at the time the request ran. (Accept answers naming the model/version identifier as a fourth item; the tutorial calls out these as the tags needed to answer "why did this go wrong.")

**Q12:** See Q11 — this question and Q11 test the same fact from two directions; either phrasing of "prompt version, retrieved documents/index snapshot, model version" is correct.

**Q13: B.** A single prompt-in/response-out call (no runtime tool/action decisions) is not agentic — per the decision tree, it's sufficient to skip M17/M18's agent-specific tracing complexity, though basic LLMOps practices (versioned prompts, cost tracking, evaluation) from the "calls an LLM" branch still apply.

**Q14:** The five axes are: (1) primary artifact, (2) what changes over time unprompted, (3) what "testing" means before release, (4) what gets versioned, and (5) what production monitoring watches (a sixth, rollback unit, is also listed in the table — accept it too). Example for "what changes over time unprompted": DevOps — nothing changes unprompted (code is static until someone commits); MLOps — the real-world data distribution can drift; LLMOps — data drift plus upstream provider model updates plus retrieved-document staleness, a strictly larger set of unprompted-change risks.

**Q15: C.** Apply full MLOps/LLMOps rigor based on blast radius (real users/money) and iteration frequency; for prototypes/POCs, skip the heavy machinery but avoid decisions that would force a rewrite later (e.g., don't hardcode prompts everywhere if you'll want to version them next month).
