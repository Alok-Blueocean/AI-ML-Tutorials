# Orchestration (Airflow, Prefect, Dagster) — Scenario-Based Q&A

**Situation:** A small team's nightly RAG re-indexing pipeline is a single Python script triggered by a cron job, and it just failed halfway through, leaving the vector index in an inconsistent state with no easy way to tell what actually ran. What would you do and why?

Model answer: Point out this is exactly the failure mode orchestration tools exist to prevent — a cron-triggered monolithic script has no dependency modeling, no partial-failure recovery, and no visibility into what ran versus what didn't. Migrate the pipeline into an orchestrator (Airflow, Prefect, or Dagster) with each logical step (fetch documents, embed, update index, evaluate) as its own task, so a failure at the "update index" step can retry or resume from there instead of forcing a full re-run from scratch. Make each task idempotent so retries and backfills are safe, and use the orchestrator's UI/run history to answer "what ran and what failed" immediately instead of reconstructing it from logs after the fact.

---

**Situation:** Your team's model retraining pipeline has grown to a dozen interdependent steps hand-coded as sequential function calls in one script, and adding a thirteenth step recently caused two earlier, supposedly-unrelated steps to silently break. What would you do and why?

Model answer: Diagnose this as implicit-dependency risk — when task order and dependencies live only in the order function calls happen to appear in a script, it's easy for a change in one place to have unintended effects elsewhere with no structural safeguard. Model the pipeline explicitly as a DAG in an orchestrator, where each task declares its actual inputs and dependencies rather than relying on being called in the right order. This makes dependencies visible and enforced by the framework, and means adding a new step only affects the tasks that actually declare a dependency on it, rather than everything downstream in the script.

---

**Situation:** A teammate wants to add a new nightly evaluation-suite job by literally copy-pasting the existing RAG-refresh DAG and swapping a few lines, resulting in near-duplicate pipeline definitions maintained separately. What would you do and why?

Model answer: Push back on copy-paste as the default extension mechanism — near-duplicate DAGs mean every future bug fix or improvement has to be applied twice (and will eventually be applied to only one, causing silent divergence). Recommend parameterizing the existing pipeline structure instead, so the same DAG/flow definition can be reused across the RAG-refresh and evaluation-suite use cases by passing different parameters (which documents, which eval suite, which schedule) rather than duplicating the underlying task graph. This is exactly the reusability/parameterization benefit orchestrators are built to provide over ad hoc scripts.

---

**Situation:** Your team needs to reprocess three months of historical data through a pipeline after fixing a bug in the embedding step, and someone proposes writing a one-off script that loops over each past date and re-runs the pipeline manually. What would you do and why?

Model answer: Point out this is exactly what backfills are for, and a hand-rolled loop reinvents (poorly) something the orchestrator likely already supports natively. Use the orchestrator's built-in backfill capability to re-run the pipeline for each past date in the affected range, which gets you the same retry, logging, and monitoring behavior as a normal scheduled run instead of an unobserved one-off script. Before running the full backfill, verify the fixed step is idempotent so re-running it for dates that already have (now-stale) outputs cleanly overwrites rather than duplicates data.

---

**Situation:** A pipeline step that calls an external LLM API for evaluation intermittently fails due to rate limiting, and currently the entire nightly pipeline run is marked failed and has to be manually restarted from the beginning. What would you do and why?

Model answer: This is a textbook case for task-level retries rather than pipeline-level failure. Configure the specific task that calls the external API with a retry policy (a bounded number of attempts with exponential backoff) so a transient rate-limit error resolves itself without human intervention, and without forcing upstream steps (data fetch, embedding, indexing) that already succeeded to re-run unnecessarily. Reserve full pipeline failure for genuinely unrecoverable errors, and alert on repeated retry exhaustion specifically, since that's a stronger signal of a real problem than a single transient failure.

---

**Situation:** Leadership asks why a model's performance regression, discovered two weeks after deployment, took so long to trace back to a specific pipeline run and a specific upstream data change. What would you do and why?

Model answer: Point to the lack of run-level visibility and lineage as the root cause, not the regression itself — without an orchestrator's run history, correlating "which pipeline run produced this model version, and what data/parameters fed into it" requires manually reconstructing logs and timestamps after the fact. Going forward, rely on the orchestrator's UI and run history to answer this directly: each run's status, duration, and what data flowed between steps should be queryable immediately. If the team is using Dagster specifically, treating the model and its input datasets as first-class tracked assets makes "what produced this artifact" a built-in lineage query rather than an investigation.

---

**Situation:** Your team is choosing between Airflow, Prefect, and Dagster for a new LLMOps pipeline, and one engineer argues for Airflow purely because "it's the industry standard." What would you do and why?

Model answer: Push back on choosing by reputation alone and match the tool to the team's actual needs. Airflow is the right call when you need broad ecosystem/integration support and are operating at an org scale with many scheduled batch jobs across teams — its maturity is a real asset there. If the team wants a lighter, more Pythonic development experience with less boilerplate and easier local testing, Prefect is a better fit. If the project cares deeply about data/model lineage and treating datasets and models as trackable, testable assets — which matters a lot for an LLMOps pipeline producing embeddings, indexes, and model versions — Dagster's asset-first model is worth the switch in mental model. Recommend a short spike building the same small pipeline in the two strongest candidates before committing, rather than deciding on reputation alone.

---

**Situation:** A daily document-ingestion task in your pipeline is not idempotent — running it twice for the same day inserts duplicate documents into the vector index — and a recent retry after a transient failure caused exactly that. What would you do and why?

