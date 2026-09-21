# Module 03 Cheat Sheet — Versioning, Registries, and Rollback

One-page reference. See `tutorial.md` for full explanations.

---

## The One Rule That Matters Most

```
prediction = f( model_version, prompt_version, dataset_version, input )
```
**Always version and log the full deployment TRIPLE — never the model alone.**

---

## SemVer Cheat Table (MAJOR.MINOR.PATCH)

| Artifact | MAJOR | MINOR | PATCH |
|---|---|---|---|
| Model | New architecture / breaks I/O contract | Retrained, meaningfully better, contract unchanged | Hyperparam tweak, no material shift |
| Prompt | Output schema / task contract changes | Wording changed to improve quality, contract same | Typo/whitespace only |
| Dataset | Schema change (cols/labels) | New records/slice added | Dedup, metadata fix only |

---

## MLflow Registry: Current (2026) API — Aliases, NOT Stages

```python
# DEPRECATED since MLflow 2.9 — do NOT use in new code:
# client.transition_model_version_stage(name, version, stage="Production")

# CURRENT pattern:
client.set_registered_model_alias(name, "champion", version)     # promote
client.set_registered_model_alias(name, "challenger", version)   # candidate
client.set_model_version_tag(name, version, "semver", "2.1.0")
client.get_model_version_by_alias(name, "champion")              # read current prod

# Serving code — NEVER hardcode a version:
model = mlflow.pyfunc.load_model(f"models:/{name}@champion")
```

### MLflow 3 GenAI Prompt Registry

```python
p = mlflow.genai.register_prompt(name="reviewer-prompt", template="...", commit_message="...")
mlflow.genai.set_prompt_alias(name="reviewer-prompt", alias="production", version=p.version)
prompt = mlflow.genai.load_prompt("prompts:/reviewer-prompt@production")
```

---

## Promotion Gate — Minimum Three (Four for LLMs) Checks

| Check | Example | Blocks |
|---|---|---|
| Absolute floor | `accuracy >= 0.88` | Simply-not-good-enough models |
| Relative regression guard | `(challenger - champion)/champion >= -0.01` | New model worse than current prod |
| Latency/cost gate | `p95_latency_ms < 2000` | SLA / budget blowout |
| Safety gate (LLM) | judge/red-team score ≥ threshold | Safety regressions even if accuracy improves |

**Rule:** Promotion is code-driven, never manual. High-stakes/regulated domains = automated floor **+** required human sign-off (never human-only).

---

## `release.yaml` Skeleton

```yaml
release:
  name: fraud-detection-service
  model:   { name: fraud-detector,   version: "2.1.0" }
  prompt:  { name: reviewer-prompt,  version: "1.3.2" }
  dataset: { name: claims-golden,    version: "5.0.0" }
compatibility:
  min_dataset_major: 5
  required_prompt_major: 1
rollout:
  stage: staging          # staging | canary | production
  approval: pending        # pending | approved
  canary_percent: 0
  approved_by: null
```

---

## Rollback Decision Rules

1. Rollback target = **last known good (LKG)** — the last release tagged `GOOD`, *not* simply "the previous release" (the previous one might itself be bad).
2. Restore the **full tuple atomically** (model + prompt + dataset) — never model-only, because regressions can come from cross-component interaction.
3. Every rollback writes an audit event: **reason, restored versions (all 3), timestamp, operator**.
4. Speed with control: rollback must be fast, but only to a *recorded* LKG tuple — never to an arbitrary guessed version.

```python
lkg = get_last_known_good(before_release_id=bad_release_id)   # query status='GOOD', most recent
restore_full_tuple(target=lkg, reason="...", operator="on-call:jdoe")
# restore_full_tuple() repoints: model alias -> prompt alias -> dataset pointer -> writes audit event
```

---

## Canary Ramp Pattern

```
5% ──(healthy?)──► 25% ──(healthy?)──► 100%
 │                   │                   │
 └── breach at ANY point ──► auto-rollback to 0% (full tuple restore)
```
Use for every new release tuple, regardless of offline eval confidence — offline sets can't simulate real production input distributions.

---

## Registry Choice — Quick Decision Rules

| If... | Choose |
|---|---|
| Need prompts + models versioned together (LLM product) | **MLflow 3** (native GenAI Prompt Registry) |
| Already deep in W&B/Comet for tracking, no strong GenAI need | That vendor's registry (less integration friction) |
| No vendor lock-in wanted, open-source default | **MLflow** (self-hosted or Databricks-managed w/ Unity Catalog) |
| Scale/compliance exceeds any off-the-shelf tool (Uber-class) | **Custom** — budget real eng time for lineage/audit/RBAC |

Decision factors: release cadence, governance load, centralized vs. decentralized deploy ownership, lineage depth needed, existing tooling gravity.

---

## Rollback Pattern — Quick Decision Rules

| Situation | Action |
|---|---|
| Release changed >1 leg of the triple | **Mandatory full-tuple rollback**; root-cause after, not during |
| Release changed 1 leg, but concurrent unrelated changes exist (feature flag, infra, data pipeline) | Still prefer full-tuple rollback (cheap insurance) |
| Release changed 1 leg, no concurrent changes | Single-leg rollback acceptable — but still log full triple |
| About to ship a new release (pre-incident) | Canary ramp, never straight-to-100% |

---

## Common Mistakes (Quick-Scan List)

- Versioning only the model, not the triple
- Using deprecated `transition_model_version_stage` in new code
- Manual "looks good to me" promotion
- Rolling back the model only during an incident
- No canary step — straight to 100%
- Decorative version bumps with no evaluation behind them
- No audit trail on rollback (missing reason/timestamp/operator)
- Confusing "previous version" with "last known good"
- Registry used as pure binary storage (no metrics/lineage attached)
- No `compatibility` enforcement between legs

---

## Key Terms, One Line Each

- **Deployment triple** — (model, prompt, dataset) version combo live in prod; the unit of reproducibility.
- **Alias** — mutable named pointer to a model/prompt version (`@champion`); replaces deprecated Stages.
- **LKG** — last release tuple that was validated in prod and triggered no rollback.
- **Lineage** — the recorded graph from deployed release back to run/commit/data that produced it.
- **Canary** — small-percentage traffic exposure before full rollout, to bound blast radius.
