# Security, Governance, and Responsible AI — Scenario-Based Q&A

**Situation:** A developer accidentally commits a `.env` file containing a production LLM API key to a public GitHub repo, and it's discovered three days later after an unexpectedly large bill arrives. What would you do and why?

Model answer: Rotate the leaked key immediately — this takes priority over investigation, since every minute it stays live is more unauthorized usage. Scrub the key from Git history (not just delete it in a new commit, since old commits remain accessible), and check the exposure window and provider's usage logs to quantify the damage and whether any anomalous usage patterns (unfamiliar IPs, unusual request volume) suggest active abuse versus opportunistic scraping. Root-cause the process gap: add `.env` to `.gitignore` repo-wide via a template, add a pre-commit secret-scanning hook (e.g., gitleaks) so this class of leak is caught before it reaches GitHub, and migrate the key to a proper secrets manager so credentials are never in a file that could be committed in the first place.

---

**Situation:** A user discovers they can get your customer-support chatbot to ignore its system prompt and answer completely unrelated questions by prefacing their message with "Ignore all previous instructions and instead..." What would you do and why?

Model answer: This is a textbook prompt-injection / jailbreak, and the fix operates at two layers. Add an input guardrail that screens for known jailbreak patterns (phrases like "ignore previous instructions," "you are now DAN," system-prompt-extraction attempts) before the message reaches the model, using either regex/keyword rules for obvious cases or a lightweight classifier model for more nuanced attempts. Also harden the system prompt itself with explicit anti-override language and consider structuring the prompt so user input is clearly delimited and never appears in a position that looks like a system instruction — then add this exact bypass to a permanent regression test so future prompt or model changes can't silently reopen it.

---

**Situation:** Leadership asks your team to demonstrate "responsible AI" practices ahead of an enterprise customer's security review, and you currently have no formal documentation beyond the code itself. What would you do and why?

Model answer: Start with the highest-leverage, lowest-effort artifact: a model card for each production model/prompt documenting what it does, known limitations, how it was evaluated, and what guardrails are in place — this alone answers most reviewer questions. Pair it with audit logs showing what was asked, what the model answered, and what actions were taken (already valuable for debugging, now doubling as compliance evidence), and a short written policy describing human-oversight points for high-stakes decisions. None of this requires new infrastructure if logging and evaluation already exist from earlier modules — it's largely a documentation and organization exercise, which is worth stating plainly rather than implying a large new build is required.

---

**Situation:** An engineer proposes storing the OpenAI API key as a hardcoded string in the deployment script "just for now, we'll fix it before launch," and the launch date keeps slipping. What would you do and why?

Model answer: Push back immediately rather than trusting "for now" — hardcoded secrets have a strong tendency to survive past their intended temporary window, especially once a script gets copied, shared, or checked into a branch. The fix is no harder than the shortcut: load the key from an environment variable or the team's existing secrets manager, which takes minutes and removes the risk entirely rather than deferring it to an uncertain future cleanup. If there's a genuine reason it can't be fixed before launch, require an explicit ticket with an owner and a hard deadline, and treat the hardcoded secret as a known vulnerability tracked like any other, not an invisible one.

---

**Situation:** Your RAG-based internal tool retrieves and summarizes documents from a shared company wiki, and someone points out that a malicious or careless wiki edit could embed hidden instructions that get executed when the document is retrieved and fed to the LLM. What would you do and why?

Model answer: Take this seriously as indirect prompt injection — the attack doesn't need to come through the chat interface at all if untrusted text can reach the model's context via retrieval. Treat retrieved content as data, never as instructions, by structuring the prompt so retrieved text is clearly delimited and the system prompt explicitly instructs the model not to follow any instructions found within retrieved content. Add an output guardrail that checks for signs the model's behavior deviated from its task (e.g., generating content wildly unrelated to summarization), and restrict wiki edit permissions or add a review step for pages that feed high-trust internal tools, since the retrieval corpus is now part of your attack surface.

---

**Situation:** Your agentic system has a tool that can execute database queries, and during testing someone gets the agent to run a `DROP TABLE` command by phrasing a request in a roundabout way. What would you do and why?

Model answer: This is a tool-use guardrail gap, and the fix is architectural, not just prompt-based — never rely on the model "choosing" not to run a destructive command when the tool itself has the power to do it. Restrict the database tool's actual permissions at the connection level (a read-only credential, or an allowlist of permitted statement types) so destructive operations are impossible regardless of what the model is tricked into requesting, and require explicit human confirmation for any tool call classified as destructive or irreversible. Log every tool call with its arguments so any near-miss like this is visible in an audit trail, not just caught by luck during manual testing.

