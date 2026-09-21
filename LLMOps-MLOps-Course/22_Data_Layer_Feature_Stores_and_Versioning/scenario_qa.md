# Data Layer: Feature Stores and Versioning — Scenario-Based Q&A

**Situation:** A model that scored well in evaluation six months ago is being revisited after a bug report, but nobody can find the exact training CSV that was used — it was overwritten in the shared drive by a "cleaned" version. What would you do and why?

Model answer: Treat this as a process failure, not a one-off inconvenience — without the exact training data, the model is fundamentally undebuggable and unreproducible. Immediately start versioning the *current* dataset with DVC or LakeFS so this can never recur, tagging each dataset version to the model/experiment run that consumed it (via the experiment tracker from Module 13). For the missing historical version, check whether any DVC-tracked hashes, S3 object versioning, or backup snapshots survived the overwrite; if nothing recoverable exists, document the gap honestly rather than guessing, and retrain against the best reconstructable approximation with a clear note in the model card that lineage before this point is broken.

---

**Situation:** A data scientist proposes using a plain shared network drive with folder names like `data_final_v2_USE_THIS` instead of adopting DVC, arguing "we're a small team, this is simpler." What would you do and why?

Model answer: Push back with a concrete failure mode rather than an abstract principle: folder-name versioning has no atomic commit, no diffing, no way to know which exact file version paired with which code commit, and it invites exactly the "final_v2_USE_THIS" ambiguity happening right now. DVC's overhead for a small team is minimal — `dvc add`, `dvc push`, `dvc pull` layered on a Git workflow the team already uses — and it immediately gives every dataset a hash-addressed, code-commit-linked history. Frame it as "you're already paying the cost of ad-hoc versioning in confusion and re-work; DVC just makes that cost visible and manageable instead of invisible and growing."

---

**Situation:** Your recommendation model performs noticeably better offline than it does in production, and after investigation you find the "average order value in the last 30 days" feature is computed with slightly different logic in the batch training pipeline versus the real-time serving API. What would you do and why?

Model answer: Name this precisely as training-serving skew and treat it as a correctness bug, not a modeling problem — no amount of retraining fixes a model that's being fed differently-computed inputs at serving time. Consolidate the feature computation into a single definition (a feature store like Feast) so both the offline store used for training and the online store used for serving derive from the same feature-view logic, rather than two independently maintained code paths. Add a monitoring check that periodically compares a sample of online-served feature values against what the offline pipeline would compute for the same entity/timestamp, so skew is caught automatically instead of being discovered through a performance regression.

---

**Situation:** Leadership wants to launch a fraud model faster and proposes skipping point-in-time correctness checks when joining features to historical training labels, since "the current feature values are what we have anyway." What would you do and why?

Model answer: Explain the specific mechanism of the leak: joining a feature's *current* value onto a historical label means the model can see information that didn't exist yet at the time of that historical event (e.g., "total lifetime fraud flags" computed today, joined onto a transaction from a year ago) — this inflates offline metrics with information the model will never have at real prediction time, producing a model that looks excellent in evaluation and fails in production. Insist on point-in-time joins (what Feast and most feature stores provide by default) even under time pressure, because the alternative isn't "faster" — it's shipping a model whose true production performance is unknown and likely much worse than the eval numbers suggest.

---

**Situation:** Three data scientists on your team have each independently written slightly different SQL to compute "customer lifetime value," and their models disagree on which customers are high-value. What would you do and why?

Model answer: This is the reuse/consistency problem a feature store's registry exists to prevent — three definitions of the same concept means three sources of disagreement that have nothing to do with modeling skill. Get the team to converge on one canonical `customer_lifetime_value` feature definition, register it in Feast (or an equivalent catalog) with clear documentation of its computation window and edge cases (e.g., how refunds are handled), and require new projects to check the registry before writing a new feature from scratch. Treat this as a one-time consolidation cost that pays for itself the next time someone needs the same feature.

---

**Situation:** Your team is about to run an ETL job against the production data lake to test a new transformation, and everyone is nervous because a bad run last quarter corrupted a table that took two days to restore from backup. What would you do and why?

