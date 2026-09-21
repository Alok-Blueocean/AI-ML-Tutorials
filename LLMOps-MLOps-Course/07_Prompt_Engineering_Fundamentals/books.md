# Books — Prompt Engineering Fundamentals

---

### Prompt Engineering for LLMs
**Authors:** John Berryman and Albert Ziegler
**Publisher:** O'Reilly Media
**What it teaches:** A full-length treatment of prompt engineering as a systems-design
discipline rather than a bag of tricks — covers the anatomy of a prompt (system/user/assistant
roles), few-shot example selection, chain-of-thought and self-consistency, retrieval-augmented
prompting, agentic tool-use patterns, and evaluation of prompt quality. Both authors worked on
GitHub Copilot, so the book leans heavily on real production lessons (context-window budgeting
for code completion, latency/cost tradeoffs, and iterative prompt testing) rather than toy
examples.
**Difficulty:** Intermediate
**Estimated reading time:** 6-8 hours for a full read; 1-2 hours if you selectively read the
chapters on structured prompts, chain-of-thought, and evaluation.
**Why it matters for this module:** This is the most direct book-length companion to this
module's scope — it treats the exact same concerns (token budgets, structuring prompts for
reliability, reducing hallucination through grounding) with the depth a single module can only
summarize. Read the chapters on prompt anatomy and chain-of-thought/self-consistency first.

---

### The Prompt Engineering Guide (book-form / long-form web guide)
**Author/Maintainer:** DAIR.AI (community-maintained, originated from a widely cited academic-
style survey; also distributed as a formatted guide and lecture series)
**What it teaches:** A comprehensive reference spanning prompting fundamentals, all major
techniques (zero-shot, few-shot, chain-of-thought, self-consistency, ReAct, tree-of-thought,
directional-stimulus prompting), model-specific guidance (GPT-family, Claude, Llama, Gemini),
adversarial prompting and prompt injection risks, and a "Prompt Hub" of categorized worked
examples (classification, coding, extraction).
**Difficulty:** Beginner → Advanced (organized so it can be read incrementally)
**Estimated reading time:** 3-5 hours for the core "Techniques" section; the full guide
(including papers, tools, and model-specific pages) is a multi-day reference, not meant to be
read cover-to-cover in one sitting.
**Why it matters for this module:** It is the most-cited free reference for exactly the
technique menu this module foreshadows (zero-shot/few-shot/CoT/ReAct) and is kept reasonably
current by an active community, making it a good living reference to send learners back to as
new techniques emerge. Available free online at promptingguide.ai and as a GitHub repository.

---

### Designing Machine Learning Systems — Chapter on "Human Feedback" / relevant LLM sections
**Author:** Chip Huyen
**Publisher:** O'Reilly Media
**What it teaches:** While not a prompt-engineering book per se, Huyen's book (widely used across
this course's earlier modules) contains directly relevant material on structuring model I/O
contracts, versioning prompts as artifacts, and treating prompt+model+parameters as a single
deployable unit that needs the same rigor as a trained model checkpoint — a framing that connects
this module back to modules 02-03 (versioning/registries) in the course.
**Difficulty:** Intermediate
**Estimated reading time:** 30-45 minutes for the directly relevant sections.
**Why it matters for this module:** Prevents prompt engineering from being taught as an isolated
"art" disconnected from the MLOps lifecycle — reinforces that a prompt template is a versioned,
tested, rolled-back artifact exactly like a model binary, which is the throughline of this entire
course.

---

### Anthropic's "Prompt Engineering" documentation, read as a book-style reference
**Author:** Anthropic (official documentation, not a bound book, but substantial enough to use
as assigned reading)
**What it teaches:** Claude-specific best practices: being clear and direct, using XML tags to
separate instructions from data, prefilling assistant responses, chain-of-thought via extended
thinking, and a dedicated "reducing hallucinations" guide covering grounding, citations, and
allowing the model to say "I don't know."
**Difficulty:** Beginner → Intermediate
**Estimated reading time:** 1-1.5 hours for the full prompt-engineering section.
**Why it matters for this module:** Maps almost exactly onto the module's three transcript
topics (context/structure/hallucination-reduction) but from a different vendor's perspective than
OpenAI's guidance, which is valuable for learners who will work across multiple model providers
in production. Available at: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview
