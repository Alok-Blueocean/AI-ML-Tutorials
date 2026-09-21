# Architecture Deep-Dive — Module 12: Deployment Quality Gates and Release Dashboards

This file complements `tutorial.md` with larger, more detailed ASCII architecture diagrams,
a full sequence diagram of a release passing (and failing) through the gate system, and a
decision tree for choosing which trigger tier, gate type, and dashboard alerting strategy to
apply to a given change. Read `tutorial.md` first for the theory; use this file as the
visual/reference layer for system design and interview whiteboarding.

---

## 1. Full-Stack Reference Architecture — Triggers, Gates, and Dashboards Together

This is the end-to-end picture: every trigger from §3.1, feeding the gate logic from §3.2,
feeding the dashboard/alerting layer from §3.3, all sharing one metrics store as the source
of truth.

```
                                    CHANGE EVENTS (four independent sources)
   +------------------+   +------------------+   +------------------+   +------------------+
   |  Pull Request     |   |  Push to main     |   |  Cron schedule    |   |  Data pipeline    |
   |  opened/updated   |   |  (merge)          |   |  (weekly)         |   |  (corpus/FT data  |
   |                    |   |                    |   |                    |   |  update)           |
   +--------+----------+   +--------+----------+   +--------+----------+   +--------+----------+
            |                        |                        |                        |
            v                        v                        v                        v
   +------------------+   +------------------+   +------------------+   +------------------+
   |  TIER 1 EVAL      |   |  TIER 2 EVAL      |   |  TIER 3 EVAL      |   |  TIER 2/3 EVAL    |
   |  50 examples       |   |  200 examples     |   |  500 prod-sampled |   |  (affected slice)  |
   |  correctness only  |   |  all metrics       |   |  hallucination +  |   |  full or targeted  |
   |  ~5 min, ~$0.36    |   |  ~20 min, ~$1.44   |   |  drift             |   |  re-eval            |
   +--------+----------+   +--------+----------+   +--------+----------+   +--------+----------+
            |                        |                        |                        |
            v                        v                        v                        v
   +-----------------------------------------------------------------------------------------+
   |                          EVALUATION RESULTS  (candidate_metrics.json)                     |
   |         { eval_score, p95_latency_ms, hallucination_rate, format_valid_rate, ... }         |
   +---------------------------------------+---------------------------------------------------+
                                            |
                                            v
   +-----------------------------------------------------------------------------------------+
   |                              QUALITY GATE  (evals/gate.py)                                |
   |                                                                                             |
   |   +----------------------+        +----------------------+                                 |
   |   |  ABSOLUTE GATE CHECK  |        |  RELATIVE GATE CHECK  |<--- fetches baseline metrics     |
   |   |  fixed floors:        |        |  vs. production        |     from Model/Prompt Registry  |
   |   |  hallucination<3%     |        |  baseline               |     (Module 03 concepts)         |
   |   |  format_valid==100%   |        |  score delta >= -1%     |                                 |
   |   |  score>=0.82          |        +-----------+-------------+                                 |
   |   |  p95<=1200ms          |                    |                                                |
   |   +-----------+-----------+                    |                                                |
   |               |                                 |                                                |
   |          both must pass                          |                                                |
   |               +---------------------------------+                                                |
   |                              |                                                                    |
   +------------------------------+----------------------------------------------------------------+
                                  |
                  +---------------+----------------+
                  |                                 |
              FAIL (any)                          PASS (all)
                  |                                 |
                  v                                 v
   +-----------------------------+    +-----------------------------------------+
   |  sys.exit(1)                 |    |  sys.exit(0)                             |
   |  - PR: block merge, comment   |    |  - PR: allow merge                        |
   |    on PR with reason           |    |  - Push: register artifact in registry    |
   |  - Push: block artifact        |    |    (Module 03), proceed to canary deploy  |
   |    registration, block deploy  |    |    (Module 06)                             |
   |  - Schedule/data: page on-call |    |  - Schedule: no action, healthy             |
   |    (already live, can't block  |    +--------------------+----------------------+
   |    the past — only alert)      |                          |
   +---------------+-----------------+                          |
                   |                                            |
                   +--------------------+-----------------------+
                                        |
                                        v
                    +----------------------------------------------+
                    |     METRICS STORE  (Prometheus / OTel /        |
                    |     experiment tracker — Module 13/14)          |
                    |     every run, every trigger, every gate result |
                    |     written here as one continuous timeline     |
                    +----------------------+---------------------------+
                                           |
                                           v
                    +----------------------------------------------+
                    |            GRAFANA DASHBOARD                    |
                    |  Row 1: headline traffic light + composite score |
                    |  Row 2: dimension breakdown (bar gauges)         |
                    |  Row 3: 30-day trend (time series)                |
                    |  Row 4: operational (cost, tokens, error rate)    |
                    +----------------------+---------------------------+
                                           |
                                           v
                    +----------------------------------------------+
                    |   ALERT RULES                                    |
                    |   hard-block panels: fire immediately (for: 0s)   |
                    |   soft/relative panels: short debounce            |
                    |   -> pages on-call / posts to release channel     |
                    +----------------------------------------------+
                                           |
                                           v
                              RELEASE MANAGER: ship / hold / rollback
```

