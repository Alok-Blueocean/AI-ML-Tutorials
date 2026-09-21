# Module 04 — Quiz: Reproducibility and Environment Management

Instructions: Attempt every question before checking the answer key at the end. Questions marked **(MC)** are multiple choice (single best answer unless stated otherwise); questions marked **(SA)** are short answer — a strong answer is 2-4 sentences that names specific mechanisms, not just vocabulary.

---

**Q1 (MC).** Which equation most accurately reflects why "the code hasn't changed" can still mean production behavior changed?

A. `output = f(code, data)`
B. `output = f(code, data, model_weights)`
C. `output = f(code, data, model_weights, environment)`
D. `output = f(model_weights)` — only the weights matter once training is complete

---

**Q2 (MC).** A teammate says: "Our `requirements.txt` pins every package to an exact version, so we're fully protected against reproducibility problems." What is missing from this claim?

A. Nothing — exact version pins are sufficient on their own.
B. Version pins don't protect against a compromised mirror or corrupted re-upload serving different bytes under the same version string; hash verification is the missing layer.
C. Version pins only work for pure-Python packages, never for compiled wheels.
D. `requirements.txt` files cannot contain exact version pins at all, only ranges.

---

**Q3 (SA).** Explain, with a concrete mechanism, why `FROM python:3.11-slim` in a Dockerfile is not actually a reproducible base image reference, and state the specific fix.

---

**Q4 (MC).** Order the following Dockerfile layers from *least* to *most* frequently changing, per the module's 5-layer build pattern:
1. `pip install --require-hashes -r requirements.txt`
2. `FROM <base>@sha256:...`
3. `COPY ./app ./app`
4. `apt-get install <system deps>`
5. `COPY requirements.txt .`

A. 2, 4, 5, 1, 3
B. 2, 1, 4, 5, 3
C. 4, 2, 1, 5, 3
D. 2, 4, 1, 5, 3

---

**Q5 (SA).** A colleague copies application source code into a Dockerfile *before* copying the lockfile and running `pip install`. Explain both consequences of this ordering: one about build speed, and one about reproducibility risk.

---

**Q6 (MC).** As of mid-2026, which statement about `nvidia/cuda:latest` is correct?

A. It is still the recommended tag for new GPU-serving Dockerfiles because it always has the newest security patches.
B. It has been deprecated and is no longer resolvable on Docker Hub or NGC — pulling it returns a `manifest unknown` error, making explicit digest pinning a hard requirement, not a style preference.
C. It was renamed to `nvidia/cuda:stable` but otherwise behaves identically.
D. It only affects local development and has no bearing on production CUDA base image choices.

---

**Q7 (SA).** You maintain digest-pinned base images for your team's services. What is the specific risk of digest pinning *without* any automated update process, and what is the standard mitigation named in this module?

---

**Q8 (MC).** Which tool combination would you reach for if your environment genuinely requires pinning a *specific compiled CUDA toolkit build* alongside non-Python native libraries not available as PyPI wheels?

A. `uv`, because it is the fastest resolver in 2026.
B. Poetry, because of its plugin ecosystem.
C. `conda-lock`, because Conda's channel-based resolution handles cross-language/native dependencies that pip-compatible resolvers cannot.
D. Plain `pip freeze`, because it captures everything installed regardless of origin.

---

**Q9 (SA).** Explain why `conda env export` (even with `--from-history`) is not a substitute for `conda-lock`, and what guarantee `conda-lock` provides that plain export does not.

---

**Q10 (MC).** In the promotion pipeline described in this module, which of the following best describes the "build-once" principle?

A. Each environment (dev, staging, production) builds its own Docker image from the same Dockerfile, ensuring consistency through identical build instructions.
B. A single Docker image is built exactly once, referenced by its immutable digest, and that same digest is deployed to every subsequent environment — no environment triggers a second build.
C. Only the production environment builds a Docker image; dev and staging run the application directly on the host without containers.
D. The image is rebuilt at each stage but from a cached Docker layer, which is functionally equivalent to building once.

