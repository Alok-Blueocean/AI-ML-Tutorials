# 60-Day Roadmap: Production Agentic AI / LLMOps on AWS

*Target stack: FastAPI, Docker, ECS Fargate, Lambda, Bedrock, Terraform, GitHub Actions, CloudWatch, Langfuse, Prometheus/Grafana, OpenSearch, LangGraph, SageMaker, Evidently AI, EventBridge, Security (IAM/Secrets Manager/guardrails).*

---

## 0. The honest answer on "one book that covers all this"

There isn't one, and be skeptical of anyone who claims otherwise. This exact combination — classic AWS infrastructure (ECS Fargate, Lambda, Terraform, CloudWatch, EventBridge, IAM) fused with fast-moving agentic/LLMOps tooling (Bedrock, LangGraph, Langfuse, Evidently AI) — is closer to a specific team's stack choice than an industry-standard curriculum. No publisher has caught up to bundling it into one coherent text, and several of these tools (Langfuse, LangGraph, Evidently AI's LLM features) are moving too fast for a book to stay current anyway.

The real preparation strategy isn't "find the one book" — it's **build one integrated project that touches all 15 pieces**, so you have real answers instead of tutorial-recall. Section 2 is that project plan. Section 1 is the best resource per cluster, to *support* the build, not replace it.

---

## 1. Best real resource per cluster

| Cluster | Best resource | Why |
|---|---|---|
| AWS core infra (ECS Fargate, Lambda, EventBridge, IAM, Secrets Manager, CloudWatch) | **AWS Skill Builder** (free, official) + AWS's own service docs | These services change often — official docs and Skill Builder labs stay current in a way books can't |
| Terraform | *Terraform: Up & Running* (Yevgeniy Brikman, O'Reilly) | The standard reference for real-world Terraform module design, still holds up |
| Docker (multi-stage builds), FastAPI | Official Docker docs + official FastAPI docs | Both are unusually well-written and example-rich; no book beats the source here |
| Bedrock, SageMaker | AWS Samples' official **Bedrock/SageMaker workshop repos** on GitHub + official docs | Hands-on, maintained by AWS, and current — books can't track Bedrock's release cadence |
| Prometheus / Grafana | *Prometheus: Up & Running* (Brian Brazil, O'Reilly) | The standard reference for metrics design and alerting |
| Langfuse, LangGraph, Evidently AI, OpenSearch-as-vector-store | Official documentation only | None of these have a mature book yet — the docs *are* the best resource |
| Conceptual grounding: drift, monitoring, retraining decisions, production ML thinking | *Designing Machine Learning Systems* (Chip Huyen, O'Reilly) | Not AWS-specific, but the closest thing to "the systematic book" for the *reasoning* behind everything in Section 2 |

**Update:** the table above has since been spot-verified — see `Verified_Study_Resources_and_Corrections.md` in this same folder for confirmed links and two corrections worth knowing: LangGraph's docs moved to `docs.langchain.com/oss/python/langgraph/overview`, and LangGraph does not include its own evaluation tooling (that lives in the separate LangSmith product — pair LangGraph with LangSmith or with DeepEval/Ragas, not with LangGraph alone). The "no single unified book" conclusion in Section 0 is still unverified either way — an open web search never became available to check it — so treat that specific claim as still open.

---

## 2. The real preparation strategy: one running project, not 15 tutorials

Interviewers can usually tell the difference between "read about it" and "built it" within two follow-up questions. Tutorial-followers can explain what a tool does; people with real experience can explain **what broke, why, and what they changed**. Reading five more resources doesn't create that — building one thing that fails in realistic ways does.

The plan below builds **one Agentic RAG system end-to-end** over 8 weeks, adding one capability cluster per phase, so that by day 60 you have a single deployed, instrumented, self-monitoring system you can walk an interviewer through — plus a build log full of real incidents.

### Day-by-day schedule (this is *your own build plan*, not an external course — "Must do" days are where the real work happens; lighter days are intentionally lighter so you don't burn out before week 4)

| Day | Focus | Priority |
|---|---|---|
| Week 1, Day 1 | AWS sandbox setup: IAM user, Bedrock model access request, Docker installed, repo scaffolded | Skim (30–60 min) — this is admin, not learning |
| Week 1, Day 2 | FastAPI service skeleton wired directly to a Bedrock call | **Must do fully** |
| Week 1, Day 3 | OpenSearch index stood up, load a small real (messy) document set | **Must do fully** |
| Week 1, Day 4 | Wire retrieval + Bedrock call into one working RAG endpoint, test end to end | **Must do fully** |
| Week 1, Day 5 | Dockerize with a real multi-stage build; deploy manually to ECS Fargate | **Must do fully** |
| Week 2, Day 1 | Load a bigger/messier real document set; fix chunking problems this surfaces | **Must do** |
| Week 2, Day 2 | Hit the public endpoint with real varied queries; fix retrieval bugs you find | **Must do** |
| Week 2, Day 3 | Write a small baseline eval set (10–20 questions) and run it manually | **Must do** |
| Week 2, Day 4 | Add real error handling/timeouts/retries — don't let one bad call 500 the service | **Must do** |
| Week 2, Day 5 | Checkpoint: confirm the Weeks 1–2 "definition of done" (curl → real RAG answer) | **Must do** |
| Week 3, Day 1 | Read LangGraph docs / a getting-started tutorial if this framework is new to you | Optional (skip if already familiar) |
| Week 3, Day 2 | Design the agent graph on paper: planner → retrieve → verify → answer, one branch point | Optional / skim |
| Week 3, Day 3 | Implement the multi-step LangGraph workflow for real | **Must do** |
| Week 3, Day 4 | Add the conditional branch (e.g. escalate/re-retrieve on low confidence) | **Must do** |
| Week 3, Day 5 | Test the branching with inputs designed to trigger each path; fix what breaks | **Must do** |
| Week 4, Day 1 | Set up an S3 bucket + EventBridge rule for document-upload events | **Must do** |
| Week 4, Day 2 | Write the Lambda that re-indexes into OpenSearch when triggered | **Must do** |
| Week 4, Day 3 | Test the event pipeline end-to-end (upload → re-index → retrievable) | Watch (lighter — mostly verification, not new building) |
| Week 4, Day 4 | Wire Langfuse into every LLM/agent call (prompts, tokens, cost, latency) | **Must do** |
| Week 4, Day 5 | Checkpoint: pull up a real trace and narrate a case where the agent branched differently | **Must do** |
| Week 5, Day 1 | CloudWatch wired in for application logs | **Must do** |
| Week 5, Day 2 | Prometheus + Grafana dashboards for infra metrics (CPU, memory, latency, error rate) | **Must do** |
| Week 5, Day 3 | Read AWS IAM least-privilege guidance before touching your roles | Optional / skim |
| Week 5, Day 4 | Rewrite IAM roles to least-privilege; move every credential into Secrets Manager | **Must do** |
| Week 5, Day 5 | Checkpoint: deliberately trigger a bad answer, diagnose it end-to-end using only your own trace | **Must do** |
| Week 6, Day 1 | Fix whatever the Week 5 checkpoint exposed (prompt/retrieval/logic) | **Must do** |
| Week 6, Day 2 | Read Evidently AI's docs on LLM-specific drift metrics | Optional / skim |
| Week 6, Day 3 | Stand up Evidently AI monitors on retrieval quality / query distribution | **Must do** |
| Week 6, Day 4 | Set real alert thresholds (start from Section 11.2 of `Continuous_Monitoring_Comprehensive_Notes.md`) | **Must do** |
| Week 6, Day 5 | Checkpoint: simulate a drift scenario and confirm the dashboard/alert actually fires | **Must do** |
| Week 7, Day 1 | Refresher on Terraform basics if rusty; otherwise skip | Optional |
| Week 7, Day 2 | Write Terraform modules for your existing hand-built infra (ECS, Lambda, OpenSearch, IAM) | **Must do** |
| Week 7, Day 3 | Migrate console-created resources to be fully Terraform-managed | **Must do** |
| Week 7, Day 4 | GitHub Actions pipeline: tests run and deploy happens automatically on merge | **Must do** |
| Week 7, Day 5 | Checkpoint: tear the whole stack down and rebuild it from `terraform apply` alone | **Must do** |
| Week 8, Day 1 | Stand up a SageMaker inference endpoint; measure cost/latency against Bedrock for your workload | **Must do** |
| Week 8, Day 2 | Read up on SageMaker training jobs if you're pursuing the classifier/retrain angle | Optional |
| Week 8, Day 3 | Train a small real classifier on SageMaker (e.g. severity/category) — your genuine retraining story | **Must do** |
| Week 8, Day 4 | Deliberately break something (inject drift, throttle a dependency, corrupt a document); use the root-cause playbook to fix it properly | **Must do** |
| Week 8, Day 5 | Finalize your build log, write up the incident from Day 4 as a real story, do a full end-to-end demo walkthrough | **Must do** |

### Weeks 1–2 — Build the core service

**Goal: something actually running, reachable by URL — not just code on your laptop.**

- [ ] FastAPI app wrapping a Bedrock model call (or a local model if you want to avoid cost) behind a basic RAG pipeline
- [ ] OpenSearch as the vector store (index a small real document set — not toy data, something with messy real structure)
- [ ] Dockerize with an actual multi-stage build (builder stage + slim runtime stage — measure the image size difference, don't just copy a Dockerfile)
- [ ] Deploy to ECS Fargate (AWS free tier / smallest task size is fine)
- [ ] **Definition of done:** you can curl a public endpoint and get a real RAG answer back

### Weeks 3–4 — Make it a real agent, not a chatbot

**Goal: a system with state and branching, not a single prompt-in/answer-out loop.**

- [ ] Add LangGraph for a multi-step, tool-using workflow (e.g. planner → retrieve → verify → answer, with at least one conditional branch)
- [ ] Add one event-driven piece with Lambda — e.g. an S3 upload triggers an EventBridge rule that invokes a Lambda to re-index a document
- [ ] **Definition of done:** you can describe a specific case where the agent took a different path depending on input, and show the trace of that decision

### Weeks 5–6 — Instrument it like production, not a demo

**Goal: you can answer "how would you debug a bad answer at 2am" with a real trace, not a hand-wave.**

- [ ] Langfuse wired in for full LLM tracing (prompts, retrieved context, tokens, cost, latency per call)
- [ ] CloudWatch for application logs + Prometheus/Grafana for infra metrics (CPU, memory, request latency, error rate)
- [ ] IAM roles scoped to least-privilege (not `*` permissions) and Secrets Manager for every credential instead of `.env` files
- [ ] **Definition of done:** deliberately trigger a bad answer (e.g. ask something outside your document set) and use your own trace to diagnose why, end to end

### Weeks 7–8 — Close the loop

**Goal: a war story you can tell in an interview — something that actually broke and that you actually fixed.**

- [ ] Evidently AI watching for drift on your retrieval quality or query distribution
- [ ] GitHub Actions pipeline: tests run and deploy happens on merge — no manual `docker push`
- [ ] All infrastructure defined in Terraform (not clicked together in the console) — this alone is a strong signal of real experience
- [ ] SageMaker touchpoint: either a small fine-tune/retrain job, or at minimum an inference endpoint you compare against Bedrock for cost/latency
- [ ] Deliberately break something — feed it a distribution shift, throttle a dependency, corrupt a document — and use Sections 5-9 of `Continuous_Monitoring_Comprehensive_Notes.md` (in this same folder) as your root-cause playbook to fix it properly, not just restart the service
- [ ] **Definition of done:** you have a dated log entry: what broke, what your dashboards/traces showed, what you changed, and why that fix (not a bigger one) was the right call

---

## 3. The single highest-leverage habit: keep a build log

Not a diary — a running log with four columns: **date, what I changed, what broke or what I measured, why I made that call instead of an alternative**. This is what actually makes an interviewer believe you built it, because it's full of specific, non-generic detail (real cost numbers, real latency numbers, a real wrong turn you corrected) that no tutorial-follower can improvise on the spot.

Suggested minimum entries to have by day 60:
- One real cost comparison (e.g. Bedrock vs. a SageMaker endpoint for your workload)
- One real latency number you measured and one thing you did to improve it
- One incident: something drifted or broke, how you noticed (which dashboard/alert), and what you changed
- One infrastructure decision you'd make differently next time, and why

---

## 4. Quick-reference: tool → week mapping

| Tool | Introduced in | Role in the project |
|---|---|---|
| FastAPI | Week 1 | Serving layer |
| OpenSearch | Week 1 | Vector retrieval |
| Bedrock | Week 1 | Model inference |
| Docker | Week 1 | Packaging (multi-stage) |
| ECS Fargate | Week 2 | Production deployment |
| LangGraph | Week 3 | Agent workflow/state |
| Lambda | Week 4 | Event-driven ingestion |
| EventBridge | Week 4 | Automation trigger |
| Langfuse | Week 5 | LLM observability/tracing |
| CloudWatch | Week 5 | Logs |
| Prometheus/Grafana | Week 6 | Infra monitoring |
| IAM / Secrets Manager | Week 6 | Security |
| Evidently AI | Week 7 | Drift detection |
| GitHub Actions | Week 7 | CI/CD |
| Terraform | Week 8 | IaC |
| SageMaker | Week 8 | Inference/retraining comparison |

This mirrors the same lifecycle taught in Modules 14, 18, and 19 of the course (Observability, Tracing, Drift) and the capstone framing in Module 26 — this project *is* a working instance of that lifecycle, just on AWS-native services instead of the generic stack used in the course modules.
