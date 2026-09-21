# Module 04 — Architecture Deep Dive: Reproducibility and Environment Management

This companion document expands the architecture views only sketched in `tutorial.md`: a full system-level diagram of where reproducibility controls sit across the ML/LLM platform, a sequence diagram of a drift-induced incident from symptom to fix, a sequence diagram of a full dev→staging→production promotion run, and a decision tree for choosing among the lockfile/container-pinning tools covered in this module.

---

## 1. System-Level Architecture: Reproducibility Controls Across the Platform

This diagram shows every point in a typical ML/LLM platform where an environment-reproducibility control is required, from a developer's laptop through to the production serving fleet.

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                              DEVELOPER WORKSTATION                                   │
│                                                                                       │
│   pyproject.toml / requirements.in   (loose, human-edited intent)                    │
│              │                                                                       │
│              ▼  uv lock / pip-compile / poetry lock / conda-lock                     │
│   uv.lock / requirements.txt(+hashes) / poetry.lock / conda-lock.yml                  │
│              │                                                                       │
│              ▼  committed to git — environment definitions are code                  │
└──────────────┼───────────────────────────────────────────────────────────────────────┘
               │
               ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                         CI: BUILD-ONCE PIPELINE (GitHub Actions)                     │
│                                                                                       │
│  ┌───────────────┐   ┌────────────────────┐   ┌──────────────────────────────────┐  │
│  │ Checkout repo  │──▶│ docker build        │──▶│ Push image to registry            │  │
│  │ at pinned SHA  │   │ FROM <base>@sha256  │   │ registry.internal/svc:<git-sha>   │  │
│  └───────────────┘   │ COPY lockfile        │   │ Resolve + record IMAGE DIGEST     │  │
│                       │ pip install          │   │ (this digest is THE artifact —    │  │
│                       │   --require-hashes   │   │  every later stage references it) │  │
│                       │ COPY app code        │   └──────────────────────────────────┘  │
│                       └────────────────────┘                                          │
│                                                                                       │
│  Also produced here: environment-snapshot.json, SBOM, CVE scan report (Trivy/Grype)   │
└──────────────┼───────────────────────────────────────────────────────────────────────┘
               │  (single immutable artifact, referenced by digest from here on)
               ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                DEV ENVIRONMENT                                       │
│   Deploy image@digest. Run unit tests + data/schema checks.                          │
└──────────────┼───────────────────────────────────────────────────────────────────────┘
               │  promote SAME digest (no rebuild)
               ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                              STAGING ENVIRONMENT                                    │
│                                                                                       │
│   Automated Staging Gate:                                                            │
│    1. Dependency / schema-mismatch check                                             │
│    2. Health check endpoint  (/healthz)                                              │
│    3. Integration test — ~20 realistic samples, schema validation                    │
│    4. Load test — 100 rps for 60s, assert P95 latency < 2000ms                       │
│    5. Evaluation gate — quality metric above threshold                               │
│                                                                                       │
│   ALL PASS ──▶ write promotion.yaml, request human sign-off                          │
│   ANY FAIL ──▶ STOP. No further promotion. Alert owning team.                        │
└──────────────┼───────────────────────────────────────────────────────────────────────┘
               │  promote SAME digest (no rebuild) + signed-off promotion.yaml
               ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                           PRODUCTION ENVIRONMENT (fleet)                             │
│                                                                                       │
│   Rolling deployment of image@digest across the serving fleet.                        │
│   Every replica runs bit-for-bit the same base image + lockfile-resolved deps.        │
│                                                                                       │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐        ┌────────────┐              │
│   │ replica 1   │  │ replica 2   │  │ replica 3   │  ...   │ replica N   │              │
│   │ image@sha256│  │ image@sha256│  │ image@sha256│        │ image@sha256│              │
│   └────────────┘  └────────────┘  └────────────┘        └────────────┘              │
│                                                                                       │
│   Continuous: environment-diff monitor compares live replicas' `pip list --format=   │
│   json` against the environment-snapshot.json recorded at build time — any diff       │
│   (e.g., a host-level driver update) raises an alert BEFORE a metric regresses.       │
└───────────────────────────────────────────────────────────────────────────────────┘
               │
               ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                        MODEL REGISTRY (e.g., MLflow)                                 │
│   Version 14 → alias @champion       (serving code resolves "models:/name@champion") │
│   Version 15 → alias @challenger     (shadow-tested before alias flip)               │
│   Environment snapshot attached as an artifact of the run that produced each version. │
└───────────────────────────────────────────────────────────────────────────────────┘
```

