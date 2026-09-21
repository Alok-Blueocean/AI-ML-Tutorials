# Module 07 — Prompt Engineering Fundamentals

> "The prompt is not a string. The prompt is an interface contract between your application code and a
> non-deterministic function you do not own." — the idea that unifies every section below.

---

## 1. Learning Objectives, Prerequisites, Key Terminology

### Learning Objectives

By the end of this module you will be able to:

1. Compute an explicit token budget for any LLM call (system prompt, history, retrieved context, output
   reserve) and defend a chunk-packing strategy for a RAG pipeline that fits inside it.
2. Explain *why* recency bias exists in transformer attention and use it deliberately when placing
   content inside long prompts.
3. Write a 5-section structured prompt (Role / Task / Constraints / Format / Examples) and explain,
   with numbers, why this structure measurably improves routing accuracy and reduces parse failures.
4. Enforce machine-parseable output using JSON Schema, Pydantic models, and provider-native
   structured-output / function-calling APIs — and know when each is the right tool.
5. Apply zero-shot, few-shot, chain-of-thought, and (at a bridging level) ReAct prompting patterns, and
   articulate the tradeoffs between them in terms of cost, latency, and reliability.
6. Diagnose and reduce hallucination using grounding, refusal clauses, citation requirements, and
   temperature control — and explain why these are behavioral *constraints*, not just polite wording.
7. Decide when hand-written iterative prompt engineering is sufficient and when an automatic
   prompt-optimization framework (e.g., DSPy) is the economically correct choice.
8. Build a production-grade, versioned, testable prompt template module in Python that a CI pipeline
   (Module 01) can lint, a registry (Module 03) can version, and an evaluation harness (Module 09) can
   score.

### Prerequisites

- Comfortable calling an LLM API (OpenAI, Anthropic, or an open-weight model server) from Python.
- Basic familiarity with JSON Schema or Pydantic (helpful, not required — this module builds it up).
- Modules 01-06 of this course: you should already think in terms of CI/CD, versioning/registries,
  reproducible environments, and containers/orchestration. This module treats "the prompt" as another
  artifact that deserves that same discipline — it does not re-teach those foundations.

### Key Terminology

| Term | One-line definition |
|---|---|
| **Context window** | The fixed maximum number of tokens (input + output combined, for most APIs) a model can process in a single request. |
| **Token budget** | An explicit accounting of how the context window is allocated across system prompt, history, retrieved context, and reserved output space. |
| **Recency bias** | The empirically observed tendency of transformer attention to weight tokens closer to the generation point more heavily — meaning position in the prompt is not neutral. |
| **Structured prompt** | A prompt organized into named sections (commonly Role, Task, Constraints, Format, Examples) rather than a single unstructured paragraph of instructions. |
| **Few-shot prompting** | Including a small number (typically 2-5) of worked input-output examples inside the prompt to demonstrate the desired pattern. |
| **Chain-of-thought (CoT)** | Prompting the model to produce intermediate reasoning steps before its final answer, which empirically improves accuracy on multi-step reasoning tasks. |
| **ReAct** | A pattern interleaving *Thought* (reasoning), *Action* (tool call), and *Observation* (tool result) in a loop — the direct conceptual ancestor of today's agent frameworks (Module 17). |
| **Structured output / constrained decoding** | Forcing a model's output tokens to conform to a schema (JSON Schema, regex, grammar) either via prompting, sampling-time constraints, or a provider-native API. |
| **Hallucination** | Model output that is fluent and confident but factually unsupported by the model's actual knowledge or the provided context. |
| **Grounding** | Instructing (and structurally forcing) the model to answer only from a supplied context, rather than from parametric memory. |
| **Refusal clause** | An explicit instruction telling the model it is acceptable — required — to say "I don't know" rather than guess. |
| **Temperature** | A sampling parameter controlling output randomness; `0` (or near-zero) makes output close to deterministic/greedy. |
| **Prompt optimization framework** | A system (e.g., DSPy) that treats prompt wording and few-shot examples as parameters to be searched/optimized against a metric, rather than hand-written. |
| **Prompt template** | A parameterized, versioned piece of code/config that renders a final prompt string from a schema of inputs — the production unit of a prompt, as opposed to an ad hoc string. |

---

## 2. Why This Topic Matters — Where It Fits in the MLOps/LLMOps Lifecycle

Modules 01-06 gave you the machinery to ship *code* and *containers* reliably: CI/CD gates, versioned
artifacts, reproducible environments, Docker, Kubernetes. All of that machinery assumes the thing you are
shipping is deterministic enough that "it passed tests, ship it" is a meaningful sentence.

An LLM call breaks that assumption in one specific way: the same code, the same container, the same
weights, can produce different behavior depending on **what string you send it**. In classical ML, the
model is fixed and the input distribution is what you monitor for drift. In LLM systems, there is a whole
additional artifact sitting between your application logic and the model weights — the prompt — and it is
frequently the highest-leverage, cheapest-to-change, easiest-to-break piece of the entire system.

```
   Classical ML lifecycle (Modules 01-06):

   Data -> Training -> Model Registry -> Container -> K8s Deployment -> Monitored Traffic
                                                                              |
                                                                    (model itself is fixed;
                                                                     you monitor input drift)

   LLM lifecycle (this module onward):

   Data/Docs -> Prompt Template -> [ + Retrieved Context ] -> LLM API Call -> Structured Output
                    ^                                              |
                    |                                              v
              (THIS MODULE:                                 Downstream code that
               design, budget,                               PARSES this output and
               structure, ground,                             acts on it (tools, DB writes,
               version this)                                  UI rendering, agent loops...)
```

Why this is a distinct MLOps/LLMOps discipline, not "just writing good prompts":

1. **The prompt is the interface contract.** Every downstream system — parsers, tool-callers, UI
   renderers, other agents — depends on the model producing output in a specific, predictable shape.
   A prompt that "usually" produces valid JSON is a production incident waiting to happen; this module is
   about making "usually" become "verifiably, close to always."

2. **It is a cost and latency lever, not just a quality lever.** Every token in the system prompt, every
   few-shot example, every unnecessarily large retrieved-context chunk is money and milliseconds on every
   single request, multiplied by request volume. Module 08 covers cost/latency economics in depth; this
   module gives you the token-budgeting discipline that Module 08 builds on.

3. **It is where hallucination — the single most reputation-damaging LLM failure mode — is
   engineered away or engineered in.** A support bot that fabricates a refund policy, a legal-summary tool
   that invents a case citation, a medical-info assistant that states an incorrect dosage: these are
   prompt-engineering failures as much as model failures, and this module's grounding/refusal/citation
   techniques are the primary lever available before you reach for fine-tuning or RAG architecture changes.

4. **It precedes and feeds every later module in this course.** Prompt versioning and lifecycle
   management (Module 09), building evaluation datasets (Module 10), LLM-as-judge (Module 11), deployment
   quality gates (Module 12), and observability (Module 14) all operate on prompts as first-class,
   versioned, measurable artifacts. You cannot evaluate, gate, or roll back what you have not first learned
   to structure, budget, and template — which is exactly this module's job.

In interview terms: a senior LLMOps engineer is expected to reason about prompts the way a senior backend
engineer reasons about API contracts — with schemas, versioning, budgets, and failure modes — not the way
a hobbyist reasons about "getting ChatGPT to do a cool thing."

---

## 3. Main Concepts

### 3.1 Context Windows and Token Budgets

#### Theory

Every model has a fixed context window measured in tokens — the atomic units a model actually processes
(sub-word pieces, not characters or whole words). In most modern chat APIs, **input and output tokens
share the same ceiling**: if the model's context window is 128K tokens and your prompt consumes 127K, you
have roughly 1K tokens left for the entire response, no matter how large the window nominally is.

The core discipline is captured in one formula, straight from production practice:

```
   available_context_for_retrieval/history =
       MODEL_CONTEXT_LIMIT
       - SYSTEM_PROMPT_TOKENS
       - OUTPUT_RESERVE_TOKENS
```

This looks trivial, but the failure mode it prevents is one of the most common in early-stage LLM
products: a team builds a RAG pipeline, retrieves the "top 20" chunks because that number felt generous,
concatenates them without counting tokens, and either (a) gets a hard context-length-exceeded API error
under some documents but not others, or (b) — worse — silently gets truncated by the provider or client
library, so the *most relevant* retrieved chunk is the one that got cut off, and nobody notices until a
user complains the answer is wrong for a document that was "definitely" in context.

**Why this matters more than it looks like it should:**

- **It's not just about avoiding errors.** Even when everything technically fits, packing the window
  inefficiently (redundant chunks, no metadata, chunks in retrieval-score order rather than
  attention-aware order) measurably degrades answer quality, because you are spending token budget on the
  wrong things and positioning the right things badly.
- **The budget is dynamic, not a constant.** System prompt length changes as you iterate on instructions.
  Output reserve depends on task (a one-word classification needs far less reserve than a
  500-word summary). History length changes every turn in a multi-turn conversation. Treating the budget
  as "we'll just use a big model with a big window and not worry about it" is an anti-pattern covered in
  §5 — bigger windows change the constant, not the discipline.

**Tradeoffs / when this gets hard:**

