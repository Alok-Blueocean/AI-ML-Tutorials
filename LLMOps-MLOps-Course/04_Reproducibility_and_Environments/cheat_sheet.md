# Module 04 Cheat Sheet — Reproducibility and Environment Management

One-page reference. For theory/intuition, see `tutorial.md`; for full diagrams, see `architecture.md`.

## The core equation

```
output = f(code, data, model_weights, environment)   ← environment is the silent 4th input
```

## Drift categories (know these cold for interviews)

| Drift type | Layer | Example symptom |
|---|---|---|
| Library version drift | Application dependency | NumPy reduction order changes a metric |
| CUDA/driver drift | Hardware/GPU stack | `CUDA error: no kernel image available`, silent precision change |
| Python interpreter drift | Language runtime | C-extension ABI mismatch across 3.10→3.12 |
| System dependency drift | OS layer | Different BLAS backend (OpenBLAS vs MKL) changes float results |

## Pin + Hash — the two-layer guarantee

- **Pin** (`torch==2.4.1`) → fixes *which release*. Does NOT protect against a compromised/corrupted re-upload under the same version string.
- **Hash** (SHA-256 of the artifact) → verifies *exact bytes*. This is the supply-chain security layer.
- Rule: **pinning alone is never enough for anything that reaches CI or production.**

## Lockfile commands, by tool

```bash
# pip-tools
pip-compile --generate-hashes --output-file=requirements.txt requirements.in
pip install --require-hashes -r requirements.txt

# Poetry
poetry lock
poetry install --sync

# uv (mid-2026 default — fast, single tool, native lockfile)
uv add torch==2.4.1 transformers==4.44.2
uv sync --frozen        # fails if pyproject.toml and uv.lock disagree

# conda-lock (needed when you must pin native/CUDA-toolkit builds)
conda-lock lock --file environment.yml --platform linux-64 --platform osx-arm64
conda-lock install --name llm-training conda-lock.yml
```

## Lockfile tool decision rule (one-liner each)

| Need | Use |
|---|---|
| Specific CUDA toolkit build / non-Python native libs | `conda-lock` (hard constraint — pip-based tools can't do this) |
| New project, 2026, no existing tooling investment | `uv` (default recommendation) |
| Must publish to PyPI, need dependency-group tooling | Poetry |
| Minimal change from existing plain-pip workflow | `pip-tools` (`--generate-hashes`) |

## Container image pinning

```dockerfile
# WRONG — mutable pointer, republished on every patch
FROM python:3.11-slim

# RIGHT — immutable content address
FROM python:3.11-slim@sha256:<digest>
```

- `nvidia/cuda:latest` is **deprecated/unresolvable** as of mid-2026 → digest pinning of GPU base images is a hard requirement, not a preference.
- Get the digest: `docker inspect --format='{{index .RepoDigests 0}}' <image>:<tag>`
- Digest pinning without an update process = "silently frozen on a stale, vulnerable image." Fix: Renovate/Dependabot PRs on digest change (review, don't auto-merge).

## The 5-layer Docker build pattern (order = least → most volatile)

```
1. FROM <base>@sha256:...              (fixed)
2. apt-get install <system deps>       (rarely changes)
3. COPY requirements.txt .             (lockfile only, NOT source)
4. pip install --require-hashes ...    (same trigger as #3)
5. COPY ./app ./app                    (changes every commit)
```
Docker cache invalidates top-down from the first changed layer → this ordering is simultaneously the most reproducible *and* the fastest-building pattern. Use multi-stage builds (`builder` → `runtime`) to ship without compilers/build tools in the final image.

## Environment snapshot (attach to every run/model version)

```yaml
run_id: "run-2026-07-14-8841"
python_version: "3.11.9"
pytorch_version: "2.4.1+cu124"
cuda_version: "12.4"
base_image: "myregistry/llm-service@sha256:9d1a3f...b02c"
requirements_lock_hash: "sha256:7c4e21...af90"   # hash of the WHOLE lockfile
gpu: "NVIDIA A100-SXM4-80GB"
git_commit: "a1b2c3d"
```
Rule: environment metadata is first-class experiment data — log it like metrics/params (`mlflow.log_dict(...)`), not as an afterthought comment.

## Promotion: build once, promote forward

```
DEV ──build once (image@digest)──▶ STAGING ──same digest──▶ PRODUCTION
```

**Never a second `docker build` after `build-once`.** Every downstream CI job references `needs.build-once.outputs.image_digest`.

### Automated staging gate (in order — ANY failure stops everything)

```
1. dependency / schema-mismatch check
2. health check          → GET /healthz
3. integration test      → ~20 realistic samples, verify response schema
4. load test             → e.g. 100 rps / 60s, assert P95 < threshold (e.g. 2000ms)
5. evaluation gate        → quality metric ≥ threshold
   ALL PASS → write promotion.yaml, request human sign-off
   ANY FAIL → STOP. No production deploy. No silent re-run.
```

### `promotion.yaml` skeleton

```yaml
artifact:
  name: "llm-support-classifier"
  version: "1.4.2"
  image_digest: "sha256:9d1a3f0c...b02cff"
promotion: {from_stage: "staging", to_stage: "production", timestamp: "..."}
checks:
  evaluation_score: {metric: "f1_macro", value: 0.913, threshold: 0.90, passed: true}
  smoke_test:       {samples_tested: 20, schema_valid: true, passed: true}
  load_test:        {rps: 100, duration_seconds: 60, p95_latency_ms: 1420, threshold_ms: 2000, passed: true}
  approval:         {approved_by: "...", approved_at: "...", passed: true}
result: "PROMOTED"
```

## Model registry: aliases, not stages (MLflow ≥ 2.9)

```python
client.set_registered_model_alias(name="llm-support-classifier", alias="champion", version="14")
model_uri = "models:/llm-support-classifier@champion"   # resolve by alias, never hardcode version
```
- Stage enum (Staging/Production/Archived) is **deprecated** — don't build new tooling around it.
- Alias = "what's live now" (registry concern). `promotion.yaml` = "audit trail of how it got promoted" (pipeline concern). Complementary, not the same mechanism.

## Debugging drift — the loop

```
1. Recognize symptom (metric drop, no code/data change)
2. Compare environments  → pip list --format=json, diff
3. Isolate the mismatch  → e.g. numpy==1.23.5 vs 1.26.4
4. Confirm root cause    → minimal repro, re-run with old pin, metric recovers?
5. Lock the fix in place → pin + hash it, and add a CI drift gate so it can't recur silently
```
The loop isn't closed at "found it" — it's closed at "added a standing CI check that would have caught this automatically."

## GPU determinism gotcha

`temperature=0` (greedy decoding) ≠ bit-for-bit determinism. GPU float reductions are non-associative across parallel threads — different kernel/driver/CUDA versions can still produce different outputs. Pin the whole stack (CUDA, driver, attention kernels, serving framework), not just decoding params.

## Red flags (common mistakes, condensed)

- Version-pinned but not hash-pinned lockfile.
- `FROM python:3.x-slim` treated as "pinned."
- Any `:latest` GPU base image.
- `conda env export` (non-`--from-history` or otherwise) used as a lockfile substitute.
- App code copied before lockfile in Dockerfile.
- Rebuilding the image at each promotion stage instead of promoting one digest.
- Manually re-running checks / skipping gates "just this once" under incident pressure.
- Metrics logged but no environment snapshot attached.
- Digests pinned once, never bumped (no Renovate/Dependabot).
- New tooling built around MLflow's deprecated stage enum.
