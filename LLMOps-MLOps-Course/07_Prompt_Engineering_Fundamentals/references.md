# References — Prompt Engineering Fundamentals

Official documentation, research papers, and engineering blog posts, organized by the module's
three source-transcript topics plus the expansion areas.

---

## 1. Context Windows, Token Budgets, and Tokenization

### tiktoken (official repository and cookbook example)
**What it teaches:** How BPE tokenization actually works for OpenAI models, how to count tokens
exactly (not estimate), and how to pick the right encoding per model (`o200k_base` for newer
models, `cl100k_base` for GPT-4-era models).
**Difficulty:** Beginner
**Reading time:** 20-30 minutes
**Links:**
- https://github.com/openai/tiktoken
- https://developers.openai.com/cookbook/examples/how_to_count_tokens_with_tiktoken

### Brex Prompt Engineering Guide — Tokens and Context Windows sections
**What it teaches:** A production engineering team's plain-language explanation of tokens,
context-window limits across model generations, and why both input and output tokens must be
budgeted against the same ceiling.
**Difficulty:** Beginner
**Reading time:** 20 minutes
**Link:** https://github.com/brexhq/prompt-engineering

### OpenAI API documentation — Prompting / long-context guidance
**What it teaches:** Current official guidance (as restructured under developers.openai.com in
2026) on structuring prompts for large context windows, including the well-established finding
that placing the actual query/instruction after long reference material improves adherence
(a "recency bias" effect the module's transcript also describes).
**Difficulty:** Beginner-Intermediate
**Reading time:** 20-30 minutes
**Link:** https://developers.openai.com/api/docs/guides/prompting

---

## 2. Structured Prompts (Role/Task/Constraints/Format/Examples) and Schema Enforcement

### Anthropic — Prompting Best Practices / Prompt Engineering Overview
**What it teaches:** Official Claude-specific guidance on being clear and direct, assigning roles
via the system prompt, separating data from instructions (often with XML tags), formatting
output, and prefilling responses to force a starting structure — the vendor-side counterpart to
this module's 5-section (Role/Task/Constraints/Format/Examples) structure.
**Difficulty:** Beginner-Intermediate
**Reading time:** 45-60 minutes for the full section
**Link:** https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices

### OpenAI — Structured Outputs documentation
**What it teaches:** How to force a model's output to conform exactly to a supplied JSON Schema
(as opposed to older, non-guaranteed "JSON mode"), the schema constraints involved (all fields
required, `additionalProperties: false`, nesting/property limits), and when to prefer the
function-calling interface versus the text-response-format interface for constrained decoding.
**Difficulty:** Intermediate
**Reading time:** 30-40 minutes
**Link:** https://developers.openai.com/api/docs/guides/structured-outputs

### LangChain — Structured Output documentation
**What it teaches:** A provider-agnostic way to request structured output from chat models using
Pydantic `BaseModel`s, dataclasses, `TypedDict`s, or raw JSON Schema — covers the two underlying
strategies (native provider structured-output APIs vs. tool-calling fallback) and how LangChain
selects between them automatically, plus retry/error-handling behavior on validation failure.
**Difficulty:** Intermediate
**Reading time:** 20-30 minutes
**Link:** https://docs.langchain.com/oss/python/langchain/structured-output

### Instructor documentation (Pydantic-based structured extraction)
**What it teaches:** How to define a Pydantic model as an LLM's "response schema," get a validated
typed object back instead of raw text, and configure automatic retries when the model's output
fails validation — the most widely adopted pattern for schema-enforced production LLM calls.
**Difficulty:** Beginner-Intermediate
**Reading time:** 30 minutes
**Link:** https://github.com/instructor-ai/instructor

---

## 3. Reducing Hallucination

### Anthropic — Reducing Hallucinations (interactive tutorial, Chapter 8)
**What it teaches:** Grounding techniques, explicit "it's OK to say I don't know" instructions,
citation requirements, and asking the model to quote supporting evidence before answering — the
same case-study-style thinking (grounding + refusal clauses + verification) this module's
transcript covers.
**Difficulty:** Intermediate
**Reading time:** 20-30 minutes
**Link:** https://github.com/anthropics/prompt-eng-interactive-tutorial (Chapter 8)

### OpenAI Cookbook — Techniques to reduce hallucinations / improve reliability
**What it teaches:** Practical patterns including retrieval grounding, having the model cite
sources, chain-of-thought before answering, and lowering temperature for factual/deterministic
tasks — directly supports the module's case study (hallucination rate 18% → 2.1%).
**Difficulty:** Intermediate
**Reading time:** 30 minutes
**Link:** https://github.com/openai/openai-cookbook

---

## 4. Prompting Technique Foundations (Zero-shot / Few-shot / CoT / ReAct)

### Chain-of-Thought Prompting Elicits Reasoning in Large Language Models
**Authors:** Jason Wei, Xuezhi Wang, Dale Schuurmans, Maarten Bosma, Brian Ichter, Fei Xia, Ed
Chi, Quoc Le, Denny Zhou (Google Research/Brain)
**What it teaches:** The original paper establishing that providing worked reasoning-step examples
(chain-of-thought) in a few-shot prompt substantially improves performance on arithmetic,
commonsense, and symbolic reasoning tasks in sufficiently large models — the foundational paper
behind essentially all modern "show your reasoning" prompting techniques.
**Difficulty:** Intermediate-Advanced (research paper)
**Reading time:** 45-60 minutes
**Link:** https://arxiv.org/abs/2201.11903

### ReAct: Synergizing Reasoning and Acting in Language Models
**Authors:** Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran, Karthik Narasimhan, Yuan
Cao
**Venue:** ICLR 2023
**What it teaches:** Interleaving reasoning traces ("Thought") with actions ("Action") and
environment feedback ("Observation") in a single prompt loop, letting a model plan, call external
tools/search, and revise its plan based on real information — the direct conceptual ancestor of
today's agent frameworks.
**Difficulty:** Advanced (research paper)
**Reading time:** 45-60 minutes
**Link:** https://arxiv.org/abs/2210.03629
**Official code:** https://github.com/ysymyth/ReAct

**Note for this module:** Cover ReAct only briefly here as a bridge concept — module 17 (Agents)
is where the course returns to build production agent loops on top of this pattern.

### DAIR.AI Prompt Engineering Guide — Techniques section
**What it teaches:** A single, continuously updated reference covering zero-shot, few-shot,
chain-of-thought, self-consistency, zero-shot-CoT ("Let's think step by step" — Kojima et al.
2022), Auto-CoT, tree-of-thought, and ReAct, each with a short explanation and a worked example.
**Difficulty:** Beginner-Advanced (progressively organized)
**Reading time:** 2-3 hours for the full techniques section
**Link:** https://www.promptingguide.ai/techniques

---

## 5. Automatic Prompt Optimization

### DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines
**Authors:** Omar Khattab et al. (Stanford NLP)
**What it teaches:** How to treat prompts as compiled artifacts of a declarative program rather
than hand-written strings — defines Signatures (typed input/output specs), Modules (composable
prompting strategies), and Optimizers/Teleprompters (algorithms like MIPROv2 and, in later
versions, SIMBA and GEPA) that search over instructions and few-shot demonstrations to maximize a
metric on a training set.
**Difficulty:** Advanced (research paper, but the accompanying docs are practitioner-friendly)
**Reading time:** 60-90 minutes for the paper; the docs/optimizers guide is a faster practical
read (~30 minutes)
**Links:**
- Paper: https://arxiv.org/abs/2310.03714
- Repository: https://github.com/stanfordnlp/dspy
- Optimizers guide: https://github.com/stanfordnlp/dspy/blob/main/docs/docs/learn/optimization/optimizers.md

**When automatic optimization beats manual iteration (synthesis for this module):** Manual
prompt iteration works well when you have a handful of examples, a human-legible failure mode,
and a small number of prompt variants to try. Automatic optimization (DSPy-style) starts to win
once you have (a) an evaluation metric you can compute programmatically (connects to module 09),
(b) a training/validation set of at least dozens to hundreds of labeled examples, and (c) a
multi-stage or multi-module pipeline where hand-tuning each prompt's interaction with the others
becomes combinatorially hard. It also wins when you need to re-optimize quickly after a model
swap (e.g., migrating from one model generation to the next), since the optimizer re-runs the
search rather than requiring a human to re-discover working phrasing by hand.

---

## 6. Multi-Vendor Official Prompting Documentation (for cross-provider awareness)

### OpenAI — Prompt Engineering / Prompting guide
**What it teaches:** Six core strategies: write clear instructions, provide reference text, split
complex tasks into subtasks, give the model time to "think" (chain-of-thought), use external
tools, and test changes systematically (evaluation-driven iteration).
**Difficulty:** Beginner
**Reading time:** 30-40 minutes
**Link:** https://developers.openai.com/api/docs/guides/prompt-engineering

### Anthropic — Prompt Engineering Overview
**What it teaches:** Claude's model-specific philosophy: prefer the simplest prompt that reliably
achieves the goal, use XML tags for structure, and a clear escalation path of techniques (being
clear and direct → examples → chain-of-thought → prefilling → chaining prompts) roughly ordered
from lowest to highest effort.
**Difficulty:** Beginner
**Reading time:** 20-30 minutes
**Link:** https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview

---

## Version/currency notes (mid-2026)

- OpenAI restructured its developer documentation from `platform.openai.com/docs` to
  `developers.openai.com/api/docs` in 2026 — update bookmarks/links accordingly; some older
  blog posts and course materials still point at the deprecated host.
- OpenAI's "Prompts" object/endpoint (a hosted prompt-management feature, `v1/prompts`) is being
  phased out in late 2026 — treat prompt versioning/storage as something you own in your own
  system (per this course's modules 02-03) rather than relying on that specific hosted feature.
- Structured Outputs (schema-guaranteed JSON) should now be treated as the default recommended
  approach over legacy "JSON mode" for any new integration requiring machine-parseable output.
- DSPy has matured well beyond its original teleprompter design; current versions ship multiple
  optimizer strategies (MIPROv2, SIMBA, GEPA) rather than a single algorithm — check the DSPy docs
  for the currently recommended default optimizer before teaching a specific one as canonical.
