# Module 04 — Exercises: Reproducibility and Environment Management

These exercises progress from "diagnose drift by hand" to "build a CI-enforced promotion pipeline." Do them in order — each one assumes the tooling/mental model from the previous one. Where a task says "done looks like," that's your self-check before moving on; don't move on until you can tick every box.

Estimated time is for an engineer comfortable with Python and basic Docker, working through the module's `tutorial.md` and `architecture.md` alongside these exercises.

---

## Exercise 1 — Reproduce the drift bug yourself (Beginner, ~30 min)

**Goal:** Feel the reproducibility paradox with your own hands, not just read about it.

**Task:**
1. Create two virtual environments, `env_old` and `env_new`.
2. In `env_old`, install `numpy==1.23.5`. In `env_new`, install `numpy==1.26.4`.
3. In both, run:
   ```python
   import numpy as np
   arr = np.random.RandomState(42).randn(10_000, 512).astype(np.float32)
   print(repr(arr.sum()))
   ```
4. Record the printed value from each environment.
5. Write a two-to-three sentence explanation, in your own words, of *why* two environments running "the same code" can print a different number, and why this is not a bug in NumPy.

**Done looks like:**
- [ ] Two virtualenvs created, each with a single deliberately different pinned NumPy version.
- [ ] Both printed values recorded side by side.
- [ ] A short written explanation that references floating-point summation order or a similar concrete mechanism — not just "versions are different."
- [ ] You can state, without looking it up, which section of `tutorial.md` documents this exact category of drift (library version drift).

---

## Exercise 2 — Freeze, diff, and triage an environment (Beginner→Intermediate, ~45 min)

**Goal:** Practice the manual debugging loop (`pip freeze` → diff → isolate) before you automate it.

**Task:**
1. Create a `requirements-A.txt` and `requirements-B.txt` by hand, each listing 8-10 packages you'd expect in a typical ML service (`fastapi`, `pydantic`, `numpy`, `torch`, `transformers`, `uvicorn`, etc.), with at least 3 deliberately mismatched versions between the two files.
2. Install each into its own virtualenv, then run `pip list --format=json` against each and save the output.
3. Write a small Python script (not using any external diff library) that loads both JSON outputs and prints only the packages whose versions differ, in the format:
   ```
   <package>: A=<version> B=<version>
   ```
4. Rank the 3 mismatches by "how likely is this to cause a silent numerical difference in model output" vs. "how likely is this to cause an outright crash," and justify the ranking in one sentence each.

**Done looks like:**
- [ ] A working diff script that takes two `pip list --format=json` outputs and reports only mismatches (this is a hand-rolled precursor to `tools/env_diff.py` in section 3.1 of `tutorial.md` — don't peek at that code until you've written your own).
- [ ] A ranked, justified list of the 3 mismatches.
- [ ] You can explain why hashing a package is a *different* guarantee than pinning its version, using one of your own mismatched packages as the example.

---

## Exercise 3 — Build a hash-pinned lockfile with `uv` (Intermediate, ~45 min)

**Goal:** Get hands-on with the mid-2026 default tool and understand what a lockfile actually contains.

**Task:**
1. Install `uv` and initialize a new project: `uv init repro-demo && cd repro-demo`.
2. Add these dependencies with exact pins: `fastapi==0.115.0`, `numpy==1.26.4`, `pydantic==2.9.2`.
3. Run `uv sync` and open the generated `uv.lock` file. Find and record:
   - The resolved version of at least one *transitive* dependency you never declared yourself (e.g., something `pydantic` or `fastapi` pulls in).
   - One embedded hash string for any package.
4. Delete `uv.lock`, change one dependency's pin (e.g., bump `numpy` to a different exact version), and regenerate the lock. Diff the old and new lockfiles (you'll need to have saved a copy of the original) and describe what changed beyond just the one package you touched.
5. Run `uv sync --frozen` and, separately, deliberately edit `pyproject.toml` to add an extra dependency *without* re-locking, then run `uv sync --frozen` again and observe what happens.

**Done looks like:**
- [ ] A `uv.lock` file exists with hash-pinned entries for every direct and transitive dependency.
- [ ] You can point to one transitive dependency you didn't ask for by name and its exact pinned version.
- [ ] You've observed and can explain why `uv sync --frozen` fails (or refuses) when `pyproject.toml` and `uv.lock` disagree — this is the mechanism that prevents "it builds, but not from what you think it's building from."
- [ ] You can articulate, in your own words, why `uv.lock` should never be hand-edited.

---

## Exercise 4 — Digest-pin a Dockerfile and prove the difference (Intermediate, ~1 hour)

