# Prompt Engineering Fundamentals — Scenario-Based Q&A

**Situation:** A support-ticket classifier's prompt stuffs every retrieved policy document into the context "just to be safe," and the team notices the model increasingly forgets to apply a rule that's clearly present in the prompt. What would you do and why?

Model answer: Diagnose this as recency bias combined with no token budgeting — transformer attention empirically weights tokens closer to the generation point more heavily, so a rule buried early in a long, unbudgeted context is easy for the model to underweight, and dumping every retrieved chunk in without accounting for relevance also dilutes the prompt with low-value content. Introduce an explicit token budget that allocates space across system prompt, retrieved context, and reserved output, switch from "dump everything" to relevance-sorted packing that fits the highest-relevance chunks first, and reposition the most load-bearing instructions/context closer to the end of the prompt, nearer the generation point, rather than assuming "written first" means "weighted most."

---

**Situation:** A team ships a JSON-mode-enabled extraction prompt and is confident the output is safe to parse directly into their database, but a downstream job starts failing on missing required fields. What would you do and why?

Model answer: Explain the JSON-mode misconception directly: "JSON mode" (or a `response_format: json_object` flag) guarantees syntactically valid, parseable JSON — it does not guarantee the correct fields, types, or that the schema was actually followed. Fix by moving to provider-native Structured Outputs (schema-constrained decoding) where available, or, failing that, function/tool-calling with the schema defined as the tool's parameters, and add client-side Pydantic (or equivalent) validation on every response before it reaches the database regardless of which structuring mechanism is used — treat the API's "valid JSON" guarantee as necessary, not sufficient.

---

**Situation:** An engineer complains that a carefully worded prompt saying "please be very careful and try not to make things up" isn't reducing hallucinations at all in a RAG-based assistant. What would you do and why?

Model answer: Point out that this is a "vibes instead of rules" constraint — phrases like "be careful" or "try to be accurate" don't structurally change what's easiest for the model to generate, because they give it no concrete decision procedure. Replace it with structural constraints: a grounding instruction restricting the model to answer only from supplied context, an explicit refusal clause permitting (and requiring) "I don't have that information" when context is insufficient, and ideally a citation requirement forcing the model to quote the supporting sentence for each claim, which makes ungrounded assertions both harder to produce and easier to detect downstream. Measure the actual before/after hallucination rate on a labeled eval set rather than trusting that the wording change worked.

---

**Situation:** A team standing up their first few-shot prompt picks five examples that are all near-identical minor variations of the easiest, most common case, and is surprised when the model still fails badly on ambiguous production inputs. What would you do and why?

Model answer: Identify the root cause as examples that are too similar to each other, covering no edge cases — few-shot examples work by anchoring the model's behavior on the patterns shown, so a set with no diversity gives it nothing to generalize from for the ambiguous inputs that actually cause production failures. Rebuild the few-shot set deliberately to include at least one or two edge cases or boundary conditions alongside the common case, sourced from real production near-misses if available rather than invented examples, and re-evaluate specifically on a held-out set of hard/ambiguous inputs rather than only on easy ones.

---

**Situation:** A prompt engineer sets temperature to 0 on a RAG assistant and reports to their manager that "hallucinations are now fixed since the output is deterministic." What would you do and why?

Model answer: Correct the framing: temperature=0 makes output stable and reproducible, it does not make ungrounded content true — a deterministic wrong answer is still wrong, every time, which is arguably worse for debugging since it looks consistent. Clarify that temperature=0 is valuable for a different reason (stable, repeatable production behavior, easier regression testing) and should be combined with, not substituted for, actual hallucination-reduction techniques — grounding, refusal clauses, chain-of-thought, and citation requirements — which is where the real accuracy improvement comes from. Ask for the eval-set faithfulness/hallucination-rate numbers before and after, since "it looks deterministic now" is not evidence of correctness.

---

**Situation:** A team migrates from GPT-4-class models to a newer model generation with a larger context window, and shortly after, their long-running chat product starts returning truncated or oddly empty responses under heavy multi-turn conversations. What would you do and why?

Model answer: Recognize this as a stale token budget after a model swap — different model generations have different context windows, different system-prompt token costs, and sometimes different tokenizers entirely, so a budget hard-coded against the old model's limits silently becomes wrong after migration, and if the budget over-allocates to input, it can leave too little reserved output space, producing truncation. Re-derive the token budget explicitly for the new model (context window size, tokenizer, reserved output allocation) as a required step of any model migration, and add an automated check that fails CI if a deployed prompt template's worst-case token usage exceeds a safety margin under the currently configured model.

---

**Situation:** A junior engineer edits a live prompt string directly in application code to fix a bug quickly, ships it, and a week later a completely unrelated production regression can't be traced back to what changed. What would you do and why?