- Long-context models (1M+ token windows, as several frontier models now offer in 2026) reduce the
  *frequency* with which you hit hard limits, but do **not** eliminate the need for budgeting: recency
  bias and "lost in the middle" effects (models attending unevenly across a very long context, generally
  favoring the beginning and end over the middle) mean that simply having room does not mean quality is
  preserved by stuffing it full. Cost also scales with tokens sent regardless of window size, so budgeting
  remains a cost-control discipline even when it stops being a hard-limit-avoidance discipline.
- For agentic/multi-turn systems, the budget must be *re-derived every turn* — see §3.1.3 below on
  multi-turn truncation.

#### Architecture — Token Budget Allocation

```
 +-------------------------------------------------------------------------------+
 |                          MODEL CONTEXT WINDOW (e.g., 128,000 tokens)          |
 |                                                                                 |
 |  +----------------+  +------------------+  +----------------------------+     |
 |  | SYSTEM PROMPT   |  | CONVERSATION      |  | RETRIEVED CONTEXT /        |     |
 |  | (role, task,    |  | HISTORY           |  | RAG CHUNKS                 |     |
 |  | constraints,    |  | (prior turns,     |  | (sorted by relevance,      |     |
 |  | format, few-shot|  | trimmed oldest-    |  | metadata-tagged,           |     |
 |  | examples)       |  | first if needed)   |  | MOST RELEVANT PLACED LAST  |     |
 |  |                 |  |                    |  | for recency-bias benefit)  |     |
 |  |   ~400 tokens   |  |   variable         |  |   ~2,800 tokens            |     |
 |  +----------------+  +------------------+  +----------------------------+     |
 |                                                                                 |
 |                                                        +--------------------+  |
 |                                                        | OUTPUT RESERVE      |  |
 |                                                        | (never spend this — |  |
 |                                                        | model needs room to |  |
 |                                                        | answer)             |  |
 |                                                        |   ~800 tokens       |  |
 |                                                        +--------------------+  |
 +-------------------------------------------------------------------------------+
```

The rule of thumb from production practice: **budget first, retrieve/pack second.** Decide
`OUTPUT_RESERVE_TOKENS` and `SYSTEM_PROMPT_TOKENS` up front (they are close to fixed per task type), then
size everything else — history and retrieval — against what remains, never the other way around.

#### Examples

**Beginner** — naive, unbudgeted prompt assembly (the anti-pattern):

```python
# DON'T DO THIS — no token accounting at all
def build_prompt(system_prompt, chunks, question):
    context = "\n\n".join(chunk.text for chunk in chunks)  # all chunks, no limit
    return f"{system_prompt}\n\nContext:\n{context}\n\nQuestion: {question}"
```