**Goal:** Understand mutable tags vs. immutable digests at the level where it actually bites you.

**Task:**
1. Write a minimal Dockerfile: `FROM python:3.11-slim`, then `COPY` a tiny `app.py` that just prints the Python version, then `CMD ["python", "app.py"]`.
2. Build it, then run:
   ```bash
   docker inspect --format='{{index .RepoDigests 0}}' python:3.11-slim
   ```
   (pull first if needed) and record the digest.
3. Rewrite the Dockerfile's `FROM` line to pin that exact digest: `FROM python:3.11-slim@sha256:<digest>`.
4. Rebuild, and confirm the image still runs identically.
5. Write two or three sentences explaining, to a skeptical teammate who says "but `3.11-slim` already tells you the version, isn't that pinned enough?", why it is not — and what class of incident digest pinning specifically prevents that version-tag pinning does not.

**Done looks like:**
- [ ] Both the tag-based and digest-pinned Dockerfiles build successfully.
- [ ] You have the actual digest string recorded (not a placeholder).
- [ ] Your explanation names "mutable pointer vs. immutable content address" as the core distinction, not just "digests are more precise."
- [ ] You can state why, as of mid-2026, this is a *hard requirement* for `nvidia/cuda` images specifically (not just a best practice) — tie it back to the `:latest` deprecation described in `tutorial.md` section 3.3.

---

## Exercise 5 — Build the full 5-layer, multi-stage, hash-verified Dockerfile (Intermediate→Advanced, ~1.5-2 hours)

**Goal:** Apply the complete production pattern from `tutorial.md` section 3.3 to a real (small) service, not a toy.

