# Prompt Engineering and In-Context Learning — References

All links below were fetched and content-verified this session (2026-09-18) unless explicitly marked otherwise.

## Papers

- Wei et al., "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models" (NeurIPS 2022). https://arxiv.org/abs/2201.11903
- Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" (ICLR 2023). https://arxiv.org/abs/2210.03629
- Zhao et al., "Calibrate Before Use: Improving Few-Shot Performance of Language Models" (2021) — on label-bias sensitivity in few-shot prompting. https://arxiv.org/abs/2102.09690
- Lu et al., "Fantastically Ordered Prompts and Where to Find Them: Overcoming Few-Shot Prompt Order Sensitivity" (2021/2022) — the primary citation for example-order sensitivity in in-context learning. https://arxiv.org/abs/2104.08786
- "EchoLeak: The First Real-World Zero-Click Prompt Injection Exploit in a Production LLM System" (AAAI Fall Symposium Series 2025) — academic writeup of CVE-2025-32711, a disclosed, CVE-tracked zero-click indirect prompt injection vulnerability in Microsoft 365 Copilot; a single crafted email's hidden instructions were processed as commands, evading the built-in prompt-injection classifier and exfiltrating data with no user interaction. The strongest available real-world citable incident for this topic. https://arxiv.org/abs/2509.10540

## Frameworks and Docs

- DSPy — "the framework for programming, not prompting, language models"; treats prompt wording and few-shot example selection as parameters optimized against a metric. Actively maintained, ~38k GitHub stars at check time. https://github.com/stanfordnlp/dspy
- TextGrad — a "textual gradient," critic-model-feedback-based alternative to DSPy's compiled-module optimization approach, worth knowing by name as a second automatic-prompt-optimization framework. (not URL-verified this session — arXiv id 2406.07496, surfaced via search corroboration only; re-check before citing)
- OpenAI Structured Outputs docs (distinguishes JSON mode — valid JSON, no schema guarantee — from Structured Outputs — schema-enforced, available since GPT-4o 2024-08-06). https://developers.openai.com/api/docs/guides/structured-outputs
- Anthropic tool use / function calling docs (client tools, server tools, strict tool use for schema conformance). https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
- LangSmith Prompt Hub — version-controlled prompt storage with environment labels and rollback, the reference implementation pattern for "treat prompts as versioned artifacts." https://docs.langchain.com/langsmith/manage-prompts

## Security

- OWASP Top 10 for LLM Applications — project home. Prompt Injection (LLM01) has held the #1 ranking across the 2023 and 2025 revisions, per multiple 2025 secondary sources — a useful talking point since it signals the risk is structural, not something patched away release over release. https://owasp.org/www-project-top-10-for-large-language-model-applications/ (live ranked list at https://genai.owasp.org/llm-top-10/ — ranking corroborated via search, not independently re-fetched from that specific page this session)

## Notes on What to Prioritize

Interview signal consistently points to: being able to name and explain CoT and ReAct precisely (not just "chain of thought means reasoning"), understanding in-context learning's brittleness (order/count/selection sensitivity, with real papers to cite) as a mechanism that can regress silently after seemingly harmless changes, knowing the exact difference between JSON mode and schema-constrained structured output, and treating prompt injection as a structural, unpatchable-by-wording risk requiring access-control mitigation, not a prompt-level fix. A live and citable current industry narrative worth mentioning at a senior level: the standalone "prompt engineer" job title is fading as base models get better at intent-reading, while the underlying discipline (templates, versioning, evaluation, injection defense, automatic optimization) is being absorbed into the broader AI/agent engineer role — sometimes now called "context engineering." This framing is directly relevant to positioning an 8-10 year senior candidate's skill set correctly in an interview.