This "works" until a document set produces enough chunks to exceed the model's window, at which point it
either errors out or gets silently truncated by the client/provider — usually cutting off exactly the part
you needed (the end, where you placed the most relevant thing, right before the model's answer).

**Intermediate** — explicit token budgeting with `tiktoken`:

```python
import tiktoken

def get_encoding_for_model(model: str):
    try:
        return tiktoken.encoding_for_model(model)
    except KeyError:
        # Fallback for models tiktoken doesn't recognize by name yet
        return tiktoken.get_encoding("o200k_base")

def count_tokens(text: str, model: str = "gpt-4o") -> int:
    enc = get_encoding_for_model(model)
    return len(enc.encode(text))

def compute_budget(model_limit: int, system_prompt: str, output_reserve: int, model: str) -> int:
    system_tokens = count_tokens(system_prompt, model)
    available = model_limit - system_tokens - output_reserve
    if available <= 0:
        raise ValueError(
            f"No room left for context: system={system_tokens}, "
            f"reserve={output_reserve}, limit={model_limit}"
        )
    return available
```

**Production-grade** — full RAG chunk-packing respecting the budget, metadata injection, and
recency-biased ordering:

```python
from dataclasses import dataclass
from typing import List
import tiktoken

@dataclass
class RetrievedChunk:
    text: str
    source: str
    date: str
    relevance_score: float

def format_chunk(chunk: RetrievedChunk) -> str:
    # Metadata injection: helps the model reason about provenance and recency,
    # and gives it something concrete to cite back (see hallucination-reduction §3.3).
    header = f"[source: {chunk.source} | date: {chunk.date} | relevance: {chunk.relevance_score:.2f}]"
    return f"{header}\n{chunk.text}"

def pack_chunks_to_budget(
    chunks: List[RetrievedChunk],
    token_budget: int,
    encoding: tiktoken.Encoding,
) -> str:
    """
    Sort by relevance (best last, exploiting recency bias), greedily add chunks
    until the budget is exhausted, then reverse the accepted set so the MOST
    relevant chunk sits closest to the model's answer.
    """
    # Sort ascending so the best chunk is considered/placed last.
    sorted_chunks = sorted(chunks, key=lambda c: c.relevance_score)

    accepted: List[str] = []
    running_tokens = 0

    # Walk from most-relevant backward, so if we run out of budget we drop the
    # LEAST relevant chunks first, not the most relevant ones.
    for chunk in reversed(sorted_chunks):
        formatted = format_chunk(chunk)
        chunk_tokens = len(encoding.encode(formatted))
        if running_tokens + chunk_tokens > token_budget:
            continue  # skip chunks that no longer fit; keep trying smaller ones
        accepted.append(formatted)
        running_tokens += chunk_tokens

    # accepted[] was built most-relevant-first; reverse so most-relevant ends up
    # LAST in the final string (closest to the model's generation point).
    accepted.reverse()
    return "\n\n---\n\n".join(accepted)


def build_grounded_prompt(
    system_prompt: str,
    chunks: List[RetrievedChunk],
    question: str,
    model: str = "gpt-4o",
    model_limit: int = 128_000,
    output_reserve: int = 800,
) -> str:
    encoding = tiktoken.encoding_for_model(model) if model in tiktoken.model.MODEL_TO_ENCODING else tiktoken.get_encoding("o200k_base")
    system_tokens = len(encoding.encode(system_prompt))
    budget_for_context = model_limit - system_tokens - output_reserve - len(encoding.encode(question))

    if budget_for_context <= 0:
        raise ValueError("System prompt + reserve leaves no room for context")

    packed_context = pack_chunks_to_budget(chunks, budget_for_context, encoding)

    return (
        f"{system_prompt}\n\n"
        f"Context:\n{packed_context}\n\n"
        f"Question: {question}"
    )
```

#### Multi-turn truncation

When a conversation grows across turns, the same budgeting discipline applies, with one additional rule:
**trim the oldest history first, but always preserve the system prompt and the most recent N turns.**

```python
from typing import List, Dict

def trim_history_to_budget(
    system_prompt: str,
    history: List[Dict[str, str]],   # [{"role": ..., "content": ...}, ...]
    min_recent_turns: int,
    token_budget: int,
    encoding,
) -> List[Dict[str, str]]:
    """
    Always keep the last `min_recent_turns` messages (they contain the active
    user intent). Drop OLDEST messages beyond that floor until we fit the budget.
    """
    if len(history) <= min_recent_turns:
        return history  # nothing safe left to trim

    protected = history[-min_recent_turns:]
    trimmable = history[:-min_recent_turns]

    def total_tokens(msgs):
        return sum(len(encoding.encode(m["content"])) for m in msgs)

    while trimmable and total_tokens(protected + trimmable) > token_budget:
        trimmable.pop(0)  # drop the OLDEST remaining message first

    return trimmable + protected
```

This mirrors exactly the production principle: the system prompt defines behavior and must never be
evicted; the most recent turns carry the live task and must never be evicted; everything else is a
disposable cache that gets pruned oldest-first when space runs short.

#### Comparison table — budgeting strategies

| Strategy | What it does | When to use | When NOT to use |
|---|---|---|---|
| **Hard truncation (client-side cutoff)** | Silently cuts text at N tokens with no awareness of structure | Never as primary strategy — only as a last-resort safety net | As your only defense — it will cut mid-sentence or mid-JSON and corrupt output |
| **Oldest-first history trimming** | Drops earliest conversation turns first | Multi-turn chat/agent loops where recent turns matter most | Tasks where an early turn set critical constraints that must persist (pin those into the system prompt instead) |
| **Relevance-sorted RAG packing** | Greedily fits highest-relevance chunks first, drops lowest-relevance | Any RAG pipeline with variable retrieval-set size | Tasks needing exhaustive coverage of *all* retrieved docs (consider map-reduce/summarize-then-combine instead) |
| **Summarization/compaction** | Replaces old turns with an LLM-generated summary instead of dropping them | Long-running agent sessions where early context has durable value | Latency/cost-sensitive paths — summarization is itself an extra LLM call |
| **Sliding window with sticky system prompt** | Fixed-size window of recent turns + permanently retained system prompt | Simple chat UIs with no long-term memory requirement | Anything needing persistent memory across sessions (use external memory store instead) |

---

### 3.2 Structured Prompts (Role / Task / Constraints / Format / Examples)

#### Theory

An unstructured prompt — "classify this support query" — asks the model to infer, on every single call,
what a good answer even looks like. A structured prompt removes that inference burden by explicitly
specifying five things:

| Section | Answers the question | Typical content |
|---|---|---|
| **Role** | Who should the model act as? | "You are a support-ticket triage classifier for a B2B SaaS company." |
| **Task** | What, concretely, should it do? | "Classify the ticket into exactly one of the categories below." |
| **Constraints** | What must it never do / always do? | "Never invent a category not in the list. Never explain your reasoning in the output." |
| **Format** | What shape must the output take? | "Respond with a single JSON object matching this schema: {...}" |
| **Examples** | What does a correct answer look like? | 2-3 worked (input ticket -> correct JSON output) pairs |

**Why this works, mechanistically:** an LLM is doing next-token prediction conditioned on everything in
its context. A loose instruction leaves enormous latitude in *how* to satisfy it — which is exactly the
freedom that produces inconsistent formatting, invented categories, and unparseable output. Structure
collapses that latitude at each of five decision points independently, and the effects compound.

**The measured impact is not a rounding error.** In the production example this module is grounded in,
changing *only* the prompt (same model, same task, same underlying data) from a one-line instruction to
the full 5-section structure with two examples took:

| Metric | Unstructured prompt | 5-section structured prompt (+2 examples) |
|---|---|---|
| Routing accuracy | 71% | 96% |
| Unparseable/parse-error rate | 15% | 0.5% |

This is one of the highest-leverage, lowest-cost interventions available in the entire LLM engineering
toolkit: no fine-tuning, no new infrastructure, no additional model calls — just better-organized
instructions. It is also why prompt review deserves the same rigor as code review (see Module 09 on
prompt lifecycle) rather than being treated as a one-off creative-writing task.

**When NOT to over-invest here:** for very simple, low-stakes, single-shot tasks (e.g., "translate this
sentence"), the full 5-section apparatus is overkill and adds token cost/latency for no measurable
accuracy gain — match structural investment to task complexity and failure cost.

#### Architecture — the structured prompt as a contract

```
 +===========================================================================+
 |                          STRUCTURED PROMPT CONTRACT                        |
 |                                                                             |
 |  [ROLE]        "You are a support-ticket triage classifier..."             |
 |       |                                                                    |
 |       v  (frames the model's persona/expertise, narrows its behavior space)|
 |  [TASK]        "Classify the ticket into exactly one category from: ..."   |
 |       |                                                                    |
 |       v  (states the concrete objective, unambiguously)                   |
 |  [CONSTRAINTS] "Never invent a category. Never output prose outside JSON.  |
 |                 If ambiguous between two categories, choose the more      |
 |                 specific one."                                            |
 |       |                                                                    |
 |       v  (rules out failure modes explicitly, rather than hoping)         |
 |  [FORMAT]      { "category": string, "confidence": float,                 |
 |                  "evidence": [string], "follow_up_needed": bool }         |
 |       |                                                                    |
 |       v  (defines the shape downstream code can rely on)                  |
 |  [EXAMPLES]    Example 1: input ticket -> correct JSON output              |
 |                Example 2: input ticket -> correct JSON output              |
 |                (anchors style, format, and edge-case handling)             |
 +===========================================================================+
                                    |
                                    v
                     +---------------------------+
                     |  Downstream code: parse,    |
                     |  validate against schema,   |
                     |  route/act on result        |
                     +---------------------------+
```

#### Examples

**Beginner** — unstructured (the "before" state from the case study):

```
Classify the support query.
```

Fragile: no category list, no output format, no constraints. Every run is a coin flip on format.

**Intermediate** — role + task + basic format, no examples yet:

```
You are a support-ticket triage classifier for a B2B SaaS company.

Classify the following support ticket into exactly one of these categories:
billing, technical_issue, feature_request, account_access, other.

Respond with only the category name, nothing else.

Ticket: {ticket_text}
```

Better — but still brittle at the edges (ambiguous tickets, multi-issue tickets) and gives the model no
worked pattern to imitate.

**Production-grade** — full 5-section structure with schema and few-shot examples:

```python
SUPPORT_TRIAGE_PROMPT = """\
[ROLE]
You are a support-ticket triage classifier for a B2B SaaS company. You have deep \
familiarity with common billing, technical, and account-access issues.

[TASK]
Classify the support ticket below into exactly one of these categories:
- billing
- technical_issue
- feature_request
- account_access
- other

[CONSTRAINTS]
- Choose exactly one category. Never invent a category not in the list above.
- If the ticket could fit two categories, choose the more specific one \
(e.g., "can't log in after being charged twice" -> account_access, not billing).
- Do not include any explanation, reasoning, or prose outside the JSON object.
- If you are not confident, still choose your best category, but set \
"confidence" honestly (low confidence is a valid, useful signal).

[FORMAT]
Respond with a single JSON object exactly matching this schema:
{{
  "category": "billing" | "technical_issue" | "feature_request" | "account_access" | "other",
  "confidence": <float between 0.0 and 1.0>,
  "evidence": [<short quoted phrases from the ticket that justify the category>],
  "follow_up_needed": <true if a human agent should review this, else false>
}}

[EXAMPLES]
Ticket: "I was charged twice this month for my subscription, please refund one of them."
Output: {{"category": "billing", "confidence": 0.97, "evidence": ["charged twice this month"], "follow_up_needed": false}}

Ticket: "The export button does nothing when I click it, no error message either."
Output: {{"category": "technical_issue", "confidence": 0.9, "evidence": ["export button does nothing", "no error message"], "follow_up_needed": true}}

Ticket: {ticket_text}
Output:"""
```

Note the output schema itself: `confidence` and `follow_up_needed` are not decorative — they let
downstream code make a *safe-handling* decision (route low-confidence or `follow_up_needed=true` tickets
to a human) instead of blindly trusting every classification. `evidence` additionally discourages
unsupported claims, foreshadowing §3.3's hallucination-reduction techniques — asking for evidence is
itself an anti-hallucination technique, not just a formatting nicety.

#### Key principles (from production practice)

1. Use the 5-section structure (Role, Task, Constraints, Format, Examples) as your default scaffold.
2. Write constraints as explicit positive *and* negative rules ("always do X", "never do Y") — don't rely
   on the model inferring what you didn't say.
3. Define the exact output format (JSON Schema, CSV columns, XML tags) rather than describing it in prose.
4. Use 2-3 diverse few-shot examples — diverse enough to cover edge cases, not so many that you burn
   token budget for diminishing returns (see §3.1 tradeoffs).
5. Treat the prompt as a contract: it defines both *content* (what to say) and *shape* (how to say it).

---

### 3.3 Structured Output Enforcement (JSON Schema / Pydantic / Function-Calling)

#### Theory

Section 3.2 showed that *asking* for a JSON format in prose dramatically improves reliability. But
"dramatically improved" is not "guaranteed." Even a well-structured prompt can occasionally produce a
trailing comma, an extra sentence before the JSON, or a field that doesn't match the schema — because the
model is still free-generating tokens; it is not mechanically constrained to only ever produce valid JSON.

This is the problem **constrained decoding / structured outputs** solves: instead of relying entirely on
instruction-following, the serving stack (provider-side or client-side) restricts *which tokens are
even sampleable* at each step so that only schema-valid continuations are possible. There are three
distinct mechanisms in production use, worth telling apart precisely because interviewers probe this
distinction:

| Mechanism | How it works | Guarantee level |
|---|---|---|
| **Prompted JSON ("JSON mode," legacy)** | Ask nicely in the prompt, maybe set a `response_format: json_object` flag that guarantees *syntactically valid* JSON only | Valid JSON syntax; **no schema guarantee** — wrong fields/types still possible |
| **Provider-native Structured Outputs** (OpenAI `json_schema` response format, Anthropic tool-use-forced schemas) | You supply a JSON Schema; the provider's decoding constrains token sampling so output is guaranteed schema-conformant | Strong: syntax + schema (types, required fields, enums) guaranteed by the provider |
| **Function/tool-calling as a structuring mechanism** | Define a "tool" whose parameters are your schema; force the model to call it; read the tool-call arguments as your structured result | Strong, same underlying mechanism as native Structured Outputs in most current APIs — often the more ergonomic path when you're already in a tool-calling architecture |

**Why prefer provider-native Structured Outputs over legacy JSON mode as of 2026:** JSON mode only
guarantees the output *parses* as JSON — it says nothing about which keys exist or what types they hold.
Structured Outputs (schema-guaranteed) is now the recommended default for any new integration that needs
machine-parseable output, precisely because it moves the guarantee from "probably, if the prompt is good"
to "mechanically enforced by the decoding process."

**Where Pydantic and libraries like Instructor fit in:** Pydantic gives you a typed, validated Python
object instead of "a dict that is hopefully shaped right." Instructor (and LangChain's structured-output
interface) wrap a provider call so that you pass a Pydantic model as the target schema and get back a
validated instance of that model — with automatic retry-on-validation-failure built in, which matters
because even schema-constrained decoding can occasionally produce output that's syntactically schema-valid
but semantically wrong (e.g., a required string field that's technically present but empty).

**Tradeoffs:**

- Provider-native structured outputs typically add a small latency overhead (the constrained decoder does
  more work per token) and, for some providers, restrict which schema features are supported (e.g., not
  every JSON Schema keyword is honored — check current docs, this changes across provider versions).
- Function-calling-as-structuring is convenient in agentic architectures where you already have tool
  definitions, but conceptually overloads "tool call" to also mean "structured response," which can be
  confusing in traces/observability — be deliberate about this in your system design (Module 14 revisits
  this in the context of tracing agent calls).
- Client-side-only enforcement (validate-and-retry with Pydantic, no provider-native constraint) is the
  most portable option (works with any model, including ones with no structured-output API) but costs
  extra round trips on validation failure and gives weaker worst-case guarantees.

#### Architecture — structured output enforcement layers

```
                     +-------------------------------------------------+
                     |     Application defines a SCHEMA once            |
                     |     (Pydantic model / JSON Schema)                |
                     +---------------------+-----------------------------+
                                           |
              +----------------------------+-----------------------------+
              |                            |                              |
              v                            v                              v
   +--------------------+     +-------------------------+     +----------------------+
   | Layer 1: PROMPT      |     | Layer 2: PROVIDER-NATIVE  |     | Layer 3: CLIENT-SIDE   |
   | shows the schema in   |     | constrained decoding       |     | validation + retry     |
   | [FORMAT] section       |     | (response_format=         |     | (Pydantic .model_      |
   | (§3.2)                |     |  json_schema, or forced    |     |  validate, Instructor  |
   |                       |     |  tool call)                |     |  retry-on-failure)     |
   | Weak guarantee alone  |     | Strong guarantee, but      |     | Safety net — catches   |
   |                       |     | provider/feature-dependent |     | anything that still    |
   |                       |     |                            |     | slips through          |
   +--------------------+     +-------------------------+     +----------------------+
              |                            |                              |
              +----------------------------+-----------------------------+
                                           |
                                           v
                          +-------------------------------------+
                          |  Validated, typed object handed to    |
                          |  downstream business logic            |
                          +-------------------------------------+
```

The production-grade pattern is **defense in depth**: use all three layers together. The prompt's
[FORMAT] section primes the model toward the right shape (cheap, always available); the provider-native
constraint mechanically enforces it when supported (strong, provider-dependent); client-side Pydantic
validation is the non-negotiable last line of defense (catches provider bugs, unsupported schema features,
and semantic-but-not-syntactic errors), with a bounded retry budget.

#### Examples

**Beginner** — prompted-only, no enforcement (fragile):

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Return JSON with fields name and age for: John, 30"}],
)
data = json.loads(response.choices[0].message.content)  # will sometimes raise JSONDecodeError
```

**Intermediate** — Pydantic model + provider-native Structured Outputs:

```python
from pydantic import BaseModel, Field
from openai import OpenAI

class TicketClassification(BaseModel):
    category: str = Field(description="One of: billing, technical_issue, feature_request, account_access, other")
    confidence: float = Field(ge=0.0, le=1.0)
    evidence: list[str]
    follow_up_needed: bool

client = OpenAI()

response = client.responses.parse(
    model="gpt-4o-2024-08-06",
    input=[
        {"role": "system", "content": SUPPORT_TRIAGE_PROMPT_HEADER},  # role/task/constraints
        {"role": "user", "content": ticket_text},
    ],
    text_format=TicketClassification,  # schema derived from the Pydantic model
)

result: TicketClassification = response.output_parsed  # guaranteed-typed, no manual json.loads
```

**Production-grade** — Instructor-style client-agnostic wrapper with validation retry, usable across
providers (OpenAI, Anthropic, etc.) without rewriting business logic per provider:

```python
import instructor
from anthropic import Anthropic
from pydantic import BaseModel, Field, field_validator

class TicketClassification(BaseModel):
    category: str
    confidence: float = Field(ge=0.0, le=1.0)
    evidence: list[str]
    follow_up_needed: bool

    @field_validator("category")
    @classmethod
    def category_must_be_known(cls, v: str) -> str:
        allowed = {"billing", "technical_issue", "feature_request", "account_access", "other"}
        if v not in allowed:
            raise ValueError(f"category {v!r} not in {allowed}")
        return v

client = instructor.from_anthropic(Anthropic())

result: TicketClassification = client.chat.completions.create(
    model="claude-sonnet-4-5",
    max_tokens=500,
    max_retries=2,  # automatic retry-with-error-feedback on validation failure
    messages=[
        {"role": "user", "content": f"{SUPPORT_TRIAGE_PROMPT_HEADER}\n\nTicket: {ticket_text}"}
    ],
    response_model=TicketClassification,
)
```

The `field_validator` matters: it encodes the [CONSTRAINTS] section's rule ("never invent a category")
as *code*, not just prose — so even a hypothetical model regression that starts inventing category names
is caught deterministically rather than silently passed downstream.

---

### 3.4 Prompting Patterns: Zero-Shot, Few-Shot, Chain-of-Thought, and (Bridging to) ReAct

#### Theory

These four patterns form an escalating ladder of "how much scaffolding does the model need to perform this
task reliably," and picking the right rung is a cost/latency/reliability tradeoff, not a "always use the
most advanced one" decision.

| Pattern | What it is | Extra cost | Best for | Weak for |
|---|---|---|---|---|
| **Zero-shot** | Instruction only, no worked examples | None (baseline) | Simple, well-known tasks the base model already handles well (sentiment, straightforward extraction) | Tasks with idiosyncratic formatting rules or ambiguous edge cases |
| **Few-shot** | 2-5 worked input/output examples included in the prompt | Extra input tokens per call (fixed cost, every request) | Tasks with a specific house style, format, or edge-case handling that's easier to *show* than *describe* | Tasks where good examples are scarce, or where examples might leak sensitive/PII data into every prompt |
| **Chain-of-thought (CoT)** | Model produces reasoning steps before the final answer (elicited via prompt or via example demonstrations) | Extra output tokens (reasoning tokens), extra latency | Multi-step arithmetic, logic, planning, anything requiring intermediate deduction | Simple lookups/classifications where reasoning adds latency/cost with no accuracy benefit; also weaker when you must show the *reasoning itself* to a user verbatim and it might contain something misleading (a known critique of raw CoT exposure) |
| **ReAct (Thought/Action/Observation loop)** | CoT interleaved with tool calls and real environment feedback, in a loop until a final answer | Multiple model round-trips, tool-call latency and cost, more complex failure surface | Tasks requiring real-world information the model doesn't have (search, calculators, APIs, retrieval) or multi-step tool orchestration | Simple single-turn tasks — the loop overhead is pure waste if no tool calls are actually needed |

**Chain-of-thought, why it works (brief theory, not a re-derivation of the paper):** the original
Chain-of-Thought paper (Wei et al., 2022) showed that for sufficiently large models, providing (or
eliciting) intermediate reasoning steps substantially improves accuracy on tasks requiring multi-step
inference — arithmetic, symbolic manipulation, commonsense chains. The intuition: forcing the model to
"show work" token-by-token gives it more computation (more forward passes, effectively) to arrive at a
correct answer, versus jumping straight to a final token with no intermediate scratch space. This is also
why CoT interacts with temperature control (§3.5): a reasoning chain benefits from a stable, low-variance
process (`temperature=0` for factual/deterministic reasoning tasks), because you want the same problem to
produce the same reasoning path and the same answer.

**ReAct as a bridge to agents (Module 17 goes deep on this — treat this as a preview):** ReAct (Yao et al.,
2023) formalized the pattern of interleaving `Thought -> Action -> Observation -> Thought -> ...` so a
model can reason about *what it doesn't know*, take an action to find out (call a search tool, query a
database, run a calculator), observe the real result, and revise its plan. This single idea — reasoning
and acting in the same loop, grounded by real observations rather than pure imagination — is the
conceptual DNA of essentially every production agent framework in use today (tool-calling loops,
LangGraph state machines, orchestrated multi-agent systems). This module introduces it only far enough to
be a good citizen of that vocabulary; the full engineering of robust, production agent loops (error
handling, loop termination, cost bounding, human-in-the-loop escalation) is Module 17's job.

#### Architecture — the ReAct loop (preview)

```
   User query
       |
       v
  +---------+     Thought: "I need the current  +---------+
  |  Model   | --> account balance, I don't      |         |
  |  (LLM)   |     have it, I should call the    |         |
  +---------+     get_balance tool."             |         |
       |                                          |         |
       v  Action: get_balance(account_id=123)      |         |
  +---------+                                     |         |
  | Tool /   | <-----------------------------------+         |
  | External |     Observation: {"balance": 542.10}          |
  | System   | -------------------------------------------> back to Model
  +---------+
       |
       v (loop continues until model decides it has enough
          information to produce a final answer, or a bounded
          step/time/cost limit is hit — that bounding logic is
          Module 17's concern)
       v
  Final answer to user
```

#### Examples

**Beginner (zero-shot):**

```
Summarize the following email in one sentence.

Email: {email_text}
```

**Intermediate (few-shot):**

```
Rewrite each customer message in a warm, professional tone. Follow the style below exactly.

Input: "this is broken again, fix it now"
Output: "I understand this issue is frustrating, especially since it's recurring. I'm looking into a fix right away."

Input: "why does this cost so much"
Output: "That's a fair question — let me walk you through what's included in the pricing."

Input: {customer_message}
Output:
```

**Production-grade (chain-of-thought for a reasoning-heavy task, with the reasoning kept internal to
avoid exposing a possibly-misleading scratchpad to end users):**

```python
COT_INVOICE_AUDIT_PROMPT = """\
[ROLE] You are a financial auditing assistant.
[TASK] Determine whether the invoice below complies with the three policy rules given.
[CONSTRAINTS] Think through each rule one at a time before concluding. Do not skip steps.
[FORMAT] Respond with a JSON object: {{"reasoning": string, "compliant": bool, "violations": [string]}}
Only the JSON object will be shown to the end user's dashboard — reasoning is logged internally for audit,
not displayed raw, so be thorough rather than terse.

Policy rules:
1. Total must not exceed the pre-approved budget of {budget}.
2. Vendor must be on the approved vendor list.
3. Invoice date must fall within the current fiscal quarter.

Invoice: {invoice_json}
"""
```

Here CoT is elicited via the `[CONSTRAINTS]` section ("think through each rule one at a time") rather than
via few-shot demonstration, and the reasoning is captured in a structured field (`reasoning`) so it's
available for audit/debugging without being blindly surfaced as the user-facing answer — a pattern worth
internalizing: **CoT output is a debugging/audit asset, not automatically a user-facing one.**

#### Comparison: when to escalate up this ladder

```
                     Is the task simple, well-known,
                     single-step, low ambiguity?
                              |
                 yes ---------+--------- no
                  |                       |
                  v                       v
            ZERO-SHOT              Does the task need a
            (cheapest,             specific format/style/
             fastest)              edge-case pattern that's
                                    easier to show than tell?
                                          |
                                yes ------+------ no
                                 |                 |
                                 v                 v
                            FEW-SHOT         Does it require multi-step
                            (fixed extra     reasoning/deduction to
                             token cost      reach a correct answer?
                             per call)             |
                                          yes ------+------ no
                                           |                 |
                                           v                 v
                                     CHAIN-OF-THOUGHT   (re-examine: task
                                     (extra output      may be simpler
                                      tokens/latency)    than assumed —
                                           |             default to
                                           v             zero/few-shot)
                                  Does it ALSO need real
                                  external information or
                                  actions (search, tools,
                                  APIs) to answer correctly?
                                           |
                                yes -------+------- no
                                 |                    |
                                 v                    v
                         REACT / AGENT LOOP      Plain CoT is
                         (Module 17 territory —   sufficient
                          multiple round trips,
                          tool orchestration)
```

---

### 3.5 Reducing Hallucination

#### Theory

Hallucination is fluent, confident, unsupported output — and it is dangerous precisely because it is
fluent and confident: users have no signal from the text itself that it's wrong. It arises structurally
from how LLMs generate text: next-token prediction optimizes for *plausible continuation*, not for
*verified truth*, and nothing in vanilla generation stops a model from producing a plausible-sounding but
fabricated policy detail, citation, or fact when its parametric knowledge is incomplete, stale, or simply
absent for the specific case at hand.

The production techniques below do not "fix" the model — they change the **task** the model is asked to
perform, from "answer this question" (which invites confident guessing when knowledge is thin) to "answer
this question **only if you can support it from the given context, and prove it**" (which makes refusal
and evidence-citing the path of least resistance instead of confident fabrication).

| Technique | Mechanism | Reported impact (case study) |
|---|---|---|
| **Grounding** | Restrict the model to answering only from supplied context, not parametric memory | ~70% reduction in hallucination rate on its own |
| **Refusal clause** | Explicitly permit/require "I don't have that information" when context is insufficient | Prevents confident wrong answers where grounding alone still leaves ambiguity |
| **Chain-of-thought** | Reason step-by-step before answering | Highest measured factual-accuracy improvement among the techniques tested |
| **Citation / verification** | Require the model to quote the exact supporting sentence used | Makes ungrounded claims detectable/rejectable, both by the model itself and by downstream validation |
| **Temperature = 0** | Deterministic/near-greedy decoding | Does not reduce hallucination by itself, but supports stable, repeatable production behavior — a distinct but related win |

**Case study grounding this module:** a support bot was fabricating policy details it had no actual basis
for. Three rules were added — (1) grounding ("only answer from the context below"), (2) refusal ("if the
context is insufficient, reply: I don't have that information"), (3) citation ("cite the exact sentence
used") — and the hallucination rate dropped from **18% to 2.1%**. No model swap, no fine-tuning: purely a
prompt-level behavioral constraint change.

**Why "abstention rules + evidence requirements" beat "careful wording" alone:** telling a model to "be
careful" or "only say things you're sure about" is a *vibe* instruction — it doesn't structurally change
what's easiest for the model to produce. Requiring a specific refusal phrase when context is insufficient,
and requiring a quoted citation for every claim, changes the actual generation task: making something up
now requires the model to *also* fabricate a plausible-looking supporting quote, which is a harder,
more detectable failure than a bare unsupported claim. In short: **make honesty structurally easier than
guessing**, don't just ask nicely.

**When grounding/refusal is NOT the right lever:** for genuinely open-ended creative or brainstorming
tasks (where there is no "ground truth" to be grounded against), these techniques are inapplicable or
actively harmful — you'd be asking a creative-writing task to constantly second-guess itself and refuse.
Apply grounding/refusal/citation specifically to factual, evidentiary, or policy-bound tasks.

#### Architecture — grounded, refusal-capable, cited prompt

```
   +---------------------------------------------------------------+
   |                     GROUNDED PROMPT STRUCTURE                  |
   |                                                                 |
   |  "Use only the supplied context below to answer.               |
   |   If the evidence needed is missing, say:                      |
   |   'I don't know from the provided context.'                    |
   |                                                                 |
   |   Return your answer as JSON with exactly these fields:         |
   |   1. answer        <- the answer, or the refusal phrase         |
   |   2. cited_evidence <- exact quoted sentence(s) supporting it    |
   |   3. missing_information <- what would be needed if you refused"|
   |                                                                 |
   |   Context: {retrieved_context}                                  |
   |   Question: {user_question}"                                    |
   +---------------------------------------------------------------+
                              |
             +----------------+-----------------+
             |                                   |
             v                                   v
     Context sufficient                  Context insufficient
             |                                   |
             v                                   v
   answer + cited_evidence         answer = "I don't know from
   populated, missing_             the provided context",
   information = null              missing_information populated,
             |                     cited_evidence = []
             v                                   |
   +-------------------------+                    v
   | Downstream: verify each  |     +-------------------------------+
   | cited_evidence string     |     | Downstream: safe to show        |
   | actually appears in the   |     | refusal to user, or trigger a    |
   | retrieved context (cheap,  |     | fallback (escalate to human,     |
   | deterministic substring    |     | broaden retrieval, etc.)          |
   | check) -> flags residual   |     +-------------------------------+
   | hallucination even after   |
   | grounding                  |
   +-------------------------+
```

That last box matters in production: even with grounding, refusal, and citation instructions, a model can
occasionally cite a sentence that doesn't actually appear in the context, or misquote it. A cheap,
deterministic post-hoc check — does `cited_evidence` actually occur (verbatim or near-verbatim) in the
retrieved context string? — catches this class of residual hallucination without another model call, and
is a pattern worth building into any production grounded-QA system.

#### Examples

**Beginner** — bare instruction, no structural enforcement:

```
Answer the question using the context. Be accurate.

Context: {context}
Question: {question}
```

Still allows confident fabrication whenever the context is thin or absent — "be accurate" is a vibe, not a
constraint.

**Intermediate** — grounding + refusal:

```
Use only the context provided below to answer the question. Do not use outside knowledge.
If the context does not contain enough information to answer, respond exactly with:
"I don't have that information."

Context: {context}
Question: {question}
```

**Production-grade** — grounding + refusal + citation + structured output + temperature=0, wired together:

```python
from pydantic import BaseModel

class GroundedAnswer(BaseModel):
    answer: str
    cited_evidence: list[str]
    missing_information: str | None

GROUNDED_QA_PROMPT = """\
[ROLE] You are a policy-question answering assistant for internal support agents.

[TASK] Answer the question using ONLY the context provided below.

[CONSTRAINTS]
- Do not use any knowledge beyond what is in the context.
- If the context does not contain enough information to answer confidently, set "answer" to \
exactly "I don't have that information from the provided context." and explain what's missing \
in "missing_information".
- Every claim in "answer" must be traceable to a quoted sentence in "cited_evidence". \
If you cannot quote a supporting sentence for a claim, do not make that claim.

[FORMAT]
{{"answer": string, "cited_evidence": [string], "missing_information": string | null}}

Context:
{context}

Question: {question}
"""

def answer_grounded_question(client, context: str, question: str) -> GroundedAnswer:
    response = client.chat.completions.create(
        model="gpt-4o",
        temperature=0,  # deterministic, repeatable output for a factual task
        response_format={"type": "json_schema", "json_schema": {
            "name": "grounded_answer", "schema": GroundedAnswer.model_json_schema(), "strict": True
        }},
        messages=[{"role": "user", "content": GROUNDED_QA_PROMPT.format(context=context, question=question)}],
    )
    parsed = GroundedAnswer.model_validate_json(response.choices[0].message.content)

    # Cheap deterministic residual-hallucination check (see architecture diagram above):
    for quote in parsed.cited_evidence:
        if quote.strip() and quote.strip() not in context:
            # Flag for logging/monitoring — do not silently trust an uncited claim
            log_unverified_citation(question=question, quote=quote)

    return parsed
```

#### Key principles

1. **Ground**: answer only from supplied context, never parametric memory, for factual/evidentiary tasks.
2. **Enable refusal**: explicitly permit and require "I don't know" when evidence is missing.
3. **Require citation**: force quoted evidence for every claim — this is both a prompting technique and a
   verification hook for downstream code.
4. **Use temperature=0** for factual tasks: it doesn't reduce hallucination directly, but removes an
   additional axis of variance so the *same* input reliably produces the *same* (hopefully correct, or
   consistently caught) output — essential for debugging and for evaluation (Modules 09-11).
5. **Verify programmatically where possible**: a cited-evidence substring check is nearly free and catches
   a meaningful share of residual hallucination that survives even a well-grounded prompt.

---

### 3.6 Automatic Prompt Optimization (DSPy and the "when to stop hand-tuning" question)

#### Theory

Everything in §3.2-§3.5 is manual prompt engineering: a human reads failure cases, hypothesizes a fix
(add a constraint, add an example, add a refusal clause), and iterates. This works extremely well and
should remain your default mode for most tasks — it's cheap, fast to iterate, and the failure modes are
human-legible.

It starts to break down under three conditions, all common in mature LLM platforms:

1. **Multi-module pipelines.** When a prompt is one stage of a multi-stage pipeline (e.g., retrieve ->
   rerank -> reason -> generate -> verify), hand-tuning one stage's prompt can silently un-tune the
   interaction with the next stage. The combinatorial search space of "which wording, in which stage,
   interacts well with which wording in the next stage" quickly exceeds what a human can explore by hand.
2. **You have a real evaluation metric and a real dataset.** If you can compute a score (accuracy,
   groundedness, a custom LLM-judge rubric — Module 11) over dozens-to-hundreds of labeled examples, you
   have exactly the ingredients an optimizer needs, and manual iteration is now strictly worse: a human
   iterating by feel is doing an inefficient, unsystematic version of the same search a program can do
   exhaustively and reproducibly.
3. **You need to re-optimize repeatedly** — e.g., after a model upgrade/swap, wording that was
   hand-tuned for one model generation frequently under-performs on the next (different models respond to
   phrasing differently). Re-discovering good wording by hand, per model generation, does not scale; an
   optimizer can be re-run against the new target model automatically.

**What DSPy actually does (conceptually, not implementation-level):** instead of writing a prompt string,
you declare a **Signature** (typed input fields -> typed output fields, e.g., `question -> answer`), a
**Module** (a prompting strategy applied to that signature — plain predict, chain-of-thought, ReAct, etc.),
and a **metric** function. An **optimizer** (teleprompter) then searches over instruction phrasings and/or
which few-shot examples to include, using your training set and metric as the objective, and produces a
compiled, concrete prompt (or set of prompts, for a multi-module pipeline) that scores well on your metric
— all without you hand-writing the final wording. As of 2026, DSPy ships multiple optimizer strategies
(MIPROv2, SIMBA, GEPA are the commonly referenced ones; consult current DSPy docs for the currently
recommended default, since this continues to evolve) rather than a single fixed algorithm — different
optimizers trade off search cost against final quality and are suited to different dataset sizes/pipeline
shapes.

**When NOT to reach for this:** if you have a handful of examples, a single-stage prompt, and failures you
can eyeball and immediately understand, an optimization framework is pure overhead — you'd be spending
engineering time standing up an optimization harness (need a dataset, need a metric, need optimizer compute
budget) to solve a problem that a 20-minute manual iteration session solves just as well. Treat DSPy-style
optimization the way you'd treat AutoML for classical ML: an accelerant for a *system with enough
components/data to make search pay for itself*, not a default starting point.

#### Architecture — manual iteration vs. optimizer-driven compilation

```
   MANUAL PROMPT ENGINEERING LOOP                  DSPy-STYLE OPTIMIZER LOOP
   (§3.2-§3.5 techniques)                          (this section)

   +------------------+                            +---------------------------+
   | Human reads       |                            | Declare Signature:          |
   | failure cases     |                            |   question -> answer         |
   +--------+---------+                            +-------------+---------------+
            |                                                     |
            v                                                     v
   +------------------+                            +---------------------------+
   | Human hypothesizes|                           | Declare Module:              |
   | a fix (add rule,   |                           |   e.g. ChainOfThought(sig)    |
   | add example,       |                           +-------------+---------------+
   | reword)            |                                          |
   +--------+---------+                                           v
            |                                       +---------------------------+
            v                                       | Provide training set +      |
   +------------------+                            | metric function              |
   | Re-run against a  |                            +-------------+---------------+
   | few test cases,    |                                          |
   | eyeball result     |                                          v
   +--------+---------+                            +---------------------------+
            |                                       | Optimizer (MIPROv2/SIMBA/   |
            v                                       | GEPA) SEARCHES over          |
      converged?  --- no, loop again                | instructions + few-shot      |
            |                                       | demos to maximize metric     |
           yes                                       | over the training set        |
            v                                       +-------------+---------------+
   +------------------+                                            |
   |  Ship the prompt   |                                          v
   |  string as-is       |                            +---------------------------+
   +------------------+                              | Compiled, concrete prompt(s)|
                                                       | emitted — feed into your    |
                                                       | normal versioning/registry  |
                                                       | pipeline (Modules 02-03,     |
                                                       | 09) exactly like a manually  |
                                                       | written one                  |
                                                       +---------------------------+
```

Note the last box on the right: **the output of an optimizer is still just a prompt (or set of prompts)**.
It gets versioned, registered, evaluated, and gated exactly the way a hand-written prompt does — DSPy
changes *how the prompt text was produced*, not how it's operated afterward. This is why this module
treats it as an authoring-time tool that plugs cleanly into the lifecycle machinery the rest of this course
builds.

#### Example — minimal DSPy-style program sketch

```python
import dspy

class SupportTriageSignature(dspy.Signature):
    """Classify a support ticket into a category with evidence."""
    ticket_text: str = dspy.InputField()
    category: str = dspy.OutputField(desc="one of: billing, technical_issue, feature_request, account_access, other")
    evidence: str = dspy.OutputField(desc="short quoted phrase justifying the category")

classify = dspy.ChainOfThought(SupportTriageSignature)

def accuracy_metric(example, prediction, trace=None) -> bool:
    return prediction.category == example.category

# training_set: List[dspy.Example] built from historical labeled tickets (Module 10 territory)
optimizer = dspy.MIPROv2(metric=accuracy_metric)
compiled_classifier = optimizer.compile(classify, trainset=training_set)

# compiled_classifier now has an optimized instruction + selected few-shot demos baked in;
# treat its serialized prompt exactly like any other versioned prompt artifact from here on.
```

---

### 3.7 Production-Grade Prompt Template Code

Everything above composes into a single production requirement: **prompts must be code-managed
artifacts** — versioned, parameterized, testable, and reviewable — not strings pasted inline wherever
they're needed. Here is a template module shape that a CI pipeline (Module 01) can lint, a registry
(Module 03) can version, and an evaluation harness (Module 09) can score.

```python
"""
prompts/support_triage.py

A versioned, testable prompt template. Notice this module has no I/O and no
provider-specific code — it is pure string/schema logic, which makes it trivially
unit-testable and reusable across providers.
"""
from __future__ import annotations
from dataclasses import dataclass
from pydantic import BaseModel, Field

PROMPT_VERSION = "support_triage.v3"  # bump on any semantic change; Module 03/09 track this

class TicketClassification(BaseModel):
    category: str = Field(description="One of: billing, technical_issue, feature_request, account_access, other")
    confidence: float = Field(ge=0.0, le=1.0)
    evidence: list[str]
    follow_up_needed: bool

_ROLE = (
    "You are a support-ticket triage classifier for a B2B SaaS company. "
    "You have deep familiarity with common billing, technical, and account-access issues."
)

_TASK = (
    "Classify the support ticket below into exactly one of these categories: "
    "billing, technical_issue, feature_request, account_access, other."
)

_CONSTRAINTS = (
    "Choose exactly one category. Never invent a category not in the list above. "
    "If the ticket could fit two categories, choose the more specific one. "
    "Do not include any explanation outside the JSON object."
)

_FEW_SHOT_EXAMPLES = [
    (
        "I was charged twice this month for my subscription, please refund one of them.",
        '{"category": "billing", "confidence": 0.97, "evidence": ["charged twice this month"], "follow_up_needed": false}',
    ),
    (
        "The export button does nothing when I click it, no error message either.",
        '{"category": "technical_issue", "confidence": 0.9, "evidence": ["export button does nothing"], "follow_up_needed": true}',
    ),
]

@dataclass(frozen=True)
class RenderedPrompt:
    version: str
    text: str
    schema: dict

def render(ticket_text: str) -> RenderedPrompt:
    """Pure function: same input always produces the same prompt text. Testable
    with a plain equality assertion, no mocking required."""
    examples_block = "\n\n".join(
        f'Ticket: "{inp}"\nOutput: {out}' for inp, out in _FEW_SHOT_EXAMPLES
    )
    schema = TicketClassification.model_json_schema()
    text = (
        f"[ROLE]\n{_ROLE}\n\n"
        f"[TASK]\n{_TASK}\n\n"
        f"[CONSTRAINTS]\n{_CONSTRAINTS}\n\n"
        f"[FORMAT]\nRespond with JSON matching this schema:\n{schema}\n\n"
        f"[EXAMPLES]\n{examples_block}\n\n"
        f'Ticket: "{ticket_text}"\nOutput:'
    )
    return RenderedPrompt(version=PROMPT_VERSION, text=text, schema=schema)
```

```python
"""
tests/test_support_triage_prompt.py

Unit tests a CI pipeline (Module 01) runs on every commit that touches the
prompt module — no LLM call required for these; they test the deterministic
rendering logic, which is where most silent regressions actually happen
(a typo in an f-string, a dropped section, a schema/prompt mismatch).
"""
from prompts.support_triage import render, PROMPT_VERSION

def test_prompt_contains_all_five_sections():
    rendered = render("test ticket")
    for section in ("[ROLE]", "[TASK]", "[CONSTRAINTS]", "[FORMAT]", "[EXAMPLES]"):
        assert section in rendered.text

def test_prompt_version_is_pinned():
    assert rendered_version_matches_registry(PROMPT_VERSION)  # ties into Module 03's registry

def test_prompt_is_deterministic_given_same_input():
    a = render("same ticket text")
    b = render("same ticket text")
    assert a.text == b.text
```

This is the concrete bridge between "prompt engineering" (this module) and "prompt lifecycle management"
(Module 09): a prompt that lives as a pure, versioned, unit-testable Python module is something CI/CD,
registries, and evaluation harnesses can all operate on mechanically — a prompt that lives as an inline
f-string scattered across API call sites is not.

---

## 4. Real-World Case Studies (Reasoned Inference)

None of the specifics below are disclosed internal implementation details of these companies — they are
reasoned inferences about how a system with these publicly known constraints would plausibly be built,
based on general engineering-blog patterns and industry norms as of mid-2026.

- **A frontier model provider's own developer-facing documentation (OpenAI, Anthropic) as a signal of
  internal practice.** Both companies publish extensive prompting best-practice guides that closely mirror
  this module's structure (role/system separation, explicit constraints, XML/section tagging, few-shot
  demonstration, chain-of-thought elicitation, schema-guaranteed structured output). It is reasonable to
  infer that internal prompt engineering for their own first-party products (customer support tooling,
  internal coding assistants) follows the same discipline they document publicly — a company that
  publishes "define your output format explicitly" as guidance for external developers would plausibly
  apply the identical discipline internally, likely enforced through internal prompt-linting or template
  libraries analogous to §3.7's pattern.

- **A large-scale customer support operation (a company like Amazon or a major SaaS platform) building an
  LLM-based ticket triage/response system** would plausibly combine this module's techniques end to end:
  a structured 5-section prompt for classification (mirroring the 71%->96% case study), grounding + refusal
  + citation for any response-drafting step that references policy documents (mirroring the 18%->2.1%
  hallucination case study), and a hard budget on retrieved-policy-document context sized against a
  known system-prompt-plus-reserve overhead — with token-usage-ratio logging (per the transcript's "log
  usage" principle) feeding into per-ticket cost dashboards that a platform team like this would need at
  the request volumes involved.

- **A code-assistant product (a company like GitHub/Microsoft, or a coding-agent vendor)** would plausibly
  rely heavily on the ReAct-style loop bridged in §3.4 — reasoning about what code context is needed,
  taking an action (read a file, run a search, run tests), observing the result, and revising — precisely
  because static, single-shot prompting cannot incorporate real repository state; this is architecturally
  why coding agents are ReAct-shaped rather than plain chain-of-thought, and why Module 17 exists as a
  dedicated module rather than being folded into this one.

- **A media/entertainment recommendation-adjacent company (a company like Netflix or Spotify) using LLMs
  for content metadata generation or conversational search** would plausibly apply strict grounding and
  structured-output enforcement for anything customer-facing (metadata must be schema-valid to flow into
  existing catalog systems — an unstructured or malformed LLM output would break a downstream pipeline the
  same way a malformed row would break an ETL job), while reserving looser, higher-temperature, less
  constrained prompting for internal experimentation/generation tasks with a human review step before
  anything reaches production catalogs.

- **A platform/tooling company (a company like Databricks) offering LLMOps tooling to its own customers**
  would plausibly treat this module's entire toolkit as a product surface rather than an internal
  practice only — i.e., building first-class support for prompt versioning, structured-output schema
  enforcement, and evaluation-metric tracking directly into their platform (mirroring how Module 09-13 of
  this course treat prompt lifecycle, evaluation datasets, and experiment tracking as platform-level
  concerns, not one-off scripts), since their customers face exactly the manual-iteration-vs-automatic-
  optimization tradeoff described in §3.6 across many independent teams simultaneously.

---

## 5. Common Mistakes

1. **Sending "all the retrieved chunks" instead of budgeting.** No token accounting before assembly leads
   to either hard errors or silent truncation of exactly the most relevant content (§3.1).
2. **Placing the most important content at the start of a long prompt "because that's how documents are
   normally written."** This ignores recency bias — the most relevant/most recently retrieved content
   should typically sit closest to the model's generation point, not buried early in a long context.
3. **Treating "JSON mode" as a schema guarantee.** It guarantees parseable JSON, not correct fields/types —
   teams that skip provider-native Structured Outputs or client-side Pydantic validation discover this the
   hard way when a required field is silently missing (§3.3).
4. **Writing constraints as vibes instead of rules** ("be careful," "try to be accurate," "please don't
   make things up") instead of structural constraints (explicit refusal phrase, required citation format).
   Vibes do not change what's easiest for the model to generate; rules do (§3.5).
5. **Using few-shot examples that are too similar to each other**, covering no edge cases, which fails to
   anchor the model's behavior on the ambiguous inputs that actually cause production failures.
6. **Reaching for temperature=0 as an anti-hallucination fix on its own.** It stabilizes output, it does
   not make ungrounded content true — teams that treat determinism as equivalent to correctness get a
   perfectly reproducible wrong answer every time (§3.5).
7. **Standing up a DSPy-style optimization pipeline for a single hand-tunable prompt with no eval set.**
   This is pure overhead when a 20-minute manual iteration session would have solved it (§3.6).
8. **No prompt versioning** — editing an inline prompt string directly in application code with no version
   tag, no test, no changelog, making regressions in production behavior nearly impossible to bisect (§3.7,
   and the entire premise of Module 09).
9. **Forgetting to reserve output tokens.** A budget that spends every available token on input context
   leaves the model with no room to answer, producing truncated or empty responses under exactly the
   conditions (long/complex queries) where a full answer matters most.
10. **Not re-deriving the token budget on model swap.** Different model generations have different context
    windows and different system-prompt token costs (tokenizer changes too) — a budget hard-coded against
    one model's limits silently becomes wrong after a migration.

---

## 6. Best Practices and Production Tips

**When to use manual prompt engineering vs. structured output enforcement vs. automatic optimization:**

- Default to manual iteration with the 5-section structure for any new task — it is fast, cheap, and
  human-debuggable.
- Add provider-native structured output / Pydantic validation as soon as *any* downstream code parses the
  response — treat this as non-negotiable for production, not an optional hardening step.
- Reach for automatic optimization (DSPy-style) only once you have a real metric, a real dataset (dozens
  to hundreds of examples), and either a multi-module pipeline or a recurring need to re-tune across model
  swaps.

**Cost and scaling:**

- Every token in the system prompt and every few-shot example is a fixed per-request cost, multiplied by
  volume — audit your system prompt's token count the same way you'd audit a hot-path function's
  algorithmic complexity.
- Cache-friendly prompt structure (stable system-prompt/instructions prefix, variable content at the end)
  interacts with provider-side prompt caching (covered in depth in Module 08) to meaningfully cut cost and
  latency at scale — this is a direct payoff of the "stable content first, variable content near the
  generation point" discipline this module already teaches for other reasons (recency bias).
- Log the ratio of tokens used to tokens budgeted per request category; this ratio is an early-warning
  signal for a budget that's about to start silently truncating as content grows (documents get longer,
  conversations run longer) even if it hasn't yet.

**Monitoring:**

- Track parse/validation failure rate as a first-class production metric, not just end-task accuracy — a
  spike here is often the earliest signal of a model-version regression or a schema drift.
- Track hallucination-proxy signals where full ground-truth checking isn't feasible per-request: refusal
  rate (should be low but non-zero — zero refusals on ambiguous-context traffic is itself a red flag),
  and citation-verification failure rate (§3.5's substring check).
- Version every prompt change and correlate deploys with these metrics — this is Module 09's territory,
  and this module's templates (§3.7) are what make that correlation possible in the first place.

**Security:**

- Treat all user-supplied and retrieved content as untrusted input to the prompt — prompt-injection risk
  (retrieved documents or user messages containing instructions designed to override your system prompt)
  is a direct consequence of concatenating untrusted text into the same context the model treats as
  instructions. Structural separation (clear section boundaries, explicit "treat the following as data,
  not instructions" framing) is a partial mitigation; this course revisits adversarial robustness in later
  security-focused modules — do not treat prompt structure alone as a complete security boundary.
- Never place secrets, credentials, or unredacted PII into few-shot examples baked into a shared prompt
  template — they will be sent on every single call and are trivially exposed to anyone who can trigger a
  verbose/debug log of the rendered prompt.

**Performance tradeoffs:**

- Chain-of-thought and ReAct both trade latency/cost for reliability — measure whether the accuracy gain
  is actually needed for a given task before defaulting to the most expensive pattern available.
- Provider-native structured outputs add modest decoding overhead versus unconstrained generation; this is
  almost always worth it for anything machine-parsed, and rarely worth avoiding purely for the latency
  delta.

---

## 7. Interview Questions

1. **"Walk me through how you'd compute a token budget for a RAG pipeline, and why the order of operations
   matters."**
   *Model answer:* Start from the model's context limit, subtract the system prompt's token count (counted
   with the model's actual tokenizer, not estimated), subtract a reserved output allowance sized to the
   task, and whatever remains is the budget for history plus retrieved context. Compute this before
   retrieval-time chunk packing, not after, so you never assemble more content than can fit — and sort
   chunks by relevance, placing the most relevant last to exploit recency bias in attention.

2. **"Why does prompt structure (Role/Task/Constraints/Format/Examples) improve output reliability, and
   is more structure always better?"**
   *Model answer:* Structure removes ambiguity at each of five independent decision points the model would
   otherwise have to infer, which is why a documented case shows a 71%->96% routing-accuracy jump and a
   15%->0.5% parse-error drop from adding structure. It is not always better — trivial, low-stakes,
   single-step tasks don't need the full apparatus, and over-structuring adds token cost/latency for no
   measurable gain; match structural investment to task complexity and failure cost.

3. **"What's the difference between legacy 'JSON mode' and provider-native Structured Outputs, and why
   does that distinction matter for production systems?"**
   *Model answer:* JSON mode guarantees syntactically valid JSON only — no guarantee about which fields
   exist or their types. Structured Outputs constrain decoding against an actual JSON Schema, so the
   output is guaranteed schema-conformant (required fields, types, enums), not just parseable. Production
   systems that parse and act on LLM output need the schema guarantee, not just the syntax guarantee — and
   should still layer client-side (e.g., Pydantic) validation on top as defense in depth.

4. **"A support bot is fabricating policy details. Walk me through the prompt-level fix, and explain why
   it works better than just telling the model to 'be careful.'"**
   *Model answer:* Add three structural rules: grounding (answer only from supplied context), a refusal
   clause (explicit permitted phrase for insufficient context), and a citation requirement (quote the
   exact supporting sentence). This is documented to drop hallucination rate from roughly 18% to 2.1% in a
   real case. It works better than "be careful" because it changes the actual generation task — making
   something up now requires fabricating a plausible citation too, which is harder and more detectable,
   whereas "be careful" doesn't change what's easiest for the model to produce next.

5. **"When would you reach for an automatic prompt-optimization framework like DSPy instead of manually
   iterating on a prompt?"**
   *Model answer:* When you have a computable evaluation metric and a training/validation dataset of
   meaningful size (dozens to hundreds of examples), when the prompt is one stage of a multi-module
   pipeline where hand-tuning one stage risks silently breaking another, or when you need to repeatedly
   re-optimize after model swaps. For a simple, single-stage, human-legible-failure-mode task, manual
   iteration remains faster and cheaper — automatic optimization is an accelerant for search-worthy
   problems, not a default starting point.

6. **"Explain chain-of-thought prompting and one scenario where you would deliberately NOT show the
   reasoning to the end user despite eliciting it from the model."**
   *Model answer:* CoT asks the model to produce intermediate reasoning steps before its final answer,
   which improves accuracy on multi-step reasoning/arithmetic/logic tasks by giving the model more
   effective "space" to work through the problem before committing to a final token. You'd hide the raw
   reasoning from end users in something like a compliance/audit tool where the reasoning trace is useful
   internally for debugging/audit but might be verbose, contain intermediate incorrect hypotheses, or
   otherwise mislead a non-expert end user if shown verbatim — structure the output so reasoning is a
   separate, internally-logged field rather than part of the user-facing answer.

7. **"How would you design multi-turn conversation truncation for a long-running chat session?"**
   *Model answer:* Always preserve the system prompt (defines behavior) and the last N turns (carries
   active intent) as non-negotiable floors. Beyond that floor, trim the oldest messages first, recomputing
   token counts against the current budget each turn since history length is inherently dynamic. For
   sessions where early context has durable value beyond what a fixed window can hold, consider
   summarization/compaction of older turns instead of outright dropping them, at the cost of an extra LLM
   call.

8. **"What's the relationship between this module's prompt-engineering techniques and ReAct/agent
   architectures covered later in the course?"**
   *Model answer:* ReAct interleaves reasoning (Thought) with tool calls (Action) and real tool results
   (Observation) in a loop, which is the direct conceptual ancestor of production agent frameworks. This
   module treats it as a bridging concept — the pattern matters here because it's a natural escalation
   from chain-of-thought when a task needs real external information, not just internal reasoning — while
   the full engineering concerns (loop termination, cost bounding, error handling, multi-agent
   orchestration) belong to the dedicated agents module later in the course.

---

## 8. Summary, Key Takeaways, and Production Checklist

### Summary

Prompt engineering, treated as an MLOps/LLMOps discipline rather than a creative-writing exercise, is the
practice of engineering the interface contract between deterministic application code and a
non-deterministic model call. This module covered five compounding techniques: (1) explicit token
budgeting so you send the *right* text rather than just *more* text; (2) structured, section-based prompts
that measurably improve accuracy and reduce parse failures; (3) schema-enforced structured output, layered
across prompt/provider/client for defense in depth; (4) an escalating ladder of prompting patterns
(zero-shot -> few-shot -> chain-of-thought -> ReAct) matched to task complexity; and (5) grounding,
refusal, and citation techniques that make honesty structurally easier than confident fabrication.

### Key Takeaways

- **Budget first, retrieve/pack second.** Never assemble prompt content before computing what fits.
- **Recency bias is real and exploitable** — place the most important content closest to the generation
  point.
- **Structure is one of the cheapest, highest-leverage interventions available** — no fine-tuning, no new
  infra, just organized instructions, with numbers to prove it (71%->96%, 15%->0.5%).
- **"JSON mode" is not a schema guarantee** — use provider-native Structured Outputs plus client-side
  validation for anything production code parses.
- **Match the prompting pattern to the task** — don't default to the most expensive technique (CoT, ReAct)
  when zero-shot or few-shot suffices.
- **Reduce hallucination with structural rules, not politeness** — grounding, refusal clauses, and
  citation requirements change the generation task itself (18%->2.1% in the case study).
- **Automatic optimization (DSPy) is an accelerant for search-worthy problems**, not a default starting
  point — most tasks are still best served by manual iteration.
- **Prompts are versioned code artifacts**, not inline strings — this is the load-bearing bridge into
  Module 09's prompt lifecycle management.

### Production Checklist

- [ ] Token budget explicitly computed (model limit - system prompt - output reserve) before any
      retrieval/history packing.
- [ ] Tokens counted with the actual model's tokenizer (`tiktoken` or provider-equivalent), never estimated.
- [ ] Most relevant/most recent content placed closest to the model's generation point.
- [ ] Multi-turn truncation preserves system prompt and last N turns; trims oldest history first.
- [ ] Every production prompt uses the 5-section structure (Role/Task/Constraints/Format/Examples) where
      task complexity/stakes justify it.
- [ ] Output schema defined as a Pydantic model / JSON Schema, enforced via provider-native Structured
      Outputs where supported, and validated client-side regardless.
- [ ] Prompting pattern (zero-shot/few-shot/CoT/ReAct) matched deliberately to task complexity, not
      defaulted to the most expensive option.
- [ ] Factual/evidentiary tasks include grounding instructions, an explicit refusal clause, and a citation
      requirement; citations spot-checked against source context programmatically.
- [ ] Temperature set to 0 (or near-zero) for factual/deterministic tasks.
- [ ] Automatic prompt optimization considered only where a metric + dataset + multi-module complexity (or
      recurring re-tuning need) justifies the overhead.
- [ ] Prompts live as versioned, unit-tested code modules (per §3.7), not inline strings — with a defined
      version tag feeding into the registry/lifecycle machinery of Modules 03 and 09.
- [ ] Token-usage ratio, parse/validation failure rate, refusal rate, and citation-verification failure
      rate are all logged/monitored in production.
- [ ] Untrusted content (user input, retrieved documents) is structurally separated from instructions to
      reduce prompt-injection risk; no secrets/PII embedded in shared few-shot examples.

---

## 9. Further Reading

This module intentionally keeps external references out of the main text so the teaching material stays
focused. The full, curated set of official documentation, papers, repositories, videos, and books —
including the OpenAI/Anthropic official prompting docs, the Chain-of-Thought and ReAct papers, DSPy's
paper and repository, Instructor, tiktoken, and the DAIR.AI Prompt Engineering Guide — is catalogued with
summaries, difficulty ratings, and estimated reading times in this same folder:

- `references.md` — official docs, papers, and version/currency notes for mid-2026
- `videos.md` — curated video courses and talks
- `books.md` — relevant books and chapters
- `github.md` — repositories worth studying or using directly