**Task:**
1. Build a minimal FastAPI service (a `/healthz` endpoint plus one real endpoint, e.g., an endpoint that loads a small `scikit-learn` or `numpy`-based toy model and returns a prediction).
2. Produce a hash-pinned `requirements.txt` for it using `uv` or `pip-compile --generate-hashes`.
3. Write a multi-stage Dockerfile following the exact 5-layer ordering from `tutorial.md`: SHA-pinned base image → system deps → lockfile copy only → hash-verified install → application code, with a `builder` stage and a slim `runtime` stage that copies only the resolved `site-packages` across.
4. Add a `HEALTHCHECK` instruction pointing at `/healthz`, and run the container as a non-root user.
5. Prove the cache-efficiency claim: build once, note the build time per layer; touch only `app.py` (not `requirements.txt`); rebuild and confirm the dependency-install layer was reused from cache (check Docker's build output for `CACHED`).
6. Then touch `requirements.txt` (bump one pin) and rebuild; confirm the install layer *now* invalidates and re-runs, while the base-image and system-deps layers remain cached.

**Done looks like:**
- [ ] A working multi-stage Dockerfile matching the 5-layer pattern, with a SHA-pinned base image (real digest, not illustrative).
- [ ] `docker build` output showing `CACHED` for the install layer on an app-code-only change.
- [ ] `docker build` output showing the install layer *re-running* after a lockfile change, while earlier layers stay cached.
- [ ] The final image runs as non-root and passes its own `HEALTHCHECK`.
- [ ] You can explain, without notes, why copying `requirements.txt` before application code is not just a performance trick but also a reproducibility guardrail.

---

## Exercise 6 — Automate the environment-diff gate (Advanced, ~1.5 hours)

**Goal:** Turn Exercise 2's manual diff into the standing CI control described in `tutorial.md` section 3.1 and shown in the system diagram in `architecture.md`.

**Task:**
1. Adapt (don't just copy) the `tools/env_diff.py` pattern from `tutorial.md`: it should snapshot the current environment (`pip list --format=json`), compare it against a committed `environment-baseline.json`, and exit non-zero with a clear diff on mismatch.
2. Extend it beyond the tutorial's version: also compute and compare a single `requirements_lock_hash` (a hash of the entire lockfile file, not per-package hashes) so a caller can check "did anything at all change" in one comparison before drilling into per-package detail.
3. Wire it into a CI workflow (GitHub Actions or equivalent) that runs this check on every pull request, using a `environment-baseline.json` committed to the repo.
4. Deliberately bump one dependency's pin without updating the baseline, open a PR (or simulate the CI run locally), and confirm the check fails with a clear, actionable message.
5. Update the baseline correctly (as part of the same PR that intentionally changes the dependency) and confirm the check now passes.

**Done looks like:**
- [ ] A working `env_diff.py` (or equivalent) that reports both per-package mismatches and a single lockfile-hash comparison.
- [ ] A CI job that fails a PR when the environment silently drifts from the baseline.
- [ ] A demonstrated failing run (dependency bumped, baseline not updated) and a demonstrated passing run (baseline updated deliberately alongside the change).
- [ ] You can explain why this check belongs in CI *before* promotion, not only as an incident-response tool — tie it to the "shift left" principle in section 6 of `tutorial.md`.

---

## Exercise 7 — Build a promotion pipeline that never rebuilds (Advanced, ~2-3 hours)

**Goal:** Implement "promote the artifact forward, never rebuild it" as actual automation, not just an explained principle.

**Task:**
1. Using GitHub Actions (or another CI system of your choice, adapting the pattern), build a pipeline with four jobs: `build-once`, `promote-to-staging`, `automated-staging-gate`, `promote-to-production` — mirroring `tutorial.md` section 3.5.
2. `build-once` must build the Docker image from Exercise 5, push it, and capture its resolved image digest as a job output.
3. Every downstream job must reference that digest (via `needs.build-once.outputs.image_digest`) — add a check (a script or a manual code review checklist item) that would fail the review if a second `docker build` command appears anywhere after `build-once`.
4. `automated-staging-gate` should run at least: a health check against `/healthz`, an integration test against a handful of sample requests asserting response schema, and a lightweight load test asserting a P95 latency threshold (a tool like `locust`, or a simple concurrent `curl`/Python script if you don't want the extra dependency).
5. Gate `promote-to-production` behind a manual approval step (GitHub Environments' required reviewers, or an equivalent manual gate in your CI system).
6. On successful promotion, generate a `promotion.yaml` manifest (matching the shape in `tutorial.md` section 3.5) recording the image digest, which checks passed with what values, and who approved.
7. Deliberately make the load test fail (e.g., set an unreasonably low latency threshold) and confirm the pipeline **stops** — no `promotion.yaml` is written, no production deploy happens, and the failure is visible.

**Done looks like:**
- [ ] A CI pipeline with exactly one `docker build` invocation across the entire pipeline.
- [ ] A passing end-to-end run producing a `promotion.yaml` with all checks recorded as `passed: true`.
- [ ] A deliberately-broken run (failing load test or integration test) that halts before production and before manifest generation — the "stop on failure" principle, verified, not just asserted.
- [ ] You can explain in an interview-style answer why "the same image digest at every stage" is the load-bearing detail of this whole exercise, not an implementation nicety.

---

## Exercise 8 — Capstone: full reproducibility stack + registry aliasing (Advanced, ~3-4 hours, cumulative)

**Goal:** Integrate everything — lockfile, digest-pinned multi-stage build, environment snapshot, CI drift gate, promotion pipeline, and model-registry aliasing — into one coherent, defensible system, at the level expected of a senior MLOps/LLMOps engineer in an interview or a design review.

**Task:**
1. Take the FastAPI + toy-model service from Exercise 5 and log a training/build run to an experiment tracker (MLflow or equivalent), attaching an `environment_snapshot.json` (Python version, framework versions, base image digest, lockfile hash) as a first-class logged artifact — not a comment, an actual attached file, per `tutorial.md` section 3.4.
2. Register the resulting model version in a model registry, and instead of any Staging/Production stage enum, assign it the alias `@challenger`.
3. Wire the full Exercise 7 promotion pipeline around this service, including the Exercise 6 drift gate as one of the automated staging-gate checks.
4. After a successful production promotion, flip the registry alias from `@challenger` to `@champion` for the promoted version (simulate a previous `@champion` existing and being superseded).
5. Write a one-page incident postmortem (fictional, but concrete) describing a scenario where a production accuracy drop is traced back to environment drift, referencing which specific artifact in your stack (the environment snapshot, the drift-gate CI job, the `promotion.yaml`) let you diagnose and rule out other causes within minutes rather than days.
6. In an interview-style self-test, answer out loud (or in writing, then check yourself): "Why are model registry aliasing and environment promotion two separate mechanisms, and what would go wrong if you tried to collapse them into one?"

**Done looks like:**
- [ ] An experiment-tracker run with an attached, inspectable environment snapshot artifact.
- [ ] A registered model version reachable via `models:/<name>@challenger` and, after promotion, via `models:/<name>@champion`.
- [ ] A complete, green, end-to-end promotion pipeline run with a `promotion.yaml` manifest as evidence.
- [ ] A written postmortem that names specific artifacts (not "we checked the logs") as the mechanism that shortened diagnosis time.
- [ ] A clear, correct, unprompted answer to the aliasing-vs-promotion question that matches the reasoning in `tutorial.md` section 3.5's closing subsection.