Model answer: Treat this as a design bug to fix at the task level, not something to work around by disabling retries. Make the ingestion task idempotent: key inserts by a stable document ID and use an upsert (insert-or-replace) rather than a blind insert, so re-running the same task for the same day produces the same end state regardless of how many times it runs. This is a general principle worth applying to every task in the pipeline, not just this one, since retries and backfills are core orchestrator features that become actively dangerous without idempotent tasks underneath them.

---

**Situation:** Your orchestrated pipeline calls out to several different systems — a SQL warehouse, a Python training job, an external LLM API, and a container-based evaluation job — and a teammate suggests it would be simpler to just write custom scripts for each and glue them together with shell commands. What would you do and why?

Model answer: Push back — this is precisely the heterogeneous-coordination problem orchestrators are built to solve, and a hand-rolled shell-glue approach reintroduces the visibility, retry, and dependency problems the team presumably adopted an orchestrator to avoid in the first place. Keep each heterogeneous step (SQL query, training job, API call, container job) as its own task within the same DAG/flow, using the orchestrator's native operators/integrations for each system where available. This keeps one place to see the whole pipeline's status and history, rather than fragmenting visibility across several independently-triggered scripts.

---

**Situation:** A pipeline that refreshes a RAG index runs on a fixed nightly schedule, but the underlying source documents actually update at unpredictable times throughout the day, meaning the index is sometimes hours stale and sometimes runs for no reason. What would you do and why?

Model answer: Reconsider whether a fixed schedule is even the right trigger here — if document updates are event-driven rather than time-driven, an event/sensor-based trigger (kick off the re-index pipeline when new or changed documents actually arrive, via a webhook, file-system event, or message queue) will produce fresher results with less wasted work than a fixed nightly cron-equivalent schedule. If a fully event-driven trigger isn't feasible yet, a reasonable middle ground is a more frequent schedule (hourly) combined with a cheap "did anything actually change" check at the start of the run, so the expensive re-embedding steps only execute when there's real work to do.

---

**Situation:** Your team's orchestrator runs are all successful, but a stakeholder wants to know how they'd find out if a pipeline that's supposed to run nightly simply didn't run at all last night, since a missing run produces no failure to alert on. What would you do and why?

Model answer: Point out that "did it run" is a different signal from "did it succeed," and a monitoring setup that only watches for task failures will miss a scheduler outage or a silently disabled/paused DAG entirely. Add a separate freshness/heartbeat check — an alert that fires if the expected run simply didn't start or complete within its expected window, independent of whether any individual run reported success or failure. Most orchestrators expose this as a "missed SLA" or "run freshness" concept; wire it into the same alerting channel as task failures so a silent no-show gets the same attention as a loud crash.

---

**Situation:** After a pipeline redesign, a step that used to run in 5 minutes now takes 45 minutes, but nobody notices until a downstream model deployment is delayed by hours. What would you do and why?

Model answer: This points to a gap in per-task performance visibility, not just overall pipeline success/failure tracking. Use the orchestrator's run history to establish a baseline duration per task and add alerting on tasks that deviate significantly from their historical runtime, not just on outright failures — a task that "succeeds" but takes 9x longer than normal is itself an anomaly worth surfacing before it cascades into a missed deployment deadline. Once flagged, investigate whether the redesign introduced an inefficiency (e.g., lost parallelism, a new synchronous dependency) that can be fixed at the task level.

---

**Situation:** A junior engineer asks why the team bothers with an orchestrator at all for a simple two-step pipeline (fetch data, train a small model) that rarely fails and runs in under a minute. What would you do and why?

Model answer: Give a direct, honest answer rather than reflexively defending orchestration for its own sake — for a genuinely simple, low-stakes, rarely-failing two-step pipeline, a plain script with basic error handling may be entirely sufficient, and adding orchestration infrastructure here would be over-engineering. Recommend reaching for an orchestrator once the pipeline has real dependencies, a need for scheduling/triggering beyond a trivial cron entry, meaningful failure modes worth retrying, or is one of several similar pipelines that would benefit from shared visibility and reuse — not by default for every pipeline regardless of size.

---

**Situation:** Your team's orchestrated model-training pipeline is version-controlled, but changes to it get pushed directly to the production orchestrator instance without any code review, and a recent change silently broke a downstream evaluation step for a week before anyone noticed. What would you do and why?

Model answer: Treat this as an infrastructure-governance gap — the pipeline definition is production infrastructure, not a disposable script, and should go through the same code review and CI process as any other production code change. Require pull requests and review for pipeline definition changes, and add a lightweight CI check that validates the DAG/flow can at least be parsed and its dependencies resolved before it's ever deployed to the production orchestrator instance. Combine this with the per-task performance/failure monitoring from other scenarios so a change that does slip through is caught by monitoring quickly rather than silently for a week.

---

**Situation:** A pipeline step is defined to embed and index new documents, but it was written years ago for a specific dataset and now needs to be reused, slightly modified, for three different document sources with different schemas. What would you do and why?

Model answer: Resist copy-pasting the task three times with small edits, and instead parameterize the existing task to accept a schema/source configuration as an input, reusing the same underlying task logic across all three document sources. This is the reusability benefit orchestrators are specifically designed to support — the same pipeline structure, parameterized rather than duplicated, so a bug fix or improvement to the embedding logic only needs to be made once and automatically applies to all three sources going forward.
