# Books and Book Chapters — Prompt Lifecycle: Versioning and Statistical Evaluation

There is no single "textbook" fully dedicated to prompt versioning and statistical promotion testing yet (mid-2026) — this is a young enough discipline that the best material is still primarily blog-length engineering writing and official documentation (see `references.md`). The books below are the closest well-established treatments of the two disciplines this module fuses: (1) rigorous experiment design/statistics, and (2) software configuration/version management applied to ML artifacts.

---

## Trustworthy Online Controlled Experiments: A Practical Guide to A/B Testing

- **Authors:** Ron Kohavi, Diane Tang, Ya Xu (all long-time experimentation leaders — Kohavi built experimentation platforms at Microsoft and Airbnb, Tang and Xu at Google/LinkedIn)
- **Publisher:** Cambridge University Press, 2020
- **What it teaches:** This is the definitive industry reference on A/B testing at scale — covering statistical foundations (hypothesis testing, p-values, confidence intervals, power analysis), the difference between statistical and *practical* significance, common pitfalls (peeking, Simpson's paradox, novelty effects, sample ratio mismatch), and the organizational/platform engineering needed to run experiments reliably at a company. Chapters 3 ("Twyman's Law and Experimentation Trustworthiness"), 17 ("Sample Size and Power"), and 19 ("The Practical Significance vs. Statistical Significance") map almost one-to-one onto the third transcript's rules (n≥200 for 80% power, p<0.05, plus a minimum-lift threshold).
- **Difficulty:** Intermediate — assumes basic stats (means, variance, hypothesis tests) but explains everything from first principles with business-relevant examples; no calculus required.
- **Estimated reading time:** Full book ~10-14 hours; the directly relevant chapters (sample size/power, practical vs. statistical significance, and metrics design) are readable in ~2-3 hours.
- **Why it matters for this module:** Everything in Part 3 of this module — paired t-tests, n≥200, p<0.05, "statistically significant but practically meaningless" — is a specific application of the general A/B-testing discipline this book codifies. Reading the power-analysis and significance chapters turns the transcript's numeric rules from "memorize these thresholds" into "understand why these thresholds exist and how to derive your own for a different metric or effect size."

---

## Statistics Done Wrong: The Woefully Complete Guide

- **Author:** Alex Reinhart
- **Publisher:** No Starch Press, 2015 (also freely readable in an earlier web version at statisticsdonewrong.com)
- **What it teaches:** A short, sharply written tour of the most common statistical mistakes in applied science — p-hacking, underpowered studies, multiple comparisons, misinterpreting p-values, and the base-rate fallacy. Every chapter is a "here's the wrong way people think about this, here's why it's wrong, here's the fix" pattern.
- **Difficulty:** Beginner — deliberately non-mathematical, written for practitioners who need to *not* be fooled by statistics rather than derive them.
- **Estimated reading time:** ~3-4 hours cover to cover; the chapters on power and p-values are ~30 minutes each.
- **Why it matters for this module:** Prompt promotion decisions are exactly the kind of small, frequent, high-stakes-if-wrong decision this book is warning about. Before wiring up an auto-promote gate on `p < 0.05`, an engineer should understand *why* a single significant p-value from one comparison isn't proof of improvement (e.g., if you compare 20 metrics and gate on "any p<0.05", you will get false positives roughly every other release by chance alone) — this book builds that skepticism cheaply.

---

## Accelerate: The Science of Lean Software and DevOps

- **Authors:** Nicole Forsgren, Jez Humble, Gene Kim
- **Publisher:** IT Revolution Press, 2018
- **What it teaches:** Not a prompt/LLM book at all — it's the research-backed case for treating software delivery (deployment frequency, lead time, change failure rate, MTTR) as something to measure and optimize scientifically, and for building fast, low-risk release pipelines with automated gates.
- **Difficulty:** Beginner — business/engineering-leadership register, no math.
- **Estimated reading time:** ~5-6 hours full book; the chapters on continuous delivery and testing are ~1 hour.
- **Why it matters for this module:** The transcripts' core operational idea — an *automated gate* that blocks a risky change before a human has to intervene, with visibility/audit trail for everything that happened — is precisely the DevOps practice this book quantifies the value of. Applying that same discipline to prompts (instead of application code) is the entire thesis of this module; this book supplies the "why does automating the safety gate matter so much" argument that's easy to skip past when you're heads-down writing `delta_check.py`.

---

## Building Machine Learning Powered Applications (chapters on evaluation and iteration)

- **Author:** Emmanuel Ameisen
- **Publisher:** O'Reilly Media, 2020
- **What it teaches:** A practitioner's guide to shipping ML products, with strong chapters on building evaluation harnesses, defining success metrics before building the model/prompt, and iterating safely in production. Predates the LLM-specific prompt-registry tooling but the evaluation-first mindset transfers directly.
- **Difficulty:** Beginner → Intermediate.
- **Estimated reading time:** The evaluation-focused chapters (typically mid-book) run ~1.5-2 hours.
- **Why it matters for this module:** Reinforces the "fixed evaluation dataset, versioned, never silently changed" principle from Part 3 of the transcripts, framed as a general ML engineering discipline rather than an LLM-specific one — useful for connecting this module back to the reproducibility principles from Module 04.

---

## Reading order recommendation

1. Skim **Statistics Done Wrong** chapters on p-values and power first (short, builds healthy skepticism).
2. Read the **Trustworthy Online Controlled Experiments** chapters on sample size/power and practical-vs-statistical significance (the technical core of Part 3).
3. Reference **Accelerate**'s continuous-delivery chapters when designing the auto-promote/auto-block gate architecture (Part 2).
4. Treat **Building Machine Learning Powered Applications** as optional reinforcement if the "fixed eval set" discipline from Module 04 needs a refresher.