---

## 2. Sequence Diagram — A Release Candidate's Journey Through the Gate System

This traces one concrete version, `v2.4.0`, from PR to (attempted) production, showing both
the happy path and a blocked path at push time.

```
 Engineer        GitHub          Tier-1 Job       Tier-2 Job        Registry         Grafana/
 (dev)           Actions         (PR gate)        (push gate)       (Module 03)      Dashboard
   |                |                 |                 |                 |               |
   |--open PR------>|                 |                 |                 |               |
   |                |--trigger------->|                 |                 |               |
   |                |                 |--run 50 ex.---->|                 |               |
   |                |                 |  eval_score=0.87|                 |               |
   |                |                 |  (no hard block)|                 |               |
   |                |<--PASS----------|                 |                 |               |
   |<--PR checks OK-|                 |                 |                 |               |
   |                |                 |                 |                 |               |
   |--merge PR----->|                 |                 |                 |               |
   |                |--trigger--------------------------|>|               |               |
   |                |                                    |--run 200 ex.-->|               |
   |                |                                    |  eval_score=0.79|              |
   |                |                                    |  format_valid=0.97|            |
   |                |                                    |  (BELOW 0.82,     |            |
   |                |                                    |   format<100%!)   |            |
   |                |                                    |--fetch baseline->|             |
   |                |                                    |<--v2.3.1 metrics-|              |
   |                |                                    |--check gates----|               |
   |                |                                    |  ABSOLUTE: FAIL  |               |
   |                |                                    |   (score, format)|               |
   |                |                                    |  HARD BLOCK: yes  |              |
   |                |                                    |--sys.exit(1)----|               |
   |                |<--pipeline FAILED------------------|                 |               |
   |<--PR comment: "BLOCKED: format_valid_rate=0.97      |                 |               |
   |    < 1.00 (HARD BLOCK), eval_score=0.79 < 0.82"------|                |               |
   |                |                                    |--write gate_reason.json--------->|
   |                |                                    |  status=red      |               |
   |                |                                    |------------------------------->  |
   |                |                                    |                 |         panel turns RED
   |                |                                    |                 |         no artifact registered
   |                |                                    |                 |               |
   |--fix format bug, push new commit v2.4.1------------->|                 |               |
   |                |--trigger--------------------------|>|               |               |
   |                |                                    |--run 200 ex.-->|               |
   |                |                                    |  eval_score=0.85|               |
   |                |                                    |  format_valid=1.00|             |
   |                |                                    |  ABSOLUTE: PASS  |               |
   |                |                                    |--fetch baseline->|              |
   |                |                                    |<--v2.3.1: 0.83--|               |
   |                |                                    |  RELATIVE: PASS  |               |
   |                |                                    |  (0.85 >= 0.82) |               |
   |                |                                    |--sys.exit(0)----|               |
   |                |                                    |--register------>|register v2.4.1 |
   |                |                                    |  as candidate    |as staging      |
   |                |                                    |                 |               |
   |                |                                    |--write gate_reason.json--------->|
   |                |                                    |  status=green    |               |
   |                |                                    |------------------------------->  |
   |                |                                    |                 |         panel turns GREEN
   |<--pipeline PASSED, proceed to canary deploy (Module 06)---------------|               |
```