**Reading this diagram.** Notice the invariant running top to bottom: exactly **one build** produces exactly **one digest**, and every subsequent box (dev, staging, production) either checks that digest or deploys it — none of them rebuild it. This is the structural embodiment of "promote the artifact forward, don't rebuild," and it's the single detail most implementations get wrong under time pressure.

---

## 2. Sequence Diagram: Drift-Induced Incident, From Symptom to Fix

This ASCII sequence diagram walks through the debugging flow from `tutorial.md` section 3.1 as an actual timed interaction between the people and systems involved.

```
 On-call Eng      Monitoring       Dev Env         Staging/Prod      CI Pipeline      Git Repo
      │               │               │                 │                │              │
      │  alert: model  │               │                 │                │              │
      │◀───────────────│               │                 │                │              │
      │  accuracy drop │               │                 │                │              │
      │                │               │                 │                │              │
      │  Step 1: Recognize the symptom (not yet localized to a cause)      │              │
      │                │               │                 │                │              │
      │  pip freeze                    │                 │                │              │
      │────────────────────────────────▶                 │                │              │
      │        dev_env.txt returned    │                 │                │              │
      │◀────────────────────────────────                 │                │              │
      │                │               │                 │                │              │
      │  pip freeze (same command, run against prod)      │                │              │
      │─────────────────────────────────────────────────▶ │                │              │
      │        prod_env.txt returned                       │                │              │
      │◀───────────────────────────────────────────────── │                │              │
      │                │               │                 │                │              │
      │  Step 2: Compare environments                                       │              │
      │  diff dev_env.txt prod_env.txt                                      │              │
      │  (local computation, no system call)                                │              │
      │                │               │                 │                │              │
      │  Step 3: Find the mismatch                                          │              │
      │  → "numpy==1.23.5" (dev) vs "numpy==1.26.4" (prod)                  │              │
      │                │               │                 │                │              │
      │  Step 4: Identify root cause — minimal repro                        │              │
      │  run identical eval script under both numpy versions                │              │
      │────────────────────────────────▶                 │                │              │
      │                │               │  eval score      │                │              │
      │◀────────────────────────────────  differs → CONFIRMED               │              │
      │                │               │                 │                │              │
      │  Step 5: Prevent recurrence                                         │              │
      │  pin numpy==1.23.5 in requirements.in                                │              │
      │───────────────────────────────────────────────────────────────────────────────────▶│
      │                │               │                 │                │  commit pushed│
      │                │               │                 │                │◀──────────────│
      │                │               │                 │                │  triggers CI  │
      │                │               │                 │                │               │
      │                │               │                 │  rebuild ONCE, re-run full      │
      │                │               │                 │  promotion pipeline (section 3) │
      │                │               │                 │◀─────────────────────────────── │
      │                │               │                 │                │               │
      │  Step 6: add environment-diff CI gate so this class of drift        │              │
      │  is caught automatically before the NEXT promotion, not after       │              │
      │  the next metric regression                                        │              │
      │────────────────────────────────────────────────────────────────────▶│              │
      │                │               │                 │                │  gate added   │
      ▼                ▼               ▼                 ▼                ▼               ▼
```

**Key structural point:** the loop closes not at "found the bug" but at "added an automated gate that would have caught this before it ever reached production." A drift incident that is fixed once but not converted into a standing CI check will recur — possibly with a different package next time.

---

## 3. Sequence Diagram: Full Dev → Staging → Production Promotion Run

