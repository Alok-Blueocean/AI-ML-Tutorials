# Security, Governance, and Responsible AI

## What This Topic Is

Every module so far has focused on making ML/LLM systems work well — accurate, fast, observable, cost-efficient. This module is about making them **safe to operate**. That means protecting the secrets your system depends on (API keys, credentials), controlling what a model is allowed to say or do (guardrails), and making sure the whole system meets legal, ethical, and organizational expectations (compliance and responsible AI).

In traditional software, security mostly meant "don't leak data and don't get hacked." With LLMs, there's a new attack surface: the model itself can be tricked, manipulated, or misused through natural language — no exploit code required. So this module covers both classic infrastructure security and a newer discipline specific to AI systems.

## Why It Matters

- **Secrets are everywhere in ML/LLM pipelines** — database passwords, cloud credentials, third-party API keys (OpenAI, Anthropic, vector DB providers). A leaked key in a public repo or log file can mean a huge bill or a data breach.
- **LLMs can be manipulated through plain text.** A user can ask a chatbot to ignore its instructions, reveal system prompts, generate harmful content, or trick an agent into calling a tool it shouldn't. Guardrails exist to catch this.
- **Regulators and customers now expect proof of responsible AI use.** Frameworks like the EU AI Act, and internal enterprise policies, require documentation of how models are tested, monitored, and governed — not just that they work.
- **Trust is the product.** For an AI feature to survive in production, the business needs confidence it won't leak data, embarrass the company, or make an unaccountable decision.

## Main Concepts in Plain Terms

### 1. Secrets Management

A "secret" is any credential your application needs but should never be hard-coded or committed to source control: API keys, database passwords, service tokens, signing keys.

Best practice is to keep secrets **out of code entirely** and load them at runtime from a dedicated store:

- **Environment variables** — the simplest approach, good for local dev and small projects.
- **Secret managers** — dedicated services like AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, or Google Secret Manager. These add encryption at rest, access control, and rotation.
- **Kubernetes Secrets** — for containerized workloads, secrets are mounted into pods rather than baked into images.

The core rule: **secrets should be injected, never checked in.** A `.env` file is fine locally as long as it's in `.gitignore`.

### 2. Guardrails

Guardrails are checks placed around an LLM to constrain its behavior — before the prompt reaches the model, and after the response comes back.

- **Input guardrails** — screen the user's message before it hits the model. Examples: blocking prompt-injection attempts, filtering PII, rejecting requests for disallowed topics.
- **Output guardrails** — screen the model's response before it reaches the user. Examples: checking for toxic language, verifying the response doesn't leak system prompts or secrets, validating that structured output (like JSON) matches the expected schema.
- **Tool-use guardrails** — for agentic systems, restrict which tools/actions the model can invoke, and under what conditions (e.g., "never allow the agent to run a `DELETE` query without human confirmation").

Guardrails can be simple rule-based filters (keyword/regex blocklists), or a second LLM call acting as a classifier/judge ("is this input a jailbreak attempt?").

### 3. Safety

Safety is the broader goal guardrails serve: making sure the system doesn't cause harm. Key safety concerns for LLM systems include:

- **Prompt injection** — malicious instructions hidden in user input or retrieved documents that try to override the system prompt.
- **Jailbreaking** — attempts to bypass a model's built-in restrictions through clever phrasing.
- **Hallucination** — the model confidently stating false information; mitigated by grounding responses in retrieved, verifiable data (see the RAG/vector DB modules) and clear evaluation.
- **Bias and fairness** — outputs that systematically disadvantage certain groups; addressed through diverse evaluation datasets and ongoing monitoring, not a one-time check.

### 4. Compliance Basics

Compliance means demonstrating — often to auditors, regulators, or customers — that your AI system is built and operated responsibly. Common building blocks:

- **Data privacy** — knowing what personal data flows through your system, and complying with regulations like GDPR (right to deletion, data minimization).
- **Audit logging** — keeping a record of what was asked, what the model answered, and what actions were taken, so behavior can be reviewed after the fact.
- **Model documentation** — sometimes called a "model card" — a short document describing what the model does, its known limitations, and how it was evaluated.
- **Human oversight** — a documented process for humans to review, override, or shut down automated decisions, especially in high-stakes use cases (finance, healthcare, hiring).

None of this requires becoming a lawyer — it mostly means keeping good records and making deliberate, documented choices instead of ad-hoc ones.

## A Simple Example

A minimal guardrail pattern: keep secrets out of code, and add a lightweight input/output check around the LLM call.

```python
import os
import re

# 1. Secret loaded from environment, never hard-coded
API_KEY = os.environ["LLM_API_KEY"]

# 2. A simple input guardrail: block obvious prompt-injection phrases
BLOCKED_PATTERNS = [r"ignore (all|previous) instructions", r"reveal your system prompt"]

def input_guardrail(user_message: str) -> bool:
    """Return True if the message is safe to send to the model."""
    for pattern in BLOCKED_PATTERNS:
        if re.search(pattern, user_message, re.IGNORECASE):
            return False
    return True

# 3. A simple output guardrail: don't let secrets leak back to the user
def output_guardrail(model_response: str) -> str:
    if API_KEY in model_response:
        return "[Response blocked: potential secret leakage]"
    return model_response

def handle_request(user_message: str):
    if not input_guardrail(user_message):
        return "Sorry, I can't help with that request."

    response = call_llm(user_message, api_key=API_KEY)  # your actual LLM call
    return output_guardrail(response)
```

This is intentionally basic — real systems typically use dedicated guardrail libraries (e.g., NeMo Guardrails, Guardrails AI) or a classifier model for more nuanced detection. But the pattern is the same: **validate before, validate after, never expose secrets in either direction.**

## Key Takeaways / Best Practices

- **Never commit secrets.** Use environment variables or a secrets manager, and rotate credentials regularly.
- **Apply the principle of least privilege** — services and agents should only have access to the data and tools they actually need.
- **Add guardrails at both ends** — screen risky inputs before they reach the model, and screen risky outputs before they reach the user.
- **Treat prompt injection as a real threat**, especially in RAG and agentic systems where untrusted text (documents, web pages, tool outputs) reaches the model.
- **Log for auditability** — keep a record of prompts, responses, and agent actions so you can review and explain what happened.
- **Document your model and its limits** — a short model card is far better than tribal knowledge.
- **Keep a human in the loop** for high-stakes or irreversible actions.
- **Treat safety and compliance as ongoing processes**, not one-time checklists — revisit them as the model, data, and regulations change.