Key details this sequence makes explicit:
- The PR-level Tier-1 pass does **not** guarantee the Tier-2 push-level pass — they check different sample sizes and different metrics, by design.
- The first push attempt (`v2.4.0`) hits **two absolute failures simultaneously** (score and format), one of which (`format_valid_rate`) is a hard block — meaning even if the score had cleared 0.82, this release would still be blocked.
- The registry only receives a new artifact entry **after** the gate passes — the gate sits structurally *before* registration, not after.
- The dashboard is updated on every run regardless of pass/fail, so the 30-day trend view captures the failed attempt too — this is deliberate: seeing "this version failed and was fixed same-day" is valuable history, not noise to hide.

---

## 3. Decision Tree — Which Trigger Tier, Gate Type, and Alert Strategy to Use

Use this when designing a gate for a new artifact type (a new prompt template, a new RAG
index, a new fine-tuned checkpoint, a new tool-calling agent) and you need to decide where it
plugs into the trigger/gate/alert framework.

```
                         START: A new kind of change needs a quality gate
                                            |
                                            v
                     Q1: Can this change reach real user traffic
                         without a human approving a merge/deploy?
                          /                                        \
                        NO                                        YES
                         |                                          |
                         v                                          v
              Research/experimental branch              Q2: Does the change originate from
              -> light or no gate; use gates            a code/config diff, or from data/
              only once it's on the path to               corpus/model drift with no diff?
              production                                          |
                                                    +---------------+----------------+
                                                    |                                 |
                                              CODE / CONFIG DIFF                 DATA / DRIFT
                                                    |                                 |
                                                    v                                 v
                                    Q3: Is this diff proposed (PR)          Use Tier 3 (scheduled)
                                    or already merged (push)?                or data-update trigger.
                                          |                    |             Cannot "block" already-
                                          v                    v             live traffic -> route
                                     Use TIER 1                Use TIER 2    failures to PAGING, not
                                     (fast, cheap,              (full eval,   to a merge-block.
                                      correctness-only,         all metrics,
                                      blocks PR merge)          blocks artifact
                                                                registration)
                                                                      |
                                                                      v
                                          Q4: Does this metric represent a failure
                                          mode with NO acceptable "ship + fast-follow"
                                          story (broken output, unsafe/hallucinated
                                          content)?
                                              /                              \
                                            YES                              NO
                                             |                                |
                                             v                                v
                                    Mark as HARD BLOCK                Standard absolute gate
                                    - zero tolerance                   (score, latency SLA, etc.)
                                    - no override path                 - may allow a documented
                                    - alert fires immediately            human-override/waiver
                                      (for: 0s, no debounce)             process for edge cases
                                             |                                |
                                             v                                v
                                    Q5: Does this metric need a RELATIVE
                                    (vs. production baseline) comparison
                                    in addition to the absolute floor?
                                              /                              \
                                          YES (quality/composite)          NO (usually latency,
                                             |                              cost — absolute SLA is
                                             v                              sufficient on its own)
                                  Add relative gate:                             |
                                  candidate >= baseline - tolerance              v
                                  (typically 1 percentage point)          Absolute gate only
                                             |
                                             v
                            Q6: Is the underlying metric noisy/sampled
                            (LLM-judge score, small-sample eval)?
                                  /                                    \
                                YES                                   NO
                                 |                                     |
                                 v                                     v
                    Use repeat + repeat-min-pass              Single-run threshold
                    (re-run N times, require majority           check is sufficient
                    pass) before failing the gate —
                    avoid lowering the threshold to
                    compensate for flakiness
                                 |
                                 v
                    DONE: wire gate result into
                    - sys.exit() in CI (block path)
                    - gate_reason.json (dashboard + PR comment)
                    - Grafana alert rule (hard block: immediate;
                      soft gate: short debounce)
```