```
 Developer     Git/CI          Dev Env        Staging Env       Automated Gate     Human Approver    Prod Env       Registry
     │            │                │               │                  │                 │              │             │
     │ push commit│                │               │                  │                 │              │             │
     │───────────▶│                │               │                  │                 │              │             │
     │            │ docker build (ONCE)             │                  │                 │              │             │
     │            │──────┐         │               │                  │                 │              │             │
     │            │◀─────┘         │               │                  │                 │              │             │
     │            │ push image@sha256:abc...        │                  │                 │              │             │
     │            │───────────────────────────────────────────────────────────────────────────────────────────────▶  │
     │            │                │               │                  │                 │              │             │
     │            │ deploy image@sha256:abc → dev   │                  │                 │              │             │
     │            │───────────────▶│               │                  │                 │              │             │
     │            │                │ unit tests +   │                  │                 │              │             │
     │            │                │ data checks    │                  │                 │              │             │
     │            │                │──────┐         │                  │                 │              │             │
     │            │                │◀─────┘ PASS    │                  │                 │              │             │
     │            │◀───────────────│               │                  │                 │              │             │
     │            │                │               │                  │                 │              │             │
     │            │ deploy SAME image@sha256:abc → staging (no rebuild)│                  │                 │              │             │
     │            │───────────────────────────────▶│                  │                 │              │             │
     │            │                │               │ trigger gate      │                 │              │             │
     │            │                │               │─────────────────▶│                 │              │             │
     │            │                │               │                  │ 1. dep/schema    │              │             │
     │            │                │               │                  │    check         │              │             │
     │            │                │               │                  │ 2. health check   │              │             │
     │            │                │               │                  │ 3. integration    │              │             │
     │            │                │               │                  │    test (20       │              │             │
     │            │                │               │                  │    samples)       │              │             │
     │            │                │               │                  │ 4. load test      │              │             │
     │            │                │               │                  │    100rps/60s,     │              │             │
     │            │                │               │                  │    P95<2000ms      │              │             │
     │            │                │               │                  │ 5. eval gate       │              │             │
     │            │                │               │                  │──────┐            │              │             │
     │            │                │               │                  │◀─────┘ ALL PASS   │              │             │
     │            │                │               │                  │ write promotion.yaml│              │             │
     │            │                │               │◀─────────────────│                 │              │             │
     │            │                │               │  request sign-off                    │              │             │
     │            │                │               │──────────────────────────────────▶  │              │             │
     │            │                │               │                  │                 │ review evidence│             │
     │            │                │               │                  │                 │ in promotion.yaml│           │
     │            │                │               │                  │                 │──────┐         │             │
     │            │                │               │                  │                 │◀─────┘ APPROVE │             │
     │            │                │               │◀─────────────────────────────────── │              │             │
     │            │  deploy SAME image@sha256:abc → production (no rebuild)               │              │             │
     │            │───────────────────────────────────────────────────────────────────────────────────▶  │             │
     │            │                │               │                  │                 │  rolling deploy│            │
     │            │                │               │                  │                 │  across fleet  │            │
     │            │                │               │                  │                 │──────┐         │             │
     │            │                │               │                  │                 │◀─────┘ LIVE    │             │
     │            │  update registry alias: model version N → @champion                              │             │
     │            │─────────────────────────────────────────────────────────────────────────────────────────────▶ │
     ▼            ▼                ▼               ▼                  ▼                 ▼              ▼             ▼
```

**Failure branch (not drawn above for clarity — described here):** if any step under "Automated Gate" fails, the sequence terminates immediately at that point. No message flows to "Human Approver," no deploy message flows to "Prod Env," and the pipeline instead emits an alert to the owning team with the specific failed check attached (this is the "stop on failure" principle — a failure must halt forward progress, not merely log a warning while continuing).

---

## 4. Decision Tree: Choosing a Lockfile Tool

```
                         ┌───────────────────────────────────────┐
                         │ Does your environment need a specific   │
                         │ CUDA toolkit BUILD, or non-Python native│
                         │ libraries not available as PyPI wheels? │
                         └───────────────────┬─────────────────────┘
                                 YES          │          NO
                     ┌───────────────────────┘          └───────────────────────┐
                     ▼                                                          ▼
        ┌─────────────────────────┐                          ┌───────────────────────────────────┐
        │       conda-lock          │                          │ Is this a NEW project starting     │
        │  (Conda/mamba channels,   │                          │ fresh in 2026, with no existing     │
        │   cross-platform lock)    │                          │ Poetry/pip-tools investment?        │
        └─────────────────────────┘                          └───────────────┬─────────────────────┘
                                                                 YES          │          NO
                                                     ┌───────────────────────┘          └─────────────────┐
                                                     ▼                                                     ▼
                                        ┌─────────────────────────┐                    ┌──────────────────────────────┐
                                        │            uv              │                    │ Do you need to PUBLISH this   │
                                        │  fastest resolve/install,  │                    │ as a package to PyPI, with rich │
                                        │  single tool, native lock, │                    │ dependency-group tooling?       │
                                        │  strong default for 2026   │                    └───────────────┬───────────────┘
                                        └─────────────────────────┘                       YES              │           NO
                                                                                ┌────────────────────────┘             └───────────────────┐
                                                                                ▼                                                          ▼
                                                                   ┌─────────────────────────┐                        ┌──────────────────────────────┐
                                                                   │         Poetry             │                        │ Is switching off pip a bigger  │
                                                                   │ (publishing workflow,      │                        │ migration cost right now than  │
                                                                   │  dependency groups, plugin │                        │ the benefit of migrating to uv?│
                                                                   │  ecosystem already in use) │                        └───────────────┬───────────────┘
                                                                   └─────────────────────────┘                       YES               │            NO
                                                                                                          ┌────────────────────────────┘             └──────────────┐
                                                                                                          ▼                                                        ▼
                                                                                             ┌─────────────────────────┐                          ┌─────────────────────────┐
                                                                                             │       pip-tools            │                          │            uv              │
                                                                                             │ (minimal change: keep pip, │                          │ (migrate — the speed and   │
                                                                                             │  add --generate-hashes)   │                          │  single-tool win outweighs │
                                                                                             │                            │                          │  a normally-cheap migration)│
                                                                                             └─────────────────────────┘                          └─────────────────────────┘
```

