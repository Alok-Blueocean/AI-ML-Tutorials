# Videos — Module 11: LLM-as-Judge: Evaluator Design and Limits

Curated, verified video and course resources. Each entry notes which part of the module it complements: (A) evaluator prompt design / CoT-first judging, (B) composite scoring and thresholds, (C) limits of automated scoring, bias, calibration, and escalation, or (D) the DeepEval/Ragas/G-Eval framework deep dive.

---

### 1. Evaluating AI Agents (DeepLearning.AI short course, in partnership with Arize AI)
- **Creator/Channel:** DeepLearning.AI, taught by John Gilhuly (Head of Developer Relations, Arize AI) and Aman Khan (Director of Product, Arize AI)
- **Duration:** ~2h 36m
- **Difficulty:** Beginner→Intermediate
- **Complements:** A, B, C — this is the single best video resource for this module
- **Why it's worth watching:** This is a full structured course, not a single talk, and it walks through the entire arc this module covers: it distinguishes evaluation approaches (code-based checks vs. LLM-as-a-judge vs. human annotation), shows how to write and iterate on judge prompts, covers trajectory/step-level scoring for agents (directly relevant to composite, per-dimension scoring), and has a dedicated lesson called "Improving Your LLM-as-a-Judge" that walks through validating a judge against ground truth and tightening its rubric — effectively a worked example of the calibration workflow this module teaches.
- **Rating:** ★★★★★

### 2. "Improving Your LLM-as-a-Judge" (lesson within Evaluating AI Agents)
- **Creator/Channel:** DeepLearning.AI / Arize AI (same course as above, standalone-linkable lesson)
- **Difficulty:** Intermediate
- **Complements:** C — calibration and judge validation specifically
- **Why it's worth watching:** Directly demonstrates iterating a judge prompt against a labeled dataset until agreement improves — the practical mechanics behind "calibrate your judge against human raters" that the transcripts introduce conceptually.
- **Rating:** ★★★★☆

### 3. DeepEval documentation walkthroughs and metric explainer videos (Confident AI)
- **Creator/Channel:** Confident AI (maintainers of DeepEval) — video content embedded in and linked from the official docs at deepeval.com
- **Difficulty:** Beginner→Intermediate
- **Complements:** D — the DeepEval framework deep dive
- **Why it's worth watching:** Walks through configuring `GEval` (DeepEval's implementation of the G-Eval chain-of-thought judge pattern), setting `evaluation_steps` explicitly (mirroring the "explicit rubric" principle from the transcripts), and running metrics inside a Pytest-style `deepeval test run` loop — a good bridge from the module's judge_prompt.txt example to a production framework.
- **Rating:** ★★★★☆

### 4. Ragas quickstart and metrics walkthroughs
- **Creator/Channel:** Ragas / explodinggradients (official docs at docs.ragas.io include runnable quickstart notebooks framed as short tutorials)
- **Difficulty:** Beginner
- **Complements:** D — the Ragas framework deep dive, specifically RAG-oriented composite metrics (faithfulness, answer relevancy, context precision/recall) that parallel the module's weighted composite score example
- **Why it's worth watching:** Shows how a real framework operationalizes "per-dimension thresholds" as separate named metrics you compose, rather than one opaque score — reinforcing why the transcripts insist on separating scoring dimensions before combining them.
- **Rating:** ★★★☆☆

---

## Notes on scope

Search access for this task was exhausted before a broader trawl of standalone conference-talk recordings (e.g., AI Engineer World's Fair sessions specifically on LLM-as-judge) could be verified individually by URL and speaker. Rather than list an unverified talk title/URL, this file intentionally stays short and high-confidence. The `references.md` file in this module covers additional written material (papers, engineering blogs, framework docs) that substitute for video coverage on bias types, ensembling, and calibration mechanics in more depth than currently-verifiable video content.

If you have access to a video platform, it is worth separately searching for:
- Recent (2025-2026) "AI Engineer" or "Databricks Data + AI Summit" conference talks specifically titled around "LLM-as-judge," "judge calibration," or "panel of judges" — this space moves fast and new talks are published every few months.
- The official Confident AI (DeepEval) and Ragas YouTube/channel pages directly, since both maintainers periodically publish short "how we built metric X" walkthroughs that update faster than this document can be re-verified.
