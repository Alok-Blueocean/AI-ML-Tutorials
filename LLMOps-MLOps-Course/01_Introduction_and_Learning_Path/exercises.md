# Module 01 — Exercises

These exercises are deliberately not coding exercises — this module builds a mental model, not a tool skill, so the "hands-on" work here is thinking artifacts: checklists, classifications, diagrams, and plans you will keep referring back to for the rest of the course. Do them in order; each one leans on the output of the one before it. Budget roughly 2–3 hours total across all eight, spread across your first couple of days in the course.

Keep everything you produce in a personal `module-01/` notes folder (or a private doc/wiki page) — several later modules' exercises (especially Modules 03, 12, 19, and the Module 26 capstone) will ask you to revisit specific artifacts from here.

---

## Exercise 1 — Skills Self-Assessment and Gap-Fill Plan

**Difficulty:** Easy

**Goal:** Honestly complete the Skills Self-Assessment Checklist from `tutorial.md` Section 1, and turn any unchecked boxes into a concrete remediation plan rather than a vague intention.

**Instructions:**
1. Copy the 8-item checklist into your notes and check off what you genuinely have, not what you think you should have.
2. For every unchecked box, write one line naming a specific resource and a specific day/time you will use it (e.g., "Docker basics — official 'Get Started' guide — Tuesday evening, 1 hour").
3. If you checked 5+ of 8, write one sentence per unchecked item on why you're comfortable proceeding without it yet.

**What "done" looks like:** A written checklist with every item marked, and either a concrete remediation plan (with resource + date) for each gap, or an explicit justified decision to proceed. No unchecked box should be left with no plan attached to it.

---

## Exercise 2 — Explain It Without the Book

**Difficulty:** Easy

**Goal:** Prove you actually internalized the Key Terminology table, not just recognized it while reading.

**Instructions:**
Without looking at `tutorial.md`, write a plain-language explanation (2–4 sentences each, no jargon-on-jargon) for these eight terms, as if explaining to a competent backend engineer who has never touched ML:
`MLOps`, `LLMOps`, `drift`, `model registry`, `RAG`, `agent`, `LLM-as-judge`, `observability` (contrasted with monitoring).

Then open the tutorial's terminology table and grade yourself: mark each explanation as Correct / Partially correct / Wrong, and rewrite the wrong or partial ones in your own words a second time.

**What "done" looks like:** All 8 terms explained from memory, self-graded against the source table, and any Wrong/Partial answers corrected and rewritten (not just copied from the table).

---

## Exercise 3 — Classify Five Systems

**Difficulty:** Easy–Medium

**Goal:** Practice applying the DevOps vs. MLOps vs. LLMOps distinction to concrete (if brief) system descriptions — the comparison table in Section 3.1 is only useful once you can apply it to something you didn't write yourself.

**Instructions:** For each system below, decide which discipline(s) apply (DevOps only / + MLOps / + LLMOps) and justify your answer along at least two of the five comparison axes (artifact, testing, versioning, monitoring, rollback):

1. A stateless internal tool that converts CSV files to Parquet on a schedule.
2. A fraud-detection model that scores transactions in real time and is retrained weekly.
3. An internal support-ticket triage tool that sends the ticket text to a hosted LLM API with a fixed system prompt and no fine-tuning, no RAG, and returns a category label.
4. A customer-facing chatbot that retrieves product documentation from a vector database and answers questions from it.
5. A recommendation feed that runs an LLM-based re-ranker on top of a classical collaborative-filtering candidate generator.

**What "done" looks like:** A short table (system → discipline(s) → 2+ axis-based justifications) for all 5 systems. System 3 is the one most beginners get wrong — if you concluded "DevOps only," go back and reread Section 3.1's closing paragraph before moving on.

---

## Exercise 4 — Walk the Decision Tree on Your Own Projects

**Difficulty:** Medium

**Goal:** Apply `architecture.md` Section 3's decision tree to real projects — either ones you currently work on, or three hypothetical ones you invent — rather than the toy examples from Exercise 3.

**Instructions:**
1. Pick three projects (real or invented) that are meaningfully different from each other — at minimum, one should have no ML/LLM component at all, one should be classical ML, and one should involve an LLM.
2. For each, manually walk every decision point in the tree, writing down your answer to each yes/no question and the reasoning behind it.
3. Produce a final bullet list per project of exactly which practices apply (e.g., "needs a model registry, needs drift monitoring, does not need Continuous Training, does not need RAG").

**What "done" looks like:** Three completed tree-walks, each ending in a concrete list of required practices — not just a final label like "MLOps." The practice list is what later modules will ask you to actually build, so vague answers here will cost you time later.

---