Model answer: Use LakeFS's branching model instead of testing directly against production data — create a branch of the data lake, run the ETL job against the branch, inspect the results, and only merge into the main branch once you're confident the transformation is correct. This gives you the isolation of a sandbox without the cost of physically duplicating petabytes of data (copy-on-write), and if anything goes wrong, you simply discard the branch instead of restoring from backup. Recommend this become the default workflow for any exploratory or risky pipeline change against the shared lake, not a one-off precaution.

---

**Situation:** An auditor asks your team to prove exactly what data trained the credit-scoring model currently in production, including any subsequent corrections to that data. What would you do and why?

Model answer: If dataset versioning and experiment tracking were done properly, this is a lookup, not an investigation: pull the model version from the registry, find its linked experiment run, and read off the exact DVC/LakeFS commit hash (or dataset version tag) that was used, plus the Git commit of the training code. If corrections were made to the source data afterward, those are separate, later versions — the audit trail should make clear which correction happened when and whether the model was retrained against it. If this lookup isn't possible today, that's the real finding: recommend closing the gap by wiring dataset version → training run → model version as a mandatory, automated link going forward, not something reconstructed by hand under audit pressure.

---

**Situation:** Your RAG system's retrieval quality quietly degraded, and after investigation the retrieval corpus (a set of markdown docs) had been edited directly in place by a content team with no tracking of what changed or when.

Model answer: Recognize that "data" in an LLM pipeline now includes the retrieval corpus, and it deserves the same versioning discipline as tabular training data — an untracked edit to source documents is functionally identical to an untracked edit to a training CSV. Bring the corpus under DVC or LakeFS versioning so every content change is a commit with a diff, tie re-indexing/re-embedding runs to specific corpus versions, and give the content team a lightweight review or approval step before edits land, rather than direct in-place writes. Retroactively, diff recent corpus states against backups if available to identify what changed and re-evaluate retrieval quality against the corrected version.

---

**Situation:** Your online feature store (Redis-backed) and offline feature store (data warehouse) have drifted out of sync after a failed materialization job, and predictions in production are now using stale feature values. What would you do and why?

