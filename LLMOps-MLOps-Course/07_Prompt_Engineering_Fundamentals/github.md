# GitHub Repositories — Prompt Engineering Fundamentals

---

### openai/tiktoken
**Purpose:** OpenAI's official fast BPE tokenizer library — the tool this module's transcript
references directly for "accurate token counting" when computing the context-budget formula
(model limit − system prompt − output reserve = chunk fit).
**Popularity tier:** Very popular / de-facto standard for token counting in the Python LLM
ecosystem.
**Why it matters:** Any production system that needs to know "will this prompt fit in the context
window" or "how much will this API call cost" needs exact tokenization, not a word-count estimate.
tiktoken is 3-6x faster than comparable open-source tokenizers and matches OpenAI's models
exactly via `tiktoken.encoding_for_model(...)`.
**Relation to this module:** Directly implements the token-budgeting theory taught in Part 1 —
learners should use this library (or the equivalent tokenizer for non-OpenAI models) in the
hands-on exercises for computing chunk fit and multi-turn truncation.
**Link:** https://github.com/openai/tiktoken

---

### anthropics/prompt-eng-interactive-tutorial
**Purpose:** Anthropic's official interactive, chapter-based course teaching prompt engineering
against Claude, run as Jupyter-notebook-style exercises with an answer key.
**Popularity tier:** Very popular / widely adopted (tens of thousands of stars) — one of the most
referenced hands-on prompt-engineering resources from any model vendor.
**Why it matters:** It is a working, runnable curriculum rather than a static article — chapters
on "Separating Data from Instructions," "Precognition (Thinking Step by Step)," and "Avoiding
Hallucinations" let learners execute real prompts against a live model and see the failure modes
first-hand instead of just reading about them.
**Relation to this module:** The closest 1:1 match to this module's three-part structure
(context/structure/hallucination) of any resource found — recommended as a companion lab.
**Link:** https://github.com/anthropics/prompt-eng-interactive-tutorial

---

### dair-ai/Prompt-Engineering-Guide
**Purpose:** The community-maintained knowledge base behind promptingguide.ai — guides, papers,
lessons, notebooks, and a "Prompt Hub" of worked examples covering the full space of prompting
techniques (zero-shot, few-shot, CoT, ReAct, self-consistency, tree-of-thought, RAG, agents).
**Popularity tier:** Extremely popular / one of the most-starred educational AI repositories on
GitHub, reached #1 on Hacker News on release and has been used by millions of learners.
**Why it matters:** It is the single broadest free reference for every technique this module
foreshadows (zero-shot/few-shot/CoT/ReAct) and stays current because of active community
maintenance, including model-specific pages (GPT, Claude, Gemini, Llama) that show how the same
technique needs different phrasing per vendor.
**Relation to this module:** Use as the canonical "menu" of techniques to point learners at after
they've internalized this module's core three concepts — it's where they go deeper on any single
technique (e.g., self-consistency, tree-of-thought) the module only foreshadows.
**Link:** https://github.com/dair-ai/Prompt-Engineering-Guide

---

### stanfordnlp/dspy
**Purpose:** A framework from Stanford NLP for "programming, not prompting" language models —
declarative modules (`dspy.Signature`, `dspy.Module`) plus optimizers (MIPROv2, SIMBA, GEPA) that
automatically search over instructions and few-shot demonstrations against a metric, instead of
hand-tuning prompt strings.
**Popularity tier:** Very popular / widely adopted in the LLMOps community as the leading
automatic-prompt-optimization framework, maintained actively by Omar Khattab and the Stanford NLP
group with a large contributor base.
**Why it matters:** It operationalizes exactly the question this module raises — "when does
automatic optimization beat manual iteration?" DSPy's answer: once you have a metric and a
labeled/held-out set (connecting forward to module 09's evaluation content), a compiler can search
a much larger space of prompt variants than a human iterating by hand, and re-optimize
automatically when you swap the underlying model.
**Relation to this module:** Directly supports the "prompt-optimization frameworks" expansion
requested for this module; also a natural forward-reference to module 17 (agents), since DSPy
treats agent loops as composable modules too.
**Link:** https://github.com/stanfordnlp/dspy

---

### 567-labs/instructor
**Purpose:** A Python library for extracting structured, validated data from LLM outputs using
Pydantic models as the schema definition — handles retries on validation failure, streaming
partial objects, and works across OpenAI, Anthropic, Google, Ollama, and Groq through one API.
**Popularity tier:** Very popular / widely adopted (reported ~13.7k GitHub stars and multiple
millions of monthly downloads), used inside teams at major AI labs and enterprises.
**Why it matters:** It is the most direct, idiomatic Python implementation of the "structured
output enforcement" theme this module expands on — instead of writing brittle regex or manual
JSON-schema validation against free-text model output, you define a Pydantic model once and get a
typed, validated Python object back, with automatic re-prompting on schema violations.
**Relation to this module:** Use as the reference implementation for the "JSON schema / Pydantic /
function-calling-style constrained decoding" section — pairs naturally with the production
prompt-template code this module includes.
**Link:** https://github.com/instructor-ai/instructor

---

### brexhq/prompt-engineering
**Purpose:** Brex's internal engineering guide to working with LLMs, published publicly — covers
tokens, context windows, "give a bot a fish vs. teach a bot to fish" (embedding data directly vs.
command grammars/ReAct-style tool use), citations, chain-of-thought, and programmatic output
formatting.
**Popularity tier:** Popular / widely cited (reported ~9.6k GitHub stars) as one of the earliest
public "real company" prompt-engineering guides.
**Why it matters:** Unlike academic surveys, this reads like an internal engineering wiki — plain
language, concrete numbers on token-to-word ratios and context limits per model generation, and
an honest accounting of "aggressive truncation strategies are necessary" once you account for
both input and output tokens counting against the same budget.
**Relation to this module:** A strong real-world companion for Part 1 (context windows/token
budgets) — written by engineers solving this exact problem in production, not researchers.
**Link:** https://github.com/brexhq/prompt-engineering

---

### openai/openai-cookbook
**Purpose:** OpenAI's official collection of example code and guides for common tasks with the
OpenAI API, including token-counting recipes using tiktoken, structured-output/function-calling
examples, and prompt-engineering patterns.
**Popularity tier:** Extremely popular / one of the most-used official reference repositories in
the ecosystem (tens of thousands of stars).
**Why it matters:** It is the first place to look for a runnable, officially-maintained example of
any given pattern (e.g., "how to count tokens with tiktoken," "how to use structured outputs") —
useful as a source of copy-adaptable code for the production prompt-template exercises in this
module.
**Relation to this module:** Direct source for the token-counting and structured-output code
patterns this module teaches; also useful for later RAG-focused modules (09-11).
**Link:** https://github.com/openai/openai-cookbook

---

### ysymyth/ReAct
**Purpose:** The official code release accompanying the ReAct paper (Yao et al., ICLR 2023),
showing the reference implementation of interleaved reasoning-and-acting prompts against
HotpotQA, FEVER, and interactive environments (ALFWorld, WebShop).
**Popularity tier:** Widely cited in academic and applied-AI circles as the reference
implementation of the technique.
**Why it matters:** Seeing the actual prompt templates and trajectory format ReAct uses
(Thought/Action/Observation loops) demystifies how today's agent frameworks (LangChain agents,
DSPy ReAct modules) are built on top of a fairly simple prompting pattern.
**Relation to this module:** Directly supports the module's brief foreshadowing of ReAct/agent
patterns ahead of the deep dive in module 17.
**Link:** https://github.com/ysymyth/ReAct
