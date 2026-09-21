# Books — Module 01: Introduction and Learning Path

These are the anchor texts for the entire 26-module course, not just this orientation chapter. Read the relevant chapters of each as you progress rather than front-loading all of them now — this module tells you *when* each becomes relevant.

---

### 1. Designing Machine Learning Systems
- **Author:** Chip Huyen (O'Reilly, 2022)
- **What it teaches:** A holistic, iterative framework for building production ML systems — covers data, feature engineering, training, evaluation, deployment, and monitoring as one continuous lifecycle rather than isolated notebook experiments. Huyen taught "Machine Learning Systems Design" (CS 329S) at Stanford, and the book distills that course.
- **Difficulty:** Intermediate
- **Estimated reading time:** ~12–15 hours for the full book; ~1 hour for the introductory chapter most relevant to this module (Ch. 1, "Overview of Machine Learning Systems").
- **Why it matters for this module:** This is the best single-source explanation of *why* MLOps exists as a discipline distinct from research ML — the gap this module's "what is MLOps" section addresses. Read Chapter 1 now; the rest maps onto Modules 03–04, 13, and 19 later in the course.

---

### 2. AI Engineering: Building Applications with Foundation Models
- **Author:** Chip Huyen (O'Reilly, January 2025)
- **What it teaches:** How building applications on top of foundation models (LLMs, LMMs) differs from classical ML engineering — the new "AI stack," evaluation of open-ended outputs (including LLM-as-judge), prompt engineering, RAG, agents, and inference optimization.
- **Difficulty:** Intermediate to Advanced
- **Estimated reading time:** ~18–20 hours for the full book (532 pages); ~1 hour for the introductory chapter that frames "AI engineering vs ML engineering."
- **Why it matters for this module:** This is the direct LLMOps counterpart to "Designing Machine Learning Systems," and as of mid-2026 it is widely regarded as the standard reference for the LLM-specific track of this course (Modules 07–18). Read the first chapter now for the MLOps-vs-LLMOps framing; return to later chapters when you reach RAG (Module 15), evaluation (Modules 09–12), and agents (Module 17).
- **Companion resource:** The author maintains a supporting GitHub repo of code and materials — see `github.md` in this module.

---

### 3. LLM Engineer's Handbook
- **Authors:** Paul Iusztin and Maxime Labonne (Packt, October 2024)
- **What it teaches:** End-to-end, hands-on LLMOps: data engineering, supervised fine-tuning, deployment, inference optimization, and preference alignment, built around a running "LLM Twin" case-study project. Iusztin is a senior MLOps/GenAI engineer and Labonne is a Google Developer Expert in AI/ML.
- **Difficulty:** Intermediate to Advanced (assumes comfort with Python and basic ML)
- **Estimated reading time:** ~15 hours for the full book (522 pages); skim the introduction (~30 minutes) for this module.
- **Why it matters for this module:** It is the most implementation-focused of the three anchor texts — useful later as a project-shaped companion to Modules 07, 17, 21, and 26 (capstone). For this orientation chapter, its introduction is a good gut-check for whether the "software engineer who knows Python and basic ML" starting point assumed by this course matches your own background.

---

### 4. Machine Learning Design Patterns
- **Authors:** Valliappa Lakshmanan, Sara Robinson, Michael Munn (O'Reilly, 2020)
- **What it teaches:** 30 catalogued, repeatable solutions to recurring ML engineering problems across data representation, model building, resilience, reproducibility, and operationalization — written by three Google engineers.
- **Difficulty:** Intermediate
- **Estimated reading time:** ~10 hours full book; individual patterns are 10–15 minutes each and can be read standalone.
- **Why it matters for this module:** Less essential as day-one reading, but valuable as a reference you return to throughout the classical-MLOps half of the course (Modules 02–06, 19–22) whenever you hit a named problem (e.g., "reproducibility," "workflow pipeline") and want the canonical pattern name and solution shape.

---

## How to use this list in the overall course
Read Chapter 1 of *Designing Machine Learning Systems* and the introduction of *AI Engineering* this week, alongside this module's roadmap chapter — together they take under 3 hours and will make nearly every later module's vocabulary feel familiar rather than new. Treat the other two books as running companions you dip into module-by-module rather than a queue to finish up front.
