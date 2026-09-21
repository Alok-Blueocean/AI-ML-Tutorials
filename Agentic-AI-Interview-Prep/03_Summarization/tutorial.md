# Summarization — Condensed Study Notes

## Extractive vs Abstractive Summarization

- **Extractive summarization** selects and stitches together existing sentences/phrases from the source text verbatim. Simple, cannot hallucinate new facts (it can only mis-select), but often reads as disjointed and can't compress ideas the way a human paraphrase would.
- **Abstractive summarization** generates new sentences that paraphrase and compress the source, the way modern LLMs summarize by default. Reads much more naturally and can compress far more aggressively, but introduces a real hallucination risk — the model can state something fluent and plausible that isn't actually supported by the source.
- Real-world example: a legal-discovery tool uses extractive summarization for case-file triage because a lawyer needs to trust that every sentence shown is a verbatim, sourced quote; a news-digest app uses abstractive summarization because readability and compression matter more than verbatim traceability.

## Single-Document vs Multi-Document vs Long-Context Summarization

- **Single-document summarization**: one source, fits comfortably in the model's context window — the simplest case, mostly a prompting problem.
- **Multi-document summarization**: synthesizing a summary across several related sources (e.g., 50 customer reviews, or 10 news articles about the same event), which requires reconciling overlapping, complementary, or contradictory information, not just compressing one stream.
- **Long-context summarization**: content that exceeds even a large model's context window (a multi-hundred-page report, a year of meeting transcripts). Two classic strategies:
  - **Map-reduce**: summarize each chunk independently ("map"), then summarize the collection of chunk-summaries into a final summary ("reduce"). Parallelizable and scales to arbitrary length, but can lose cross-chunk connections (a fact that only makes sense combining chunk 3 and chunk 9 may get lost).
  - **Refine**: summarize the first chunk, then iteratively feed the running summary plus the next chunk back to the model to produce an updated summary, one chunk at a time. Preserves more cross-chunk continuity than map-reduce, but is sequential (slower, not parallelizable) and can drift or overweight later chunks (recency bias).

```python
# Map-reduce summarization loop (conceptual)
chunk_summaries = [summarize(chunk) for chunk in chunks]          # map (parallelizable)
final_summary = summarize("\n\n".join(chunk_summaries))            # reduce

# Refine loop (conceptual) — sequential
summary = summarize(chunks[0])
for chunk in chunks[1:]:
    summary = refine(existing_summary=summary, new_chunk=chunk)
```

- Real-world example: a compliance team summarizing a 400-page regulatory filing used map-reduce for speed on the first pass, but found refine produced a more coherent narrative for the final executive summary because obligations mentioned in early sections and referenced again in later sections stayed connected in the running summary.
- Larger context windows (some current models support well over 100K, even 1M+ tokens) reduce how often map-reduce/refine is *necessary* for moderate-length single documents, but they don't eliminate the need for it — truly massive multi-document corpora (legal discovery, multi-year transcript archives) still exceed even very large windows, and evidence shows long single-pass generation itself degrades in faithfulness toward the end of very long outputs, so chunked or agent-orchestrated approaches remain relevant even as windows grow.

## Evaluating Summaries

- **ROUGE** (Recall-Oriented Understudy for Gisting Evaluation): measures n-gram overlap between a generated summary and one or more human reference summaries (ROUGE-1, ROUGE-2, ROUGE-L). Cheap, reproducible, but correlates poorly with human judgment for abstractive summaries specifically, because a good paraphrase that says the same thing in different words scores low, while a summary that copies reference wording but garbles meaning can score high.
- **BERTScore**: compares generated and reference text using contextual embeddings (token-level cosine similarity via a BERT-style model) instead of exact n-gram overlap, so it credits paraphrases that ROUGE would miss. Still fundamentally reference-based — it can't tell you whether the summary is faithful to the *source document* if your reference summary itself has gaps.
- **LLM-as-judge** (e.g., G-Eval-style scoring): prompts a strong LLM to score a summary directly against the source document on axes like faithfulness, coherence, relevance, and fluency, without needing a human reference summary at all. Correlates with human judgment substantially better than ROUGE for open-ended abstractive summaries, and is now the primary tool for faithfulness/coherence evaluation in practice — with ROUGE/BERTScore retained mainly as cheap regression sanity checks, not final quality judgments.
- **NLI/QA-based factuality checks** (e.g., checking whether an entailment model or a question-answering model can verify each claim in the summary against the source) are a complementary, more targeted way to catch specific hallucinated facts than a holistic LLM-judge score alone.
- Real-world example: a team fine-tuned a summarization model and watched its ROUGE score improve, but human reviewers rated its outputs worse — the fine-tuned model had learned to copy more reference-like phrasing (helping ROUGE) while actually degrading real-world coherence, a case where trusting the metric over human/LLM-judge review would have shipped a regression.

