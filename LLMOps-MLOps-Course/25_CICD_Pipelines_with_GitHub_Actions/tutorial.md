# CI/CD Pipelines with GitHub Actions

## What This Topic Is

CI/CD stands for **Continuous Integration** and **Continuous Delivery/Deployment**. It's the practice of automatically building, testing, and (optionally) deploying your code every time you push a change, instead of doing those steps by hand.

**GitHub Actions** is GitHub's built-in automation tool for doing exactly this. You describe a pipeline as a simple YAML file, put it in your repository, and GitHub runs it automatically whenever the events you specify happen (a push, a pull request, a schedule, etc.).

For an MLOps/LLMOps project, this pipeline is the glue that connects everything you've learned so far: it checks out your version-controlled code and data references (Modules 2-3), rebuilds your reproducible environment, and runs your quality gates (Module 4) automatically, on every change — so nothing broken or unreviewed slips into your main branch.

## Why It Matters

In ML and LLM projects, "it works on my machine" is a common trap: notebooks that run out of order, dependency drift, a model that quietly regresses after a data change. CI/CD closes that gap by making verification automatic and consistent:

- **Catches problems early.** A failing test or a broken environment shows up in minutes, on the commit that caused it — not weeks later in production.
- **Enforces reproducibility.** The pipeline always builds from a clean environment, so you find out immediately if something only worked "by accident" on your laptop.
- **Removes manual toil.** No one has to remember to run tests or lint the code before merging — the pipeline does it every time.
- **Builds trust in the main branch.** If your gates are solid, `main` is always in a state you could ship or deploy from.
- **Scales with the team.** As more people contribute models, prompts, and pipelines, automated checks are what keep quality consistent without a human bottleneck reviewing everything manually.

## Main Concepts in Plain Terms

**Workflow**
A workflow is one automated pipeline, defined in a YAML file stored at `.github/workflows/` in your repo. You can have several workflows (e.g., one for tests, one for deployment).

**Trigger (`on`)**
The event that starts the workflow — most commonly a `push` or a `pull_request`. You can also trigger on a schedule (like a nightly retrain-check job) or manually.

**Job**
A workflow is made of one or more jobs. Each job runs on a fresh virtual machine ("runner") — think of it as a clean computer that spins up, does its work, and disappears.

**Step**
Each job is a sequence of steps: check out the code, install dependencies, run tests, and so on. Steps either run a shell command or use a pre-built **Action** (a reusable step someone else packaged, like `actions/checkout` or `actions/setup-python`).

**Runner**
The machine that executes your jobs. GitHub provides free hosted runners (Linux, Windows, macOS); you can also use your own ("self-hosted") if you need special hardware, like a GPU.

**CI gate**
A "gate" is simply a check that must pass before code can merge — tests, linting, type-checking, a model-evaluation threshold. In GitHub, this ties into **branch protection rules**: you can require a workflow to succeed before a pull request is allowed to merge. This is what makes the pipeline an actual quality gate, not just a notification.

**Continuous Integration vs. Continuous Delivery/Deployment**
- *CI*: automatically build and test every change.
- *CD*: automatically package and ship the result — anything from publishing a Docker image to deploying a model endpoint. Many teams start with CI only, and add CD once they trust the gates.

## A Simple Example

Here's a minimal GitHub Actions workflow that ties together the ideas from earlier modules: it checks out the code, rebuilds the exact environment from a pinned dependency file (reproducibility), and runs the test suite (the CI gate) on every push and pull request.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest --maxfail=1 --disable-warnings
```

What happens here:

1. Every push or pull request to `main` triggers the workflow.
2. GitHub spins up a clean Ubuntu runner.
3. It checks out your repo, installs Python, and installs your pinned dependencies from `requirements.txt` — reproducing your environment exactly, not "whatever happens to be installed."
4. It runs your test suite. If any test fails, the job fails.
5. If you've turned on branch protection for `main` and required this workflow to pass, GitHub blocks the merge until it's green.

You could extend the `steps` list with things like linting, data-schema checks, or a quick model-evaluation script — each one becomes another gate.

## Key Takeaways / Best Practices

- **Keep workflows in version control alongside the code.** The `.github/workflows/` folder is just YAML in your repo — it's reviewed and versioned like everything else.
- **Pin your dependencies.** CI is only meaningful if it rebuilds the same environment every time; unpinned versions defeat the purpose.
- **Start small.** A single workflow that installs dependencies and runs tests is a complete, useful CI pipeline. Add complexity only as needed.
- **Make failures a hard stop.** Use branch protection so a red pipeline actually blocks merging — a gate no one is required to pass isn't a gate.
- **Separate CI from CD.** Test-on-every-push is safe to run constantly; deployment steps should usually be more deliberate (e.g., only on merge to `main`, or with a manual approval).
- **Keep pipelines fast.** Slow pipelines get ignored or bypassed; cache dependencies and only run the checks that matter for each change.
- **Treat the pipeline as part of the product.** Just like your model code, your CI/CD setup should be reviewed, improved, and trusted over time — it's the safety net for everything else in your MLOps workflow.