Model answer: Name the missing practice directly: no prompt versioning — an inline, unversioned prompt string edited in place destroys the ability to bisect regressions, since there's no version tag, no changelog, and no record of what the prompt looked like before. Require prompt templates to be treated as versioned, production code artifacts (a parameterized, tracked unit with a version tag and changelog, as covered in Module 09's prompt-lifecycle practices), not ad hoc strings edited inline. As immediate remediation, reconstruct the prompt's prior state from version control or logs if possible, and going forward gate any prompt change behind the same review and versioning discipline as a code change.

---

**Situation:** A team wants to squeeze more accuracy out of a well-performing internal classification prompt and proposes standing up a DSPy-based automatic prompt-optimization pipeline before even trying manual iteration. What would you do and why?

Model answer: Push back on the ordering: standing up an optimization framework for a single hand-tunable prompt with no eval set in place yet is pure overhead — a 20-minute manual iteration session against a handful of representative examples would likely solve it faster and more transparently. Recommend building a small labeled eval set first (even 30-50 examples), doing quick manual prompt iteration against it, and only reaching for DSPy-style automatic optimization once there's a real, measurable eval loop and either the search space is large enough or the team needs to re-optimize repeatedly across models/versions — DSPy is a force multiplier on an existing eval loop, not a substitute for having one.

---

**Situation:** A ReAct-style agent prompt is being used for a simple, single-turn FAQ-answering task, and a stakeholder asks why response latency is so much higher and less predictable than a comparable non-agentic assistant. What would you do and why?

Model answer: Point out the mismatch between technique and task: ReAct's Thought/Action/Observation loop is designed for tasks requiring real-world information the model doesn't have or genuine multi-step tool orchestration, and its cost is multiple model round-trips plus tool-call latency per turn — for a simple single-turn task with no tool calls actually needed, that loop overhead is pure waste. Recommend a plain, direct prompt (zero-shot or few-shot, no ReAct loop) for the FAQ case, and reserve ReAct-style looping for tasks that genuinely need external lookups or multi-step tool use, matching prompting-pattern complexity to actual task complexity rather than defaulting to the most sophisticated available pattern.

---

**Situation:** A code-generation feature and an FAQ-answering feature share the same prompt template's `max_tokens` cap, and engineers notice generated code getting cut off mid-function while FAQ answers are needlessly slow to fully generate even when short. What would you do and why?

Model answer: Identify a uniform output-length cap applied across task types with very different natural output-length distributions as the root issue — a cap tuned to be "safe" for FAQ answers will truncate and corrupt legitimate long-form outputs like generated code, while the same cap can be needlessly generous (or reserve unnecessary token budget) for short answers. Set `max_tokens` per task type based on each task's realistic output-length distribution rather than one global value, and specifically validate that the code-generation cap is large enough to cover the 95th-percentile realistic function/file length observed in real usage, not just typical cases.

---

**Situation:** A prompt engineer building an extraction pipeline places the user's variable input question before the static system instructions in every request, and later can't figure out why provider-side prompt caching never seems to reduce their latency or cost. What would you do and why?

Model answer: Explain that prompt/context caching on the provider side works by caching a matching prefix, so interleaving static and dynamic content — or putting the variable content before the static content — silently defeats caching regardless of provider, since the "prefix" that would need to match across requests keeps changing. Restructure the prompt so all static content (system instructions, few-shot examples, stable context) forms a stable, unchanging prefix, with variable user input appended at the end, and verify the fix by checking the provider's cache-hit telemetry before and after, rather than assuming the reordering worked from response time alone.

---

**Situation:** A team building a customer-facing assistant writes a structured prompt with a Role and a Task section but no explicit Constraints or Format sections, reasoning "the model will figure out reasonable behavior." Production logs later show wildly inconsistent output shapes across similar requests. What would you do and why?

Model answer: Point to the incomplete structured-prompt pattern as the cause: Role and Task alone tell the model who to act as and what to do, but leave "what must it never do" and "what shape must the output take" entirely to the model's own judgment, which is exactly where inconsistency creeps in across similar-but-not-identical requests. Add explicit Constraints (hard rules — never invent values, never explain reasoning in the output, etc.) and a Format section (an exact schema or shape specification, backed by structured-output enforcement rather than prose description alone), and validate consistency improvement with a small eval sampling several near-duplicate inputs and checking output-shape variance before/after.

---

**Situation:** A team debugging a subtly wrong classification prompt discovers that adding "think step by step before answering" improved accuracy substantially on their internal eval set, and a stakeholder asks whether they should apply this to every prompt in the product going forward. What would you do and why?

Model answer: Explain that chain-of-thought reliably helps on multi-step reasoning tasks but isn't a universal accuracy lever — for simple, well-known tasks the base model already handles well (straightforward sentiment or extraction), CoT mainly adds latency and token cost with little to no accuracy benefit, and for some latency-sensitive product surfaces that cost isn't worth paying. Recommend applying CoT selectively, based on measured lift on each specific prompt's own eval set (not a blanket policy), and where CoT is used but the reasoning trace itself shouldn't be shown to the end user, ensure the prompt/response-handling separates the reasoning from the final answer rather than exposing raw chain-of-thought in the product UI.