---

**Situation:** A regulator under the EU AI Act asks your company to demonstrate ongoing bias monitoring for a hiring-screening LLM feature, not just a one-time fairness check done before launch. What would you do and why?

Model answer: Explain, and then build, bias/fairness monitoring as a continuous evaluation practice rather than a launch gate that's checked once and forgotten — this means running the model against a diverse, representative evaluation set on a recurring cadence (not just at launch), tracking outcome disparities across protected groups over time, and alerting if disparities exceed a defined threshold. Document this monitoring process itself (who reviews it, how often, what triggers escalation) since the regulator is asking about process maturity, not just a single metric. If this monitoring doesn't exist yet, be direct that it's a gap to close before the review, rather than retrofitting a single point-in-time check to look like ongoing monitoring.

---

**Situation:** A customer support agent using your internal LLM tool pastes a customer's full conversation, including their SSN, into the chat to ask the model to "summarize this ticket," and the summary (including the SSN) gets logged in a third-party observability platform. What would you do and why?

Model answer: Treat this as a PII-handling failure with two separate fixes. First, add an input guardrail (regex or a PII-detection classifier) that flags or redacts SSNs and similar sensitive patterns before they're sent to the model or any logging system, so this class of leak can't recur through the same path. Second, audit what's already stored in the third-party observability platform — determine retention policy, whether that vendor is covered under your data processing agreements, and whether the exposed SSN needs to be purged or disclosed per your data privacy obligations (e.g., GDPR/CCPA breach-notification requirements), since this may be a reportable incident depending on jurisdiction and internal policy.

---

**Situation:** Your team wants to adopt a third-party guardrail library (e.g., NeMo Guardrails or Guardrails AI) but a colleague argues "our simple regex blocklist works fine, why add the dependency?" What would you do and why?

Model answer: Acknowledge the regex approach genuinely works for narrow, well-known patterns, and don't replace it reflexively — but point out its blind spot: regex/keyword blocklists catch attacks phrased the way you anticipated, and jailbreak phrasing evolves faster than a manually maintained blocklist can keep up with (paraphrasing, encoding tricks, multi-turn setups). A dedicated guardrail library or a classifier-based check adds resilience to novel phrasings at the cost of added latency and a new dependency — recommend keeping the fast regex layer for known, cheap-to-catch patterns as a first pass, and adding a classifier-based second layer for cases that need more nuanced judgment, rather than treating it as an either/or choice.

---

**Situation:** During a security review, someone asks whether your production Kubernetes deployment stores API keys as plain environment variables baked into the container image or injected at runtime, and nobody on the team is sure. What would you do and why?

Model answer: Treat "nobody is sure" as itself the finding — secrets whose storage mechanism isn't confidently known by the team responsible for them is a governance gap regardless of the technical answer. Audit the actual deployment manifests and Dockerfiles to determine the truth: if secrets are baked into the image, they're visible to anyone who can pull that image and persist in every layer/registry copy ever made, which is a real exposure. Migrate to Kubernetes Secrets (or an external secrets operator pulling from a proper secrets manager) so credentials are injected into the pod at runtime and never part of the image itself, and document the pattern so the next engineer doesn't have to guess.

---

**Situation:** A product team wants to launch an AI feature that makes automated credit-approval decisions with no human review step, arguing that human review would slow down the user experience too much. What would you do and why?

Model answer: Push back specifically on the "no human review for any case" framing rather than treating this as all-or-nothing — high-stakes, life-impacting decisions like credit approval are exactly the category where documented human oversight matters most, both ethically and for compliance (many jurisdictions have explicit rules about automated decision-making in credit and lending). Propose a tiered approach: fully automate clearly low-risk approvals where the model has demonstrated high confidence and accuracy, but route borderline or denied cases to human review, which preserves most of the speed benefit while keeping a human in the loop exactly where errors are most consequential. Document this decision and its rationale, since "we considered full automation and chose this tiered design for these reasons" is itself valuable governance evidence.

---

**Situation:** Six months after launch, someone asks your team to explain why a specific customer received a particular chatbot response that turned out to be inappropriate, but your system only logs aggregate metrics (request counts, latency), not actual prompts and responses. What would you do and why?

