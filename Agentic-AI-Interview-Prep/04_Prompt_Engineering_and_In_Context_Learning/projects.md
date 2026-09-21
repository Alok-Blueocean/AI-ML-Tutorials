# Prompt Engineering and In-Context Learning — Projects

## Small: Versioned Prompt Template Library with Regression Tests

Build a small Python library that stores prompts as versioned, parameterized templates (not raw f-strings), with a changelog and rollback support, plus a regression test suite that checks format compliance, basic injection resistance, and expected refusal behavior on out-of-scope input for each template version. Wire it into a CI check that fails if a template change regresses any test. This proves you treat prompts as production artifacts with the same discipline as code, which is a common senior-level differentiator.

## Medium: Structured-Extraction Pipeline with Injection Red-Teaming

Build a document-extraction pipeline (pull structured fields from invoices, resumes, or contracts — use a public dataset or synthetic documents) using schema-constrained structured output, then deliberately red-team it with indirect prompt injection attempts embedded in the source documents (hidden instructions in invoice text, for example) to see what breaks. Document every successful injection and the specific mitigation (least-privilege tool access, input sanitization, trust-boundary framing in the prompt) you applied for each. This proves you understand both the productive side of structured-output prompting and its real security surface.

## Medium: In-Context Learning Sensitivity Study

Pick a classification or extraction task and build an experiment harness that systematically varies few-shot example count, order, and label balance, measuring accuracy/consistency across each variation on a fixed test set. Produce a short report (a real deliverable, e.g., a one-pager) with your findings and a recommended "stable" few-shot configuration for production use. This proves you understand in-context learning as an empirically fragile mechanism worth testing, not something to configure once and forget.

## Large: DSPy-Optimized Multi-Step Reasoning Pipeline vs. Hand-Tuned Baseline

Build a multi-step task (e.g., a ReAct-style research assistant that reasons, calls a search/calculator tool, and synthesizes an answer) two ways: a hand-tuned prompt pipeline you iterate on manually, and a DSPy-compiled version optimized against a defined metric on a training set. Evaluate both on a held-out test set for accuracy, consistency across reruns, and the effort/iteration cost to build each. This proves you understand automatic prompt optimization as a real engineering tool with measurable tradeoffs against manual prompt engineering, not just a framework name to drop.