---

**Situation:** A model-migration eval shows a new model performing worse on a production extraction task, and the team's first hypothesis is that the new model is simply less capable. What would you do and why?

Model answer: Before accepting "the model is worse," rule out prompt-related confounds first: check whether the structured-output mechanism (JSON mode vs. function-calling vs. provider-native Structured Outputs) behaves identically on the new model, whether the token budget was re-derived for the new model's context window and tokenizer, and whether prompt-caching prefix ordering still holds for the new provider's caching implementation. Many apparent "model got worse" regressions after a migration are actually prompt-infrastructure assumptions baked in for the old model silently breaking on the new one — isolate the prompt-versus-model variable explicitly (same prompt, both models, same eval set) before concluding the new model itself underperforms.

---

**Situation:** A stakeholder, alarmed by a viral story about a competitor's chatbot hallucinating a policy, asks the team to "just tell the model not to hallucinate" as the fix for their own assistant. What would you do and why?

Model answer: Reframe the ask in terms of what actually moves the needle: a bare "don't hallucinate" instruction is a vibes-based constraint with little effect, but the module's measured techniques are concrete and combinable — grounding (restricting answers to supplied context) delivers the largest single reduction on its own, a refusal clause prevents confident wrong answers when context is genuinely insufficient, and chain-of-thought and citation/verification requirements add further, measurable improvement, with temperature=0 supporting stable (not more accurate) behavior. Propose implementing grounding plus a refusal clause first as the highest-leverage change, measure the faithfulness/hallucination-rate delta on a labeled eval set, and layer in citation requirements next if the gap remains.

---

**Situation:** An engineer wants to add a "recent conversation history" feature to a chat product and is deciding how to handle context that grows unbounded over a long session. What would you do and why?

Model answer: Walk through the context-management strategy tradeoffs rather than reaching for the simplest option by default: hard truncation is a last-resort safety net only (it cuts mid-sentence or mid-JSON with no structural awareness), oldest-first trimming works well for most multi-turn chat where recent turns matter most but silently drops early turns that may have set constraints that must persist, and summarization/compaction preserves durable value from early context at the cost of an extra LLM call per compaction. Recommend oldest-first trimming as the default for this chat product, with any long-term-important instructions pinned permanently into the system prompt (a sliding window with sticky system prompt) rather than relying on them surviving in rolling history, and reserve summarization for sessions that are unusually long-running and where early context has genuine durable value.

---

**Situation:** A team's few-shot extraction prompt works great in testing but starts leaking what looks like sensitive customer data patterns into outputs for unrelated customers after launch. What would you do and why?

Model answer: Investigate whether the few-shot examples embedded in the prompt template themselves contain real customer data or PII-shaped content that's scarce enough or distinctive enough for the model to pattern-match against and leak into unrelated outputs — the module flags "examples might leak sensitive/PII data into every prompt" explicitly as a case where few-shot is the wrong tool. Replace any real-customer-derived examples with synthetic or thoroughly anonymized ones that preserve the structural pattern being taught without carrying real sensitive content, and audit whether few-shot is even the right technique here versus a zero-shot instruction with a strict output-format constraint, especially for a task class (PII-adjacent extraction) where examples are inherently risky to embed in every request.

---

**Situation:** A junior engineer, reviewing a colleague's PR, sees a prompt template hard-coding "You are ChatGPT, a helpful assistant" as the Role section for a specialized internal fraud-triage tool, and isn't sure if that's a problem worth raising. What would you do and why?

Model answer: Flag it as a real, fixable issue: a generic, mismatched Role section gives the model an unhelpfully broad persona for a task that needs precise, narrow behavior — the Role section's actual job is to answer "who should the model act as" for this specific task, and a vague or wrong role wastes the highest-leverage, cheapest part of a structured prompt. Rewrite it to something like "You are a fraud-ticket triage classifier for a B2B SaaS company," paired with a concrete Task, explicit Constraints, and a Format section, and treat this kind of review comment as exactly the low-cost, high-value catch that a prompt-focused code review should surface routinely.

---

**Situation:** A team's chatbot occasionally returns a perfectly formatted, confident, completely fabricated citation to a document that doesn't exist in their knowledge base. What would you do and why?

Model answer: Recognize this as hallucination surviving even with some grounding in place — likely the citation requirement exists in name but isn't being verified, or grounding is present but not strict enough to prevent the model from inventing a plausible-looking reference when its retrieved context is thin. Add a verification step requiring the model to quote the exact supporting sentence (not just a document name), and add a post-generation validation step that programmatically checks the quoted citation actually exists in the retrieved context set before the response is shown to the user, rejecting or flagging responses whose citations can't be verified — moving hallucination detection from "hope the prompt prevents it" to a checkable, enforced step in the pipeline.