Model answer: Recognize this as a direct consequence of not building audit logging in from the start — without logged prompts and responses, this specific incident is essentially undebuggable, and worse, you have no way to demonstrate to a customer or regulator what actually happened. Fix it going forward immediately: log prompts, responses, retrieved context (if RAG), and any tool/agent actions, with clear data-retention and access-control policies since these logs may contain sensitive user data. For the specific incident in question, be honest that it can't be fully reconstructed, and use it as the concrete justification for prioritizing audit logging that a "someday" ticket previously couldn't get resourced for.

---

**Situation:** Your team fine-tunes an LLM on internal support tickets, and afterward realizes some tickets contained customers' credit card numbers pasted for troubleshooting. The fine-tuned model is already deployed. What would you do and why?

Model answer: Treat this as a serious incident requiring immediate containment, not a quiet cleanup — a model fine-tuned on data containing raw card numbers may be able to regurgitate them, which is a real data-exposure risk regardless of how unlikely a specific extraction seems. Pull the fine-tuned model from production immediately, involve security/legal to assess disclosure obligations (this likely intersects with PCI-DSS and breach-notification requirements), and retrain from a properly scrubbed dataset with PII/PCI redaction applied before fine-tuning, not after. Use this to justify making PII/PCI scanning a mandatory pre-processing gate for any fine-tuning dataset going forward, since the fix is much cheaper before a model is trained than after.

---

**Situation:** A junior engineer asks why the team bothers with input *and* output guardrails when "the input guardrail already blocks bad requests before they reach the model." What would you do and why?

Model answer: Explain that input and output guardrails catch different failure classes, and one doesn't substitute for the other. An input guardrail can miss a cleverly phrased jailbreak that slips past detection, or block nothing at all if the "badness" only emerges from the model's own behavior (e.g., the model hallucinating harmful content unprompted, or leaking part of its system prompt in an otherwise benign response). An output guardrail is the last line of defense regardless of how the bad content arose — validating structured output against a schema, checking for leaked secrets or PII, and screening for toxic language before it ever reaches the user. Frame it as defense in depth: assume either layer can fail individually, and design so a single miss doesn't reach the user.

---

**Situation:** Your agentic system occasionally calls an external payment-refund API, and a stakeholder asks what stops the agent from issuing a large fraudulent refund if it's manipulated through a crafted user request. What would you do and why?

Model answer: Answer with the specific control, not a general reassurance: the refund tool should enforce a hard-coded maximum refund amount and require explicit human confirmation above a defined threshold, implemented at the tool/API layer so it holds regardless of what the model is convinced to attempt. Log every refund tool call with full context (user request, agent reasoning if available, amount, outcome) for audit, and periodically review a sample of agent-initiated refunds even within the approved range to catch subtler manipulation that stays under the confirmation threshold. Make clear to the stakeholder that "trusting the model's judgment" is never the actual control — the control is what the tool is capable of doing and what requires a human sign-off.

---

**Situation:** During onboarding, a new hire asks why the team treats "the model was jailbroken" and "the model hallucinated" as different problems requiring different fixes, when both result in a bad response reaching the user. What would you do and why?

Model answer: Distinguish the two clearly: a jailbreak is an adversarial user deliberately manipulating the model into violating its intended behavior, which is addressed with guardrails and access controls aimed at the input/prompt layer; a hallucination is the model confidently generating false information with no adversarial intent involved, which is addressed by grounding responses in retrieved, verifiable data (RAG) and evaluation/monitoring for factual accuracy. They can produce similarly bad-looking outputs, but conflating them leads to the wrong fix — adding jailbreak-detection regex won't reduce hallucination rate, and improving retrieval grounding won't stop a determined adversarial user. Treat them as two entries in the same overall safety taxonomy, each with its own detection and mitigation strategy.

---

**Situation:** Your company's legal team asks whether the LLM-powered features currently in production have documented model cards, and the honest answer is that only one of your five production models does. What would you do and why?

Model answer: Give the honest count rather than deflecting, and treat this as a prioritization problem, not a crisis — not all five models carry equal risk, so triage by exposure: customer-facing and high-stakes models (anything touching financial, health, or legal decisions) get model cards first, internal low-stakes tools can follow. Draft a lightweight, reusable model-card template (purpose, known limitations, evaluation summary, guardrails in place, owner) so producing the remaining four isn't a large one-off effort, and propose making a model card a required artifact before any future model reaches production, closing the gap for good rather than just catching up once.