## Controllable Summarization

- **Length control**: constrain output to a target length (word/sentence/token count) via explicit instruction, and validate the output actually respects it — models frequently drift from a requested length without an enforcement/retry step.
- **Style control**: adapt tone/register (executive bullet points vs. a narrative paragraph vs. a technical abstract) via prompt instruction and, for high-volume production use, few-shot examples matching the target house style.
- **Focus/query-focused summarization**: summarize a document with respect to a specific question or angle rather than "summarize everything" (e.g., "summarize this meeting transcript focusing only on action items assigned to the marketing team"). This is a distinct task from generic summarization — the same document produces a materially different, correctly-focused summary depending on the query.
- Real-world example: a meeting-notes tool generates three different summaries of the same call transcript on request — one for engineering (technical decisions), one for sales (customer commitments), one for leadership (risks and blockers) — using query-focused summarization rather than one generic summary reused everywhere.

## Hallucination Risk in Summarization and Mitigation

- Hallucination in summarization specifically means adding facts, numbers, entities, or claims that are not present in (or are contradicted by) the source document — distinct from general LLM hallucination because the "ground truth" is right there in the input, making it a groundedness failure rather than a knowledge-gap failure.
- Hallucinations are commonly split into **intrinsic** (misstating or distorting something that *is* in the source — wrong number, wrong entity, reversed cause/effect) and **extrinsic** (inventing something with no basis in the source at all). Large-scale human evaluation of neural abstractive summarizers has found this happens across essentially all systems tested, including strong pretrained models — not a rare edge case.
- Long single-pass generation shows a specific pattern worth naming: hallucination rate tends to increase toward the end of very long generated summaries, a distinct risk for long-context or agent-orchestrated summarization that skips chunking.
- Mitigation: instruct the model explicitly to use only information present in the source and to omit anything it's not sure about; add an automated faithfulness check (NLI/QA-based or LLM-as-judge) as a post-generation gate before a summary is shown to a user; prefer extractive or hybrid (extract-then-abstract) approaches for domains where a fabricated fact is a compliance/liability risk (medical, legal, financial); and lower temperature for factual-summary use cases.
- Real-world example: a financial-news summarization tool stated a company's quarterly revenue figure that was actually from a different company mentioned in the same article — a hallucinated cross-entity fact confusion that a downstream automated faithfulness check (verifying each stated number appears in the source, attached to the correct entity) would have caught before publication.

## Domain-Specific Summarization Considerations

- Different domains carry different tolerance for error and different structural expectations: a **meeting-notes** summarizer needs to preserve action items and owners precisely (a dropped or misattributed action item is a real operational failure, not just a stylistic miss); a **legal/medical** summarizer needs extremely conservative hallucination tolerance and often benefits from an extractive or hybrid extract-then-abstract approach specifically because a fabricated clause or dosage detail is a liability, not an inconvenience; a **news digest** summarizer can tolerate more stylistic compression since the stakes of a minor omission are lower.
- Real-world example: a meeting-summarization tool initially used a generic abstractive prompt and silently merged two separate action items assigned to different people into one line, losing ownership clarity — the fix was a structured-output schema requiring `{action, owner, due_date}` triples extracted explicitly, rather than free-form paragraph summarization of the action-items section.