**How to read this tree in an interview setting:** the two branch points that most distinguish
a senior answer from a junior one are **Q2** (recognizing that data/drift changes need a
fundamentally different response — paging, not merge-blocking, because the bad version is
already live) and **Q4** (being able to name *which specific metrics* deserve hard-block status
and defend why latency typically does not get the same treatment as hallucination or format
validity).

---

## 4. Grafana Panel Layout — Wireframe Detail

A more detailed wireframe of the dashboard introduced in `tutorial.md` §3.3, showing exact
panel types and the query shape each would use against a Prometheus-compatible metrics store.

```
+==================================================================================================+
|  LLM RELEASE READINESS — v2.4.1 candidate vs. v2.3.1 production           [time range: last 30d] |
+==================================================================================================+
| +----------------+  +----------------+  +----------------+  +---------------------------------+  |
| | COMPOSITE SCORE|  | STATUS          |  | vs. BASELINE    |  | LAST EVAL RUN                    |  |
| |  Stat panel     |  | Traffic light    |  | Stat + arrow     |  | Stat panel (timestamp)           |  |
| |  query:          |  | panel            |  | query:            |  | query:                             |  |
| |  avg(llm_eval_   |  | derived from     |  | candidate -      |  | max(llm_eval_run_timestamp)        |  |
| |  composite_score |  | thresholds below |  | production score  |  |                                     |  |
| |  {version=      |  |                  |  |                    |  |                                     |  |
| |  "v2.4.1"})      |  |  GREEN           |  |  +0.02 (+2.4%)     |  |  2026-07-29 06:03 UTC               |  |
| |  = 0.85          |  |                  |  |                    |  |                                     |  |
| +----------------+  +----------------+  +----------------+  +---------------------------------+  |
+==================================================================================================+
| DIMENSION BREAKDOWN (bar gauges, one per dimension, thresholds as colored zones)                   |
| +-----------------+  +-----------------+  +-----------------+  +-----------------+  +------------+ |
| | correctness      |  | format_valid     |  | hallucination_   |  | p95_latency_ms   |  | relevance   | |
| | 0.88  [green]     |  | 1.00  [green]     |  | rate              |  | 940ms [green]     |  | 0.86 [green] | |
| |                    |  |                    |  | 0.011 [green]     |  |                    |  |             | |
| +-----------------+  +-----------------+  +-----------------+  +-----------------+  +------------+ |
+==================================================================================================+
| 30-DAY TREND (time series, one line per dimension, threshold line overlaid, annotations for        |
| deploy events)                                                                                       |
|                                                                                                        |
|  0.90 |                                              *---*---*  <- v2.4.1 (fixed format bug)          |
|  0.85 |                              *---*---*---*--/                                                 |
|  0.82 |------------------------------------------------------------------- absolute threshold line   |
|  0.80 |                    x (v2.4.0 attempt, blocked, never registered)                               |
|  0.75 |         *---*---*-/                                                                            |
|       +----------------------------------------------------------------------------> time (30 days)   |
|             deploy: v2.2.0      deploy: v2.3.1        BLOCKED: v2.4.0   deploy: v2.4.1                  |
+==================================================================================================+
| OPERATIONAL METRICS (cost, token volume, request rate, error rate — separate from quality/safety)   |
| +----------------------+  +----------------------+  +----------------------+  +------------------+ |
| | Weekly eval spend      |  | Tokens/day             |  | Requests/min           |  | Error rate 5xx    | |
| | $14.40 (budget: $25)    |  | 2.1M (7d avg)           |  | 340 rpm                 |  | 0.02%              | |
| +----------------------+  +----------------------+  +----------------------+  +------------------+ |
+==================================================================================================+
```

The horizontal rule between "dimension breakdown / trend" and "operational metrics" is
deliberate, not cosmetic — it is the same operational-vs-evaluation separation industry
dashboards (Grafana Cloud's prebuilt GenAI dashboard set among them) have converged on, so that
a viewer's eye is never asked to compare a cost number against a quality-threshold color on the
same visual scale.