## Exercise 5 — Redraw the Reference Architecture from Memory

**Difficulty:** Medium

**Goal:** Test whether you have a working mental model of the full platform (`architecture.md` Section 1) or just recognize it when shown.

**Instructions:**
1. Close `architecture.md`. On paper or in a plain-text file, redraw the reference architecture from memory: at minimum, the 8 major bands (source of truth, CI pipeline, registry layer, CD/orchestration, serving layer, application layer, gateway, observability + feedback loop) and the arrows connecting them.
2. For each band, write the module number(s) that teach it, from memory.
3. Open `architecture.md` Section 1 and compare. Circle anything you got wrong, missing, or in the wrong position (especially: did you remember the feedback loop closing back to the top?).

**What "done" looks like:** A from-memory diagram covering all 8 bands with reasonably correct module mappings, plus an honest self-comparison against the source noting every gap. Do not skip the comparison step — the value of this exercise is in seeing exactly where your mental model was wrong.

---

## Exercise 6 — Justify the Roadmap Order (Essay)

**Difficulty:** Medium–Hard

**Goal:** Move from being able to recite the module order to being able to defend it, which is what an interviewer or a skeptical teammate will actually ask you to do.

**Instructions:** Write 300–500 words answering: *"Why does this course teach release engineering (M02–M06) before prompt engineering (M07), teach evaluation (M09–M12) before RAG/agents (M15–M18), and teach security/cost (M23–M24) last rather than first?"*

Your answer must:
- Reference the "real platform team" build-order argument from Section 3.2 of the tutorial, in your own words.
- Give at least one concrete failure mode that would result from doing it in a *different* order (e.g., building agents before evaluation exists).
- Take a position on whether *build* order should match *teaching* order (hint: the tutorial explicitly says no for security/cost — do you agree, and why?).

**What "done" looks like:** A written essay meeting all three bullet points above, without copy-pasting sentences from the tutorial. If you can, have a peer or study-group partner read it and challenge one part of your argument.

---

## Exercise 7 — Build Your Real Pacing Plan

**Difficulty:** Hard

**Goal:** Turn the generic Section 3.5 pacing table into a plan that is actually yours, accounting for your real background and real available hours.

**Instructions:**
1. Honestly estimate your available focused hours per week for this course over the next ~4 months.
2. Using your Exercise 1 self-assessment, decide which blocks you should compress (e.g., strong Docker/K8s background → compress M05–M06) and which you should expand (e.g., no cloud exposure → add 50% to M05–M06 and M20–M22, as the tutorial suggests).
3. Produce a week-by-week calendar (a simple table is fine) mapping every module to specific calendar weeks, with your adjusted hour estimates, ending in a target completion date.
4. Add one row per block for "risk buffer" — where are you most likely to fall behind, and how much slack have you built in?

**What "done" looks like:** A week-numbered or dated calendar covering all 26 modules and the capstone, with your personalized (not the textbook default) hour estimates and at least one identified risk/buffer point. Revisit and adjust this plan after Module 06 and after Module 18 — pacing plans make on day one are never exactly right.

---

## Exercise 8 — Capstone Pre-Mortem

**Difficulty:** Hardest

**Goal:** Use the "governance gap" insight and the reference architecture to practice the single hardest cognitive skill this course teaches: reasoning backward from a production failure to the missing engineering practice that caused it — before you have any of the tooling in hand.

**Instructions:**
Imagine you have completed the Module 26 capstone and shipped a production RAG+agent system. Six months later, each of the following incidents happens. For each one, identify (a) which band of the Section 1 reference architecture most plausibly failed or was missing, (b) which specific module's practice would have prevented or caught it, and (c) one sentence on how you'd instrument the system today (in Module 01, before writing any code) so this failure is at least *diagnosable* later, even if you can't yet prevent it.

1. A customer complains about a specific bad answer given "sometime last month." Nobody on the team can reconstruct which prompt version, retrieved documents, or model version produced it.
2. The system's answer quality has been silently degrading for three weeks; nobody noticed until a customer escalated, even though no code has been deployed in that window.
3. A cost alert fires showing the monthly LLM API bill tripled; nobody can say which feature, prompt, or user segment caused it.
4. An agent takes an unexpected real-world action (e.g., sends an email it shouldn't have) and no one can reconstruct the reasoning chain that led to it.

**What "done" looks like:** Four scenarios, each with a named architecture band, a named module, and one concrete "instrument this now" sentence. This exercise has no single correct answer key — grade yourself on whether your reasoning is traceable back to a specific box in the Section 1 diagram, not on matching a model answer. Keep this document — you will want to reread it after Module 18 (tracing) and again before starting Module 26.