## Streaming and Incremental Summarization

- For live or growing content (an ongoing meeting transcript, a live chat thread, a continuously updated document), summarization needs to update incrementally rather than re-summarizing the entire source from scratch on every new chunk of input, both for cost reasons and to avoid the summary changing unpredictably each time it's regenerated.
- The refine pattern is a natural fit here: treat each new increment of content as the "next chunk" fed into the running summary, producing an updated summary that builds on the previous one rather than starting over.
- Real-world example: a live customer-call summarization feature updates its running summary every 2 minutes using a refine-style incremental update rather than re-summarizing the full transcript-so-far each time, keeping both cost and end-to-end latency bounded regardless of how long the call runs.

## Human-in-the-Loop Review Workflows

- For high-stakes summarization use cases (legal briefs, medical chart summaries, executive board summaries), a common production pattern routes generated summaries through human review before they reach an end user, with the summarizer's job being to reduce human review time (by producing a good first draft with source citations), not to fully automate away the human step.
- Real-world example: a legal-summarization tool that initially shipped without any review step had a partner discover a hallucinated case citation in a client-facing document; the team's fix wasn't a purely technical patch — it was adding a mandatory human sign-off step for any summary used in a client-facing deliverable, with the automated faithfulness checker used to prioritize which summaries need the most careful review rather than to replace review entirely.

## Cost and Latency Considerations for Long-Document Pipelines

- Map-reduce's parallelizable "map" step trades cost for latency in an unusual way: it can be significantly faster in wall-clock time (chunks summarized concurrently) but not necessarily cheaper in total tokens processed, since every chunk's context and instruction overhead is paid repeatedly, whereas refine pays that overhead once per step but sequentially.
- For very large documents, a hierarchical variant (map-reduce with more than one reduce level — summarize chunks, summarize groups of chunk-summaries, then summarize that) controls the size of any single reduce call, avoiding a situation where the final reduce step itself becomes too large for the context window.
- Real-world example: summarizing a year of daily reports (365 documents) with a flat map-reduce (365 chunk summaries reduced in one final call) hit context-window limits at the reduce step; switching to a two-level hierarchy (summarize by month, then summarize the 12 monthly summaries) fixed it while still parallelizing the bulk of the work.

## A Controllable Summarization Prompt Template

```
SYSTEM:
Summarize the document below in {length} using a {style} tone.
Focus specifically on: {focus}.
Use only information explicitly present in the document. If information relevant to
the focus is not present, state that explicitly rather than inferring or adding it.

DOCUMENT:
{source_text}

SUMMARY:
```