---

**Q11 (SA).** List, in order, the checks that make up the automated staging gate described in this module (dev → staging → production), and explain what should happen if any one of them fails.

---

**Q12 (MC).** Why has MLflow deprecated the Staging/Production/Archived stage enum in favor of aliases like `@champion`/`@challenger`?

A. Aliases are faster to query than the old stage field.
B. The rigid stage enum forced every model version into one of a small fixed set of global states, which doesn't map well onto workflows needing champion/challenger comparisons, multiple simultaneous consumers on different versions, or shadow deployments; aliases are flexible, team-defined tags instead.
C. Stages were removed purely for backward-compatibility reasons unrelated to workflow flexibility.
D. Aliases replace the need for a promotion pipeline entirely.

---

**Q13 (SA).** Explain the difference between a *model registry alias* (e.g., `@champion`) and a *promotion manifest* (`promotion.yaml`). Are they redundant with each other? Why or why not?

---

**Q14 (MC).** Which of the following is the *best* first diagnostic step when investigating a production accuracy drop with no code or data changes, following the debugging methodology in this module?

A. Immediately retrain the model from scratch on the same data to see if the accuracy recovers.
B. Compare the environment (e.g., `pip freeze` or `pip list --format=json`) between the last-known-good environment and the current one to look for a version mismatch.
C. Roll back to a previous model version without further investigation.
D. Assume the incoming data has drifted and re-run data-quality dashboards, since environment issues are statistically rarer.

---

**Q15 (SA).** A senior engineer claims: "Since we set `temperature=0` for our LLM's decoding, our outputs are fully deterministic across deployments." Is this claim correct? Explain the specific mechanism involved and what actually needs to be pinned to approach determinism.

---

---

## Answer Key

**A1: C.** `output = f(code, data, model_weights, environment)`. The module's central framing is that environment is a fourth, often-invisible input — library versions, CUDA/driver versions, Python interpreter version, and system dependencies can all change output even when code, data, and weights are held constant.

**A2: B.** Version pins fix *which release* you asked for, but assume the registry/mirror always serves identical bytes for that version string forever — an assumption that doesn't hold against a compromised mirror, a corrupted re-upload, or general supply-chain tampering. Hash verification (SHA-256 of the actual artifact) closes that gap and is why lockfile tools (`pip-compile --generate-hashes`, Poetry, `uv`, `conda-lock`) embed hashes, not just versions.

**A3:** The tag `3.11-slim` names a minor Python version but is a *mutable pointer* — image maintainers republish the same tag with patched OS packages over time, so two builds of an identical Dockerfile weeks apart can pull different underlying bytes. The fix is digest pinning: `FROM python:3.11-slim@sha256:<digest>`, which references an immutable content address that can only ever resolve to one exact set of bytes.

**A4: A** (2, 4, 5, 1, 3). Base image (fixed/never changes for a given build) → system deps (rarely change) → lockfile copy (changes only when dependencies change) → hash-verified install (same trigger as the layer above) → application code (changes on every commit). Docker's cache invalidates top-down from the first changed layer, so this ordering maximizes cache reuse.

**A5:** Speed consequence: copying application code first busts the pip-install cache layer on *every commit*, even when dependencies haven't changed at all, because Docker's layer cache is keyed on the hash of everything in and before that layer — forcing a full dependency reinstall on every build. Reproducibility consequence: if this is combined with an unpinned or loosely-pinned `requirements.txt`, each forced reinstall becomes a fresh opportunity for the dependency resolver to pick a slightly different version than the previous build, silently reintroducing drift that hash-pinning + correct layer ordering was supposed to eliminate.

**A6: B.** As of mid-2026, `nvidia/cuda:latest` is deprecated and unavailable on both Docker Hub and NGC (`manifest unknown` on pull), making explicit version + digest pinning of GPU base images a hard operational requirement — not merely a best practice, since the convenient floating-tag option has been removed outright for `:latest`.