**How to read this tree in an interview setting:** the first fork is the only one that's a hard technical constraint (Conda channels genuinely do things `pip`-based tools cannot — pinning a specific compiled CUDA toolkit build being the clearest example). Every fork after that is a judgment call about existing team investment and priorities, which is exactly why a senior engineer's answer should name the *criteria* (publishing needs, migration cost vs. speed benefit, existing tooling investment) rather than asserting one tool is universally "best."

---

## 5. Decision Tree: Container Base Image Pinning Strategy

```
                    ┌───────────────────────────────────────────┐
                    │   Will this image ever run in CI, staging,  │
                    │   or production (i.e., anywhere other than  │
                    │   a single disposable local build)?         │
                    └────────────────────┬──────────────────────┘
                          NO              │              YES
              ┌──────────────────────────┘              └──────────────────────────┐
              ▼                                                                     ▼
  ┌─────────────────────────┐                                     ┌───────────────────────────────────┐
  │ Tag-based FROM is fine    │                                     │ Does the image need GPU/CUDA        │
  │ for pure local scratch     │                                     │ support?                             │
  │ work (still avoid `:latest`│                                     └───────────────┬───────────────────┘
  │ out of habit)              │                                       YES           │            NO
  └─────────────────────────┘                                ┌────────────────────┘             └───────────────────┐
                                                               ▼                                                     ▼
                                                  ┌─────────────────────────────┐                     ┌─────────────────────────────┐
                                                  │ Pin nvidia/cuda (or vendor    │                     │ Pin python:<version>-slim     │
                                                  │ ML base image) by exact        │                     │ (or distroless equivalent)     │
                                                  │ version + @sha256 digest.      │                     │ by exact version + @sha256     │
                                                  │ `:latest` is deprecated/        │                     │ digest.                         │
                                                  │ unavailable for nvidia/cuda —   │                     └───────────────┬───────────────┘
                                                  │ this is a hard requirement,     │                                     │
                                                  │ not a preference.               │                                     ▼
                                                  └───────────────┬─────────────────┘                    ┌─────────────────────────────┐
                                                                  │                                        │ Wire Renovate/Dependabot to   │
                                                                  ▼                                        │ open a PR whenever the tracked │
                                                     ┌─────────────────────────────┐                     │ digest changes — review, don't │
                                                     │ Verify PyTorch/framework       │                     │ auto-merge, security-relevant  │
                                                     │ build matches the pinned CUDA  │                     │ base-image bumps.               │
                                                     │ version (e.g., torch+cu124      │                     └─────────────────────────────┘
                                                     │ wheel against a cu124 base)      │
                                                     └─────────────────────────────┘
```

---

## 6. Summary of Diagrams in This Document

| Diagram | Purpose |
|---|---|
| System-level architecture (Section 1) | Shows every point in the platform where a reproducibility control (lockfile, digest pin, gate, snapshot) is enforced, from laptop to production fleet, and the single-build/many-promotions invariant. |
| Sequence: drift incident (Section 2) | Shows the human/system interaction timeline of debugging a drift-induced incident, ending in a standing CI gate rather than a one-off fix. |
| Sequence: full promotion run (Section 3) | Shows exactly one build, three deploys of the same digest, an automated gate, and a human approval step gated on evidence, plus the failure-halts-everything branch. |
| Decision tree: lockfile tool (Section 4) | A criteria-driven walkthrough for choosing between `pip-tools`, Poetry, `uv`, and `conda-lock`. |
| Decision tree: base image pinning (Section 5) | A criteria-driven walkthrough for tag-vs-digest pinning and CPU-vs-GPU base image selection. |

For narrative theory, code samples, comparison tables, case studies, and interview Q&A, see `tutorial.md` in this same folder. For source citations, see `references.md`, `github.md`, `videos.md`, and `books.md`.