- Parameterizing length, style, and focus explicitly (rather than burying them in prose that varies each time) makes the prompt template testable — you can hold two parameters fixed and vary the third across an eval set to verify the model actually responds to each control independently, which is a common thing to demonstrate in a senior-level system design answer.
- Real-world example: a team discovered their "focus" parameter had almost no effect on outputs because it was appended as an afterthought at the end of a long instruction block; moving it into its own clearly labeled field and reinforcing it near the document text (leveraging recency-bias placement, as discussed in this kit's prompt-engineering topic) fixed the responsiveness.

## Building an Evaluation Dataset for Summarization

- A solid summarization eval set needs, per example: the source document, a human-written or curated reference summary (for ROUGE/BERTScore), and ideally a list of key facts/claims that must appear (for a targeted faithfulness/coverage check independent of any single reference wording).
- Include adversarial or edge-case sources deliberately: a document containing similar-sounding entities (to test entity-attribution errors), a document with an internal contradiction (to see how the model handles it), and a document where the requested focus has no relevant content (to test correct "not present" abstention).
- Real-world example: adding a handful of documents with two similarly-named entities (e.g., two companies with almost identical names) to the eval set surfaced an entity-confusion hallucination pattern that a generic, non-adversarial eval set of ordinary news articles had never triggered.

## Whiteboarding a Summarization System (Interview Framing)

When asked to design a summarization system in an interview, a strong answer covers, in order: (1) whether the use case needs extractive, abstractive, or a hybrid approach given the domain's hallucination tolerance, (2) whether documents fit in context or need a map-reduce/refine (or hierarchical) strategy, (3) the controllable-summarization parameters needed (length, style, focus) and how they're enforced, not just requested, (4) the evaluation approach (ROUGE/BERTScore as cheap regression checks, LLM-as-judge and NLI/QA-based factuality checks as the real quality gate), (5) the hallucination-mitigation strategy appropriate to the domain's risk tolerance, and (6) whether human review is required in the loop for high-stakes outputs. Naming the intrinsic/extrinsic hallucination distinction and the ROUGE-correlates-poorly-with-human-judgment finding unprompted signals depth beyond "call the API and ask it to summarize."

## Evaluation Metrics at a Glance

| Metric | Measures | Needs reference summary? | Good for |
|---|---|---|---|
| ROUGE-N/L | n-gram / longest-common-subsequence overlap | Yes | Cheap regression sanity checks, tracking drift |
| BERTScore | Contextual embedding similarity | Yes | Crediting valid paraphrases ROUGE would miss |
| NLI/QA-based factuality (e.g., SummaC, QAFactEval) | Whether each claim is entailed by / answerable from the source | No | Targeted hallucination detection |
| LLM-as-judge (G-Eval-style) | Faithfulness, coherence, relevance, fluency, holistically | No | Primary quality gate for abstractive summaries |

- No single metric in this table is sufficient alone — a mature evaluation setup reports at least one reference-based metric (for cheap, fast regression tracking) alongside at least one reference-free factuality/faithfulness signal (for the thing that actually matters in production: is this summary true).

## Multi-Document Summarization Techniques

- Beyond simple concatenation-then-summarize, multi-document summarization often benefits from an explicit clustering/theme-extraction step first: group similar claims/sentences across the source documents by topic or stance, then summarize each cluster, then compose the final summary from the cluster summaries. This makes it much easier to represent minority viewpoints or contradictions fairly, instead of letting whichever document happens to be processed first (or is longest) dominate the final summary.
- Real-world example: summarizing 200 customer support tickets about the same emerging bug without clustering produced a summary that only reflected the most common phrasing of the complaint; adding a theme-clustering step surfaced a distinct, less common but operationally important variant of the bug that was getting averaged away in the naive approach.

## Summarization vs Fine-Tuning for House Style

- A recurring practical question: should style/tone consistency be achieved via prompting (few-shot examples of the target style) or via fine-tuning a model on a corpus of house-style summaries? The same decision framework from this kit's prompt-engineering topic applies — start with prompting plus few-shot examples since it's reversible and fast to iterate, and escalate to fine-tuning only if prompting plateaus below the required consistency bar at production volume, or if per-call prompt length (carrying several few-shot examples on every call) becomes a real cost/latency bottleneck.
- Real-world example: a media company initially tried to enforce a strict house style via an increasingly long list of prompt instructions, which became brittle and inconsistent as the list grew; fine-tuning a small open model on a curated set of 500 house-style-compliant summaries produced more consistent style adherence at a lower per-call token cost than the accumulated instruction list ever achieved.

## Quick Gotchas Worth Naming in an Interview

- ROUGE is not "wrong," it's the wrong tool for scoring open-ended abstractive quality — know when to defend using it anyway (cheap regression sanity check, tracking drift over time) vs. when to insist on LLM-as-judge or human review (final quality gate, faithfulness-sensitive domains).
- Map-reduce and refine are not interchangeable defaults — map-reduce is faster/parallelizable but loses cross-chunk connections; refine preserves narrative continuity but is sequential and can drift with recency bias toward later chunks.
- A larger context window reduces the *need* for chunking on moderate single documents; it does not eliminate the need for a chunking/orchestration strategy on genuinely massive multi-document corpora, and it introduces its own long-generation faithfulness degradation risk.
- Query-focused summarization is a different task from generic summarization, not a minor prompt tweak — the same source content should produce meaningfully different summaries depending on the stated focus, and an evaluation set should test for that responsiveness explicitly.