**A7:** Without an automated update process, digest pinning trades one risk (silent drift) for another: the image silently freezes on old, potentially vulnerable software forever, because nobody is prompted to re-pin. The standard mitigation is a tool like Renovate or Dependabot configured to open an automated pull request whenever a tracked image tag's underlying digest changes, so a human reviews and merges the bump deliberately rather than the build either drifting silently or freezing silently.

**A8: C.** `conda-lock`, because it operates through Conda/mamba's channel-based resolution, which is genuinely capable of pinning a specific compiled CUDA toolkit build and non-Python native libraries — something pip-compatible resolvers like `uv` or Poetry, built around PyPI wheels, cannot replace in that specific scenario.

**A9:** `conda env export` (even `--from-history`) captures a point-in-time resolution of *your current platform*; it is not guaranteed to re-resolve identically on a different platform or at a different point in time, because it does not freeze the complete transitive dependency graph with hashes. `conda-lock` produces a fully pinned, hash-verified, per-platform lockfile (`conda-lock.yml`) that is the actual reproducibility guarantee — treat `conda env export --from-history` as a human-readable declaration of intent, not a substitute for a real lockfile.

**A10: B.** The image is built exactly once, referenced by its immutable digest, and every downstream stage (dev, staging, production) deploys — never rebuilds — that same digest. This is the structural embodiment of "promote the artifact forward, never rebuild it," and it's the detail most real-world implementations violate under time pressure.

**A11:** In order: (1) dependency/schema-mismatch check, (2) health check against `/healthz`, (3) integration test against a set of realistic samples (e.g., ~20) verifying response schema, (4) load test (e.g., 100 requests/sec for 60s) asserting a latency SLO such as P95 below a threshold, (5) evaluation gate (a quality metric above a defined threshold on held-out/canary data). If *any* check fails, the pipeline must stop — it does not proceed to human approval or production, and must not be silently re-run or bypassed under pressure; a failure halts forward progress entirely rather than soft-failing with a warning.

**A12: B.** The rigid stage enum forces every model version into one of a small fixed global set of states, which doesn't accommodate real workflows like champion/challenger comparison, multiple production consumers pinned to different versions simultaneously, or shadow deployments. Aliases are flexible, arbitrary, team-defined tags that map more naturally onto these workflows, which is why MLflow has deprecated stages (since MLflow 2.9) in favor of aliases.

**A13:** A model registry alias (`@champion`/`@challenger`) is a movable pointer indicating which *model version* is currently serving in a given role — it answers "what is live right now." A promotion manifest (`promotion.yaml`) is an audit record of an *environment promotion event* — it answers "what artifact moved from which stage to which, what checks passed with what values, and who approved it, and when." They are complementary, not redundant: one tracks current serving state for the model registry, the other tracks the historical evidence trail for the deployment pipeline. Collapsing them would conflate "what's currently live" with "the audit history of how it got there," losing the ability to answer either question cleanly.

**A14: B.** Compare environments (e.g., diffing `pip freeze` / `pip list --format=json` output between the known-good and the currently-misbehaving environment) to look for an unexpected version mismatch. This follows the module's methodical debugging sequence: recognize the symptom, compare environments, isolate the mismatch, confirm root cause with a minimal repro, then fix and lock in place — jumping straight to retraining, rollback, or blaming data drift skips the diagnostic step that would tell you whether environment drift, not the model or the data, is the actual cause.

**A15:** No, the claim is not correct. Greedy decoding (`temperature=0`) removes *sampling* randomness, but GPU floating-point reductions are non-associative across parallel threads — the order in which partial sums are combined can differ across kernel versions, CUDA/driver versions, or hardware, producing different results even under fully deterministic (greedy) decoding. Approaching real determinism requires pinning the entire stack the generation depends on — CUDA toolkit, driver, attention-kernel implementation (e.g., FlashAttention version), the serving framework, and the model/quantization build — not just the sampling parameters.