Model answer: First contain the immediate problem — check monitoring/alerting on the materialization pipeline (or add it if it doesn't exist) to detect when a scheduled sync fails silently, and manually trigger a fresh materialization to bring the online store current. Then address the root cause: a feature store's entire value proposition collapses if the offline-to-online sync isn't reliable, so this needs the same monitoring rigor as any other production pipeline — freshness checks (how old is the newest value in the online store), alerting on failed runs, and ideally automatic retries. Communicate to stakeholders that predictions during the drift window may have used stale data, since that affects how much to trust recent model outputs.

---

**Situation:** A new team member asks why you can't just store all your training data directly in Git, since "it's just files and Git already does version control." What would you do and why?

Model answer: Explain the specific limitation: Git's diffing and storage model is built for text, and it performs very poorly on large binary files (images, parquet, model weights) — every version gets stored close to in full, repo size balloons, clone/fetch times explode, and diffs are meaningless for binary content. DVC solves this by keeping only small pointer files (hashes, metadata) in Git while the actual data lives in object storage (S3/GCS/Azure Blob), giving you Git's commit/branch/rollback semantics without Git's binary-file performance problems. The mental model to convey: "Git tracks that the data changed and to what version; it doesn't need to physically store the data itself."

---

**Situation:** Your company is fine-tuning an LLM on a proprietary support-ticket dataset, and legal asks whether you can identify and remove a specific customer's data from all model training history if they invoke a deletion request (GDPR right to erasure). What would you do and why?

Model answer: This is only answerable if the fine-tuning dataset was versioned with enough granularity to trace which records went into which training run — if the data was tracked from the start (DVC/LakeFS commits tied to each fine-tuning experiment), you can identify every dataset version containing that customer's records and determine which deployed model versions were trained on them. If it wasn't tracked, the honest answer is that a clean per-customer removal may not be technically feasible without retraining from a corrected dataset, which is a costly and consequential gap to have discovered only when a legal request arrives. Recommend building version-level data lineage into any pipeline touching personal data as a compliance requirement, not just an engineering nicety.

---

**Situation:** A teammate wants to skip building a feature registry, arguing that with only two models in production, "everyone just knows what features exist" informally. What would you do and why?

Model answer: Agree this is low-risk today but point out that feature sprawl compounds quietly — two models becomes five, one engineer becomes a rotating team, and "everyone just knows" degrades into "nobody's sure if this feature already exists" within a year, by which point migrating to a registry means auditing years of undocumented tribal knowledge. Propose registering features incrementally as they're built, even with a small team, since the marginal cost of registering a feature in Feast at creation time is small compared to reconstructing that documentation later. Frame it as a cheap insurance policy against a cost that scales with team and model count, not a bureaucratic requirement for its own sake.

---

**Situation:** During a postmortem, you discover a training run consumed a dataset version that was later found to contain a labeling bug affecting 8% of records, and the model built on it is still serving production traffic. What would you do and why?

Model answer: Use the dataset versioning history to precisely scope the blast radius: identify every model version trained on the affected dataset version (this is exactly the lookup a proper version-to-model link enables), and assess whether the 8% labeling error is large enough to materially affect the currently deployed model's behavior. Fix the labels, create a new dataset version (not an edit to the old one, to preserve history), retrain, and re-evaluate against the golden set before rolling out a replacement. Use this incident to argue for a data-quality check (e.g., automated label-distribution or anomaly checks) as a gate before a dataset version is allowed to feed a production training run.

---

**Situation:** Your team is choosing between DVC and LakeFS for a new project, and a stakeholder asks for a one-sentence justification for whichever is picked. What would you do and why?

Model answer: Base the choice on scale and organizational shape rather than feature checklists: if this is a project-level concern — a handful of datasets tracked alongside a specific codebase, by a small team already comfortable with Git — DVC's lightweight, repo-centric model is the better fit and the one-sentence justification is "it's Git-like versioning for our specific project's data, with minimal new infrastructure." If multiple teams read and write a shared data lake and need isolated branches to test pipelines without risking shared production data, LakeFS's infrastructure-level branching is the better fit, justified as "it gives every team a safe sandbox on the same lake without duplicating data." Avoid picking based on novelty — match the tool to whether the problem is "one project's datasets" or "our whole data lake."

---

**Situation:** A machine learning engineer wants to compute a new streaming feature (rolling 5-minute transaction count) but is unsure whether to build custom streaming infrastructure or use the feature store's existing online store. What would you do and why?

Model answer: Default to extending the existing feature store's online-store pattern rather than building parallel streaming infrastructure, since the entire point of a feature store is to be the single place features are computed and served — a bespoke streaming pipeline outside that system immediately reintroduces the training-serving skew and discoverability problems the feature store exists to solve. Check whether Feast (or whichever store is in use) supports streaming sources natively; if it doesn't cleanly support sub-minute freshness, that's a real constraint worth raising, but the fallback should still register the feature in the catalog and clearly document that it's computed outside the standard pipeline, rather than silently forking the architecture.

---

**Situation:** After an incident, you learn a production model was silently retrained nightly on a rolling data window with no dataset versioning at all — every night's training data overwrites the previous night's, by design, "to always use the freshest data." What would you do and why?

Model answer: Separate two different needs that got conflated: wanting fresh data for training is reasonable, but losing the ability to reproduce or roll back any specific model version is not an acceptable tradeoff for that freshness. Introduce dataset versioning into the nightly pipeline so each night's training window becomes an immutable, addressable snapshot (even if it's later deleted for storage cost, it should exist long enough to debug against), and link each nightly model version to the exact snapshot that produced it. This lets the team keep the "always fresh" retraining cadence while regaining the ability to answer "what changed" when a nightly model regresses.

---

**Situation:** A product manager asks whether adding a feature store is worth the engineering investment for a team currently building their second model, arguing "we can always add it later once we have more models." What would you do and why?

Model answer: Acknowledge the real tradeoff — a feature store has genuine setup cost and isn't justified for a single throwaway model — but point out that the second model is exactly the moment training-serving skew and feature duplication typically start, since now there are two independent code paths that could compute the same concept differently. Recommend at minimum establishing the pattern now (a lightweight Feast setup, or even just a shared, well-documented feature-computation module) so the second model reuses the first model's feature logic instead of reimplementing it, since retrofitting consistency across N models later is far more expensive than establishing it at N=2.
