**“Best Practices — Integration Tests”** video : https://www.youtube.com/watch?v=lBX0Gl7Z1ck&list=PL3MmuxUbc_hIUISrluw_A7wDSmfOhErJK&index=33.
---

## 🧩 Context Recap

In the previous module:

* We had **unit tests** for individual functions in a Lambda-based streaming service.
* The Lambda code was **refactored** into a class (`ModelService`) to make testing easier (no S3 downloads at import time).
* Unit tests checked small, isolated pieces: feature preparation, base64 decoding, simple predictions, etc.

But unit tests don’t verify if **everything works together** — the API, Docker container, dependencies, S3, etc.
That’s where **integration tests** come in.

---

## 🔍 What This Video Adds

We now want to test the **entire service end-to-end**:

* Run it in **Docker** (as if it’s deployed).
* Send a **request** to it.
* Assert that the **response** is correct.

So this video teaches:

1. Turning a *manual test script* into a *real automated integration test*.
2. Making the test independent from AWS (so it can run locally).
3. Automating the whole process with a **bash script** and **Docker Compose**.
4. Making it **CI/CD-ready** — so it can run automatically in pipelines.

---

## 🧱 Step 1 — Turning `test_docker.py` into a Real Test

Previously, the file just printed the response and “eyeballed” it.
Now we:

* Define an **expected response** (a dict with prediction, model version, etc.).
* Use `assert actual == expected`.

### 💡 Key Concept: `assert`

In Python, `assert condition` means:

* If `condition` is true → do nothing.
* If false → raise an `AssertionError` (which makes tests fail).

You can run these tests either:

* Manually (`python file.py`), or
* Automatically with **pytest** — which gives nice reports.

---

## 🧮 Step 2 — Making Assertions More Informative

Instead of just printing “AssertionError,” we use a package called **DeepDiff**.

### 🧰 DeepDiff

* Compares two dictionaries deeply.
* Shows *exactly where* they differ.
* Example output:

  ```
  {'type_changes': {"root['predictions'][0]['version']": {'old_value': None, 'new_value': 'test123'}}}
  ```

This makes debugging much easier than eyeballing differences.

We can then assert:

```python
assert 'type_changes' not in diff
```

So the test passes only if the dictionaries are *identical*.

---

## 🧮 Step 3 — Handling Floating-Point Precision

Predictions are floats (e.g., 123.45678).
Two runs may differ slightly due to floating-point arithmetic.

DeepDiff allows a **tolerance**, e.g.:

```python
DeepDiff(expected, actual, significant_digits=1)
```

Meaning: ignore differences beyond 1 decimal digit.
✅ Useful for ML models where minor float differences are fine.

---

## ☁️ Step 4 — Avoiding External Dependencies (S3)

Currently, the test loads the model from **AWS S3** — which is bad for tests:

* Slower.
* Requires credentials.
* Could fail offline.
* Could break if model is deleted.

### ✅ Solution:

Modify the code to allow a **local model path** (`model_location`):

* If `model_location` is set → load from that path.
* Else → default to S3.

This way:

* For tests → use a **local model file**.
* For production → use **S3**.

This makes your test **self-contained and deterministic**.

---

## 📁 Step 5 — Creating the Integration Test Setup

A new folder: `integration_test/`

* Contains `test_docker.py`
* Contains a **copy of the model** downloaded from S3
  (so the test doesn’t need to fetch it)
* Contains a **bash script (`run.sh`)** to automate everything

The model files are small enough to commit to Git — so they’re available anywhere tests run.

---

## 🧰 Step 6 — Bash Script Automation (`run.sh`)

This script:

1. **Builds the Docker image** (fresh each time)
2. **Runs** the container with correct environment variables
3. **Runs** the test file against the container
4. **Stops** the container afterward
5. **Returns** an error code (0 = success, 1 = failure)

### 🧱 Bash Concepts Explained

* **Shebang (`#!/usr/bin/env bash`)**
  Tells Linux: run this script using Bash.
* **`chmod +x run.sh`** → make it executable.
* **`set -e`** → stop script if any command fails.
* **`$?`** → gives exit code of last command.
* **`if [ "$error_code" -ne 0 ]`** → check if it failed.

This ensures CI/CD (like GitHub Actions or Jenkins) can detect failure automatically.

---

## 🐋 Step 7 — Using Docker Compose

Instead of running `docker run` with lots of flags:

* Create a `docker-compose.yml` file that defines:

  * The image to run.
  * Environment variables.
  * Mounted volumes (for local model).
  * Ports.

Then run:

```bash
docker compose up -d
```

to start the service in **detached mode**,
and `docker compose down` to stop it.

---

## 🧪 Step 8 — Automating Everything in CI/CD Style

The `run.sh` script now:

1. Builds the Docker image (with timestamp tag).
2. Runs it via Docker Compose.
3. Waits 1 second (to let service start).
4. Runs the integration test (`python test_docker.py`).
5. Checks the error code.
6. If failed:

   * Prints **container logs**.
   * Stops everything.
   * Exits with nonzero code → CI marks as failed.
7. If successful → exits with 0 → CI marks as passed.

That’s exactly how **continuous integration pipelines** (like GitHub Actions) operate.

---

## 🪣 Step 9 — Next Step Teaser

Currently, `KinesisCallback` (the function that writes to AWS Kinesis) is disabled in tests (`TEST_RUN=True`).
In the next video, we’ll learn to **test AWS services locally** (using **LocalStack**), so you can test S3, Kinesis, etc. *without* needing AWS credentials.

---

## 🧠 Key Takeaways for You as a Junior Data Scientist

| Concept                     | What it means in production                                                     |
| --------------------------- | ------------------------------------------------------------------------------- |
| **Unit test**               | Tests one small piece of logic (e.g., function). Fast, isolated.                |
| **Integration test**        | Tests full system end-to-end (service, API, Docker, etc.). Slower, but crucial. |
| **DeepDiff**                | Makes test failures readable and actionable.                                    |
| **Tolerance in floats**     | Avoids false failures due to floating-point precision.                          |
| **Model location override** | Lets you test locally without cloud dependencies.                               |
| **Docker Compose**          | Orchestrates multiple containers (e.g., app + dependencies) easily.             |
| **run.sh script**           | Automates test build → run → teardown. A pattern common in CI/CD.               |
| **Exit code (0/1)**         | The standard way systems know if tests passed or failed.                        |



---

# Module 6 — Best Engineering Practices (part: testing Kinesis with LocalStack)

## Recap (where we are)

* You already have:

  * **Unit tests** (with `pytest`) for small functions in `ModelService` (after refactor).
  * An **integration test** that runs the service in **Docker** and asserts the HTTP response (DeepDiff for nice diffs).
* **What’s missing:** we didn’t test the **Kinesis callback** (the bit that writes predictions to a Kinesis stream).

---

## Goal of this video

**Test the Kinesis integration locally** (no real AWS) by using **LocalStack** and extend the integration tests to verify that:

1. the service returns the right response **and**
2. a **record is written to Kinesis** with the expected content.

> ✅ **LocalStack**: a local emulator for AWS services (Kinesis, S3, etc.). Lets you run & test AWS code on your machine, no cloud needed.

---

## Step 1 — Spin up LocalStack (Kinesis only) with Docker Compose

* Add a **`localstack`** service in `docker-compose.yml`:

  * Image: `localstack/localstack`
  * Expose the **edge port** (e.g., 4566) — that’s where AWS SDKs/CLI talk to.
  * Set env var `SERVICES=kinesis` to start **only Kinesis** (faster).
* Start it:

  ```bash
  docker compose up localstack
  ```
* Use **AWS CLI** to point to LocalStack instead of AWS:

  ```bash
  aws kinesis list-streams --endpoint-url http://localhost:4566
  ```

  → Should return no streams initially (LocalStack is empty).

> 🧰 **endpoint-url**: tells AWS CLI (or your SDK) to talk to **LocalStack** instead of the real AWS endpoint.

---

## Step 2 — Create a Kinesis stream in LocalStack

* Create the stream:

  ```bash
  aws kinesis create-stream \
    --stream-name ride_predictions \
    --shard-count 1 \
    --endpoint-url http://localhost:4566
  ```
* Confirm:

  ```bash
  aws kinesis list-streams --endpoint-url http://localhost:4566
  ```

  → You should see `ride_predictions`.

> 🧠 **Kinesis basics**: a stream has **shards** (think partitions). You **put records** into the stream; consumers read via **shard iterators**.

---

## Step 3 — Point your service to LocalStack (not AWS)

* Add an env var for the service to use when creating the Kinesis client:

  * `KINESIS_ENDPOINT_URL=http://localstack:4566`
    (Inside Docker Compose networks, you address services by their compose **service name**, not `localhost`.)
* In code, wrap Kinesis client creation:

  ```python
  def create_kinesis_client():
      endpoint = os.getenv("KINESIS_ENDPOINT_URL")
      if endpoint:
          return boto3.client("kinesis", endpoint_url=endpoint, region_name="us-east-1")
      return boto3.client("kinesis", region_name="us-east-1")
  ```
* Use this factory in your `KinesisCallback` wiring.

> 💡 **Inside vs. outside Docker Compose**:
>
> * **Inside** the compose network: use `http://localstack:4566`.
> * **Outside** (your terminal): use `http://localhost:4566`.

---

## Step 4 — Keep services running to inspect Kinesis

* Temporarily **don’t** tear down containers at the end of the integration script.
* After sending the request to your service, confirm the stream exists and records were written.

---

## Step 5 — Manually read from Kinesis (sanity check)

* With AWS CLI:

  1. Get a **shard iterator**:

     ```bash
     aws kinesis get-shard-iterator \
       --stream-name ride_predictions \
       --shard-id shardId-000000000000 \
       --shard-iterator-type TRIM_HORIZON \
       --endpoint-url http://localhost:4566
     ```
  2. Fetch **records** using that iterator:

     ```bash
     aws kinesis get-records \
       --shard-iterator <iterator> \
       --endpoint-url http://localhost:4566
     ```
* You’ll see the payloads the service wrote.
  (In this LocalStack run, the payload was **not** base64-encoded; JSON could be read directly.)

> ⚠️ **Note**: Real Kinesis returns base64-encoded `Data`. LocalStack behavior may differ; handle both in tests if needed.

---

## Step 6 — Automate Kinesis validation with a Python test script

* Create **`test_kinesis.py`** that:

  1. Creates a Kinesis client using `KINESIS_ENDPOINT_URL`.
  2. Gets a **shard iterator** for `ride_predictions`.
  3. Reads **records**, parses the JSON (or decodes base64 then JSON).
  4. Compares the **actual record** to an **expected record** using **DeepDiff** with float tolerance if needed.
* The **expected record** should mirror what your service writes (e.g., `ride_id`, `model_version`, `ride_duration`, etc.).

> 🧰 **DeepDiff reminder**: `DeepDiff(expected, actual, significant_digits=1)` to avoid failing on tiny float diffs.

---

## Step 7 — One-click integration runner (`run.sh`)

Extend your bash script to do all the things:

1. **Build** the service image (timestamped tag).
2. `docker compose up -d` for both:

   * **backend** (your service)
   * **localstack** (Kinesis)
3. **Sleep 1s** (give containers time to start).
4. Run the existing **HTTP integration test** (the one that hits the service and asserts the response).

   * If it fails → print `docker compose logs`, bring everything **down**, `exit 1`.
5. Run the new **Kinesis test** (`test_kinesis.py`).

   * If it fails → print logs, bring everything **down**, `exit 1`.
6. If both pass → `docker compose down`, `exit 0`.

> 🧪 You now have **two test suites**:
>
> * **Unit tests**: `pytest tests/`
> * **Integration tests**: `integration_test/run.sh` (service + Kinesis via LocalStack)

---

## What you learned (production lens)

* **LocalStack** lets you test AWS integrations without touching AWS (faster, cheaper, safer).
* **Endpoint overrides** (`endpoint_url`) are the key to pointing SDKs/CLI at LocalStack.
* **Service wiring via env vars** is crucial for **12-factor** style apps (no hardcoded endpoints).
* **Kinesis test flow**:

  * Create stream → run service → send request → read from stream → assert on record.
* **DeepDiff** boosts test readability (exact diffs).
* **run.sh** + **Docker Compose** = a CI-friendly way to run **end-to-end** tests consistently anywhere.

---

## Tiny glossary (quick reads)

* **Shard / Shard iterator (Kinesis):** a shard is a unit of scale in a stream; an iterator is a token to read from a shard.
* **LocalStack:** local emulator for AWS services.
* **endpoint-url:** tells AWS tools where to connect (LocalStack vs AWS).
* **Callback:** function run **after** a prediction (here, to write to Kinesis).
* **DeepDiff:** Python lib to compare nested dicts and report precise differences.

---




# Module 6 — Code Quality Tools: **Linting** & **Formatting**

## Why this lesson (what’s new vs earlier tests)

* Before: we improved **reliability** (unit & integration tests).
* Now: we improve **code quality & consistency** (how code **looks** and **reads**) so teams can maintain it.

  * **Linting** = find issues & style violations.
  * **Formatting** = automatically fix the layout/style.

---

## PEP 8 (the baseline)

* **PEP 8** is Python’s style guide: naming, spacing, line length, imports, etc.
* Tools can **check** you follow it and **suggest/fix** issues.

> quick def: **PEP** = Python Enhancement Proposal. PEP 8 is the style guide; other PEPs define other standards.

---

## Linter used in the video: **pylint**

* What it does:

  * Checks PEP 8-ish style.
  * Catches **common mistakes** (unused variables, import order, too-long lines, global state, missing docstrings, etc.).
* Install (as a **dev** dependency):

  ```
  pip install -U pylint
  ```
* Run (whole project or specific files):

  ```
  pylint .
  # or
  pylint model.py tests/
  ```

### Use it inside VS Code

* Enable **Python > Linting: Pylint** (Ctrl+Shift+P → “Select Linter” → Pylint).
* You’ll see **squiggly underlines** with messages like:

  * trailing whitespace
  * missing module/function/class docstring
  * variable naming not snake_case
  * line too long
  * unused arguments (e.g., `context` in Lambda handlers)

> tip: many warnings are valuable; some are noisy for app code (e.g., mandatory docstrings everywhere). You can **configure/disable** selectively.

---

## Configuring pylint

### Project-wide config via **pyproject.toml** (preferred)

Inside `pyproject.toml`:

```toml
[tool.pylint.'MAIN']
# top-level options go here if needed

[tool.pylint.'MESSAGE CONTROL']
disable = [
  "missing-module-docstring",
  "missing-class-docstring",
  "missing-function-docstring",
  "too-few-public-methods",
  # add more as your team agrees
]
```

> Many modern tools read config from `pyproject.toml`, so you don’t need separate files.

### Local, per-line/per-scope suppressions

Add inline comments to **just this spot**:

```python
def lambda_handler(event, context):  # pylint: disable=unused-argument
    ...
```

Or on a class/function block to limit suppression scope.

> junior note: keep suppressions **surgical**. If you disable everything globally, linting loses its value.

---

## Fixing lint warnings the video shows

* **Docstrings**: either add short docstrings **or** disable for now (project-wide or locally).
* **Trailing whitespace / final newline**: a **formatter** will fix these.
* **Line too long (base64 blobs / huge JSON)**:

  * Move big literals to **external files** (e.g., `data.b64`, `event.json`), then read them:

    ```python
    from pathlib import Path
    def read_text(file: str) -> str:
        return Path(__file__).parent.joinpath(file).read_text(encoding="utf-8").strip()
    data_b64 = read_text("data.b64")
    ```
  * This also makes tests cleaner.
* **Unused args** (e.g., Lambda `context`): disable locally (`pylint: disable=unused-argument`).
* **Naming** (e.g., capital `X`): in ML it’s common (`X`, `y`). Either accept the warning or disable that rule.

---

## Auto-formatter **black**

* Purpose: **opinionated** formatting (spacing, line breaks, wrapping).
* Install:

  ```
  pip install -U black
  ```
* Try a **preview** first:

  ```
  black --diff .
  ```
* Apply:

  ```
  black .
  ```

### Useful options & tweaks from the video

* Keep single quotes as-is:

  ```
  black --skip-string-normalization .
  ```
* Preserve multi-line layout by **adding a trailing comma** inside bracketed lists/dicts/args.
* Configure in `pyproject.toml`:

  ```toml
  [tool.black]
  line-length = 88
  target-version = ["py39"]
  skip-string-normalization = true
  ```

> junior note: Let black “win” on style. You stop arguing over nits and ship code.

---

## Import sorter **isort**

* Purpose: group & **sort imports** deterministically.
* Install:

  ```
  pip install -U isort
  ```
* Preview & run:

  ```
  isort --diff .
  isort .
  ```

### Configuration (in `pyproject.toml`)

* Basic:

  ```toml
  [tool.isort]
  # example profiles; you can omit if you want full control
  # profile = "black"
  length_sort = true         # the video uses size/length-based ordering preference
  ```
* If `profile = "black"` fights with your preferences, remove it and set options explicitly.

> junior note: isort will place stdlib imports first, then third-party, then local, and can alphabetize or length-sort within groups.

---

## Putting it all together (the order matters)

1. **isort** (fix imports)
2. **black** (format code)
3. **pylint** (lint; should be mostly clean after formatting)
4. **pytest** (unit tests) + your **integration tests** script

Example one-liner (from the video’s flow):

```bash
isort .
black .
pylint -ry .
pytest tests/
```

* `pylint` exits **non-zero** if warnings/errors remain → CI will **fail** (good).
* Keep only **intentional** disables in config to avoid masking real issues.

---

## Small but important details the video highlights

* **Exit codes**: 0 = success; non-0 = fail. CI/CD depends on this.
* **UTF-8 encoding** when reading files (esp. on Windows):

  ```python
  Path(...).read_text(encoding="utf-8")
  ```
* **Path handling**: use `pathlib` and paths relative to the **test file** so tests run from any working dir.

---

## What to practice (hands-on, quick wins)

1. Add **pylint**, **black**, **isort** as **dev** deps.
2. Create a **`pyproject.toml`** with:

   * `[tool.pylint.'MESSAGE CONTROL']` minimal disables,
   * `[tool.black]` with `skip-string-normalization = true`,
   * `[tool.isort]` with your chosen ordering (e.g., `length_sort = true`).
3. Move any long base64/JSON literals in tests to small **fixture files**; load them with `pathlib` + `encoding="utf-8"`.
4. Run the full sequence: `isort . && black . && pylint -ry . && pytest`.
5. Fix/suppress intentionally:

   * local `# pylint: disable=unused-argument` on Lambda handler,
   * docstrings (either add or disable those rules).
6. Commit; re-run your **integration test** script to ensure nothing broke.

---

## Tiny glossary

* **Linter**: static analyzer that flags style and potential bugs.
* **Formatter**: tool that **rewrites** code layout automatically.
* **PEP 8**: community style guide for Python.
* **`pyproject.toml`**: a central config file many Python tools read.
* **Exit code**: program return status (0=OK). CI uses it to mark jobs pass/fail.

---



# What I’d update in 2025

## 1) Consider **Ruff** as your default linter/formatter/import sorter

* **Why**: it’s blazing fast (Rust), widely adopted, and can replace **pylint (most rules)** + **isort** + **many flake8 plugins**. It also has a built-in formatter (`ruff format`) if you want to ditch Black (Black’s still great—choose one).
* **Balanced choice**:

  * **Ruff + Black** → fastest lint + the most battle-tested formatter.
  * Or **Ruff-only** (lint + `format` + import sort) → simplest toolchain.
* **Keep `pylint`?** Only if you rely on a few design-level checks Ruff doesn’t aim to cover. Most teams don’t need both anymore.

## 2) Type checks: add **mypy** or **pyright**

* Even light annotations + a type checker catches tons of bugs earlier than runtime.
* **Pick one**: `mypy` (Python ecosystem standard) or `pyright` (fast, MS). You can run either via **pre-commit** and CI.

## 3) Testing extras that pay off

* **pytest-cov** (coverage), **pytest-xdist** (parallel), **hypothesis** (property-based tests for tricky logic/edge cases).
* **mutation testing** (e.g., **mutmut**) if you want to measure test *quality* (advanced but amazing).

## 4) AWS mocking/emulation

* **LocalStack** is still the best for **Kinesis/SQS/SNS** integration tests.
* **Moto** is lighter/faster for **S3/DynamoDB** unit tests (where supported).
* Rule of thumb: **unit** → Moto/fakes; **integration** → LocalStack.

## 5) Dependency & env management

* Consider **uv** (from Astral) for **ultra-fast** installs/locking (`uv pip`, `uv venv`, `uv lock`). It’s getting very popular.
* Or **Poetry/Hatch** if you want full project management (builds + publishing).
* Add **pip-audit/uv pip audit** (or **safety**) in CI for vuln scans.

## 6) Security/quality checks (cheap wins)

* **bandit** (Python security lint), **detect-secrets** or **gitleaks** (prevent secret commits).
* **EditorConfig** to enforce basics (final newline, UTF-8, indent…) across all editors.

## 7) Pre-commit + CI

* Run *everything* locally via **pre-commit**, then mirror in CI:

  * ruff/black/isort (if used), mypy/pyright, pytest, bandit, pip-audit.
* Use **pre-commit.ci** or GitHub Actions to auto-fix on PRs.

## 8) Docker & compose tweaks

* Use **multi-stage builds** and a slim base (e.g., `python:3.11-slim` or `3.12-slim`).
* Cache deps smartly; consider **uv** in Docker for speed.
* Pin Python (3.11/3.12) consistently with your type checker/pyproject.

---

# Drop-in configs (copy/paste and tweak)

## Option A — **Ruff + Black** (my favorite balance)

**pyproject.toml**

```toml
[tool.black]
line-length = 88
target-version = ["py311"]
skip-string-normalization = true

[tool.ruff]
line-length = 88
target-version = "py311"
# Enable common rule sets (flake8 + tidy imports + pyupgrade + bugs)
select = ["E","F","I","UP","B","SIM","C4","RUF"]
ignore = [
  # Examples to relax for ML code:
  "E501",  # handled by black for wrapping
]
# isort-equivalent behavior
[tool.ruff.lint.isort]
force-sort-within-sections = true
length-sort = true

[tool.mypy]
python_version = "3.11"
warn_unused_ignores = true
ignore_missing_imports = true
strict_optional = true
disallow_untyped_defs = false  # ease-in for teams starting with typing

[tool.pytest.ini_options]
addopts = "-q --maxfail=1"
testpaths = ["tests"]

[tool.bandit]
skips = ["B101"]  # assert used in tests; tune as needed
```

**.pre-commit-config.yaml**

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks:
      - id: black
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.2
    hooks:
      - id: mypy
        additional_dependencies: []
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.9
    hooks:
      - id: bandit
        args: ["-q", "-r", "."]
```

Run once:

```bash
pip install ruff black mypy bandit pre-commit
pre-commit install
```

**Makefile** (handy aliases)

```makefile
.PHONY: fmt lint type test all
fmt:
	ruff --fix .
	ruff format .
	black .

lint:
	ruff .
	bandit -q -r .

type:
	mypy .

test:
	pytest -q

all: fmt lint type test
```

## Option B — **Ruff-only** (simplest stack)

* Drop Black and isort. Replace `fmt` with just `ruff --fix` and `ruff format`.
* Keep mypy/pyright as you like.

---

# Small pragmatic tips

* **Float diffs in tests**: keep using DeepDiff’s `significant_digits` or compare with tolerances (`math.isclose`) to avoid flaky tests.
* **Long blobs in tests**: externalize to fixtures (`tests/fixtures/...`) and read with `pathlib` + `encoding="utf-8"`.
* **Docstrings**: don’t fight the tools—write **short** docstrings for public functions/classes; disable per-scope where it truly adds no value.
* **Lambda `context`**: `# pylint/ruff: disable=unused-argument` at the function line is fine.
* **Coverage gate**: add `pytest --cov=yourpkg --cov-fail-under=75` to nudge quality up over time.
* **Secrets**: add `detect-secrets` or `gitleaks` to pre-commit now. It prevents painful incidents later.
* **Local AWS**: Moto for **unit** S3/DDB; LocalStack for **integration** (Kinesis/etc.). Keep both tools in your belt.

---



# Pre-commit hooks — what, why, how (with plain-english explanations)

## 1) The problem we’re solving

When you commit code, it’s easy to forget to run formatters, linters, or tests. That leads to:

* messy diffs (inconsistent spacing/import order),
* preventable bugs (lint catches them early),
* red CI pipelines for things you could have fixed locally.

**Goal:** run a small “quality gate” automatically *before* a commit is accepted.

---

## 2) What is a Git hook?

**Git hook:** a small program Git runs at specific moments.

* **pre-commit** hook runs **before** Git finishes a commit.
  If the hook **exits with non-zero** (i.e., reports failure), the commit is **blocked**.

> Think of it like a nightclub bouncer for your repo: if your code doesn’t meet the dress code (format/lint/tests), you don’t get in.

---

## 3) What is the `pre-commit` tool (Python package)?

`pre-commit` is a **framework** that:

* lets you **declare** your checks in one YAML file: `.pre-commit-config.yaml`,
* **installs** the Git hook for you,
* runs the checks **only on changed files** by default (fast),
* can **auto-fix** certain issues (e.g., formatter rewrites files).

You still use Git normally (`git add`, `git commit`), but the framework slips in checks before the commit completes.

---

## 4) What are the checks we run?

From the video, four families of checks:

* **Housekeeping hooks** (built-in): trim trailing spaces, ensure a newline at the end of file, validate YAML, block huge files.
* **Formatter**:

  * **Black** formats Python code so style is consistent. Opinionated. Auto-fixes.
* **Import sorter**:

  * **isort** puts imports in a stable order (and groups stdlib/3rd-party/local). Auto-fixes.
* **Linter**:

  * **pylint** statically analyzes code: naming conventions, unused vars, risky patterns, etc.
* **Tests**:

  * **pytest** runs your test suite (or a subset). If tests fail, commit fails.

> **Formatter vs Linter**
>
> * *Formatter* changes how code **looks** (spacing/quotes/line breaks) but **not** what it does.
> * *Linter* warns about likely **problems** (logic smells, unused imports, risky code).

---

## 5) Minimal setup (exact commands)

1. **Install the framework** (in your project’s virtual env):

```bash
pip install pre-commit
```

2. **Create a Git repo** (if you’re treating a subfolder as a standalone project, as in the video):

```bash
git init
```

3. **Generate a starter config**:

```bash
pre-commit sample-config > .pre-commit-config.yaml
```

4. **Install the Git hook** (writes `.git/hooks/pre-commit` for you):

```bash
pre-commit install
```

5. **Try it on all files once** (normalize the repo):

```bash
pre-commit run --all-files
```

Now on every `git commit`, the hooks run automatically.

---

## 6) A practical `.pre-commit-config.yaml` (video-aligned)

### Built-in housekeeping + isort + black + pylint + pytest

```yaml
repos:
  # 1) Built-in housekeeping hooks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace           # removes extra spaces at line ends
      - id: end-of-file-fixer             # ensures final newline
      - id: check-yaml                    # YAML files must parse
      - id: check-added-large-files       # prevents committing very large files

  # 2) isort — sort & group imports (auto-fix)
  - repo: https://github.com/pycqa/isort
    rev: 5.13.2
    hooks:
      - id: isort

  # 3) black — code formatter (auto-fix)
  - repo: https://github.com/psf/black
    rev: 24.8.0            # note: black tags have no 'v' prefix
    hooks:
      - id: black
        language_version: python3.11      # or your interpreter

  # 4) pylint — linter (as a local hook)
  - repo: local
    hooks:
      - id: pylint
        name: pylint
        entry: pylint
        language: system
        types: [python]
        args: ["-ry"]                     # -r: report, -y: yes to all; tailor as you like

  # 5) pytest — run tests before commit
  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: pytest
        language: system
        pass_filenames: false             # run the suite, not only staged files
        args: ["tests"]
```

> After editing, re-run `pre-commit install` (safe) and try `pre-commit run --all-files`.

---

## 7) What happens on commit? (the “bouncer logic”)

* You run `git commit -m "msg"`.
* Hooks execute in order:

  1. **isort** may rewrite imports → **commit fails** (by design) → `git add .` → `git commit` again.
  2. **black** may rewrite formatting → same re-add/commit step.
  3. **pylint** may report warnings as errors → fix code → commit again.
  4. **pytest** may fail tests → fix tests/code → commit again.
* If all hooks exit **0**, the commit is accepted.

**Why fail after auto-fix?**
Because your working tree changed. Git wants you to **review** the changes and explicitly add them to the commit.

---

## 8) Local vs shared

* The **hook script** in `.git/hooks/` is **local** (not committed).
  Each teammate runs `pre-commit install` once after cloning.
* The **`.pre-commit-config.yaml`** lives in the repo → everyone shares the same rules.

---

## 9) Common errors, with translations

* **“Hook not found / hooks don’t run”** → You likely forgot `pre-commit install`.
* **“Could not fetch tag …”** → Bad `rev` value; use an existing tag (e.g., Black has no `v` prefix).
* **“language_version …”** → Pin the Python version that matches your env (`python3.10`, `python3.11`, etc.).
* **“Takes too long”** → Consider:

  * run only **fast checks** in pre-commit (format/lint, a *smoke* test subset),
  * run full test suite in **CI**.

---

## 10) 2025 upgrades (clear, practical)

### A) Replace isort + most lint with **Ruff** (faster, simpler)

* **Ruff** = super-fast linter + import sorter.
* You can keep Black for formatting, or use `ruff format` to replace Black too.

**Ruff config in pre-commit (lean stack):**

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff           # lint + import sort; add --fix to auto-fix
        args: [--fix]
      - id: ruff-format    # formatter (use this OR keep Black)
```

**Why Ruff?**

* One tool covers **isort-like import order** + **flake8/pylint-like rules** (most of what you need) with blazing speed.
* Fewer tools to configure and update.

> If you love Black’s exact style, keep Black and use Ruff only for lint/imports.

### B) Add **type checks** (catches whole classes of bugs)

* **mypy** (classic) or **pyright** (fast, great editor support).

```yaml
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.2
    hooks:
      - id: mypy
```

*(Add a `mypy.ini` later to set strictness gradually.)*

### C) Secrets & security

* **detect-secrets** / **gitleaks** → block committing tokens/keys.
* **bandit** → static security analysis for Python.

```yaml
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets

  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.9
    hooks:
      - id: bandit
        args: ["-q", "-r", "."]
```

### D) Mirror hooks in CI

* Use **pre-commit.ci** or run `pre-commit run --all-files` in your CI so PRs are auto-fixed/validated the same way.

---

## 11) Glossary (super short)

* **Trailing whitespace**: spaces at the end of a line. No meaning, but noisy diffs.
* **Final newline**: a newline at the end of a file (POSIX convention; many tools expect it).
* **YAML**: config file format using indentation. Invalid YAML = tools can’t parse configs.
* **Auto-fix**: a hook that edits your files (formatters, import sorters).
* **Exit code**: `0` = success, non-zero = failure → Git blocks the commit.
* **Pinning**: specifying exact versions (tags) in config so tools don’t change under you.
* **Staged files**: files you added with `git add` and are about to commit.

---

## 12) Suggested starter packs

### Starter (simple + fast)

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: pytest
        language: system
        pass_filenames: false
        args: ["-q", "tests"]    # quick tests locally; run full suite in CI
```

### Classic (exactly like the video)

Use the longer config above with **isort + black + pylint + pytest**.

---

## 13) Your next steps

1. Pick **Ruff (+ Ruff format)** or **isort + Black + pylint**.
2. Add **pytest** hook (keep it quick).
3. Run `pre-commit install` and `pre-commit run --all-files` to normalize the repo.
4. Add types (mypy/pyright) and secrets scanning when you’re comfortable.



---

# Make & Makefiles — concepts you actually use in real projects

## 1) What problem does `make` solve?

You often have a repeatable workflow:

* “format + lint my code → run unit tests → run integration tests (Docker/LocalStack) → build image → publish image”.

Typing all of that (and remembering the flags + order) is error-prone.
**Make** lets you give this workflow names like `test`, `build`, `publish` and then it runs the right steps **in the right order** with one command.

**One-liner intuition:** `make` is a **task orchestrator** with **named tasks** and **dependencies**.

---

## 2) Vocabulary (gentle & precise)

* **Make**: a command-line tool that reads instructions from a file named **`Makefile`**.
* **Makefile**: a plain text file where you define **targets**, their **dependencies**, and the **recipe** (the shell lines to run).
* **Target**: the label you type, e.g. `make test`. Think of it as a **task name**.
* **Dependency (prerequisite)**: tasks a target needs **before** it can run. If `build` depends on `test`, running `make build` first runs `test`.
* **Recipe**: the shell commands for a target (the “how”). Each visible recipe line must start with a **tab**.
* **Variable**: reusable values inside the Makefile (e.g., `IMAGE=my-svc`). You can override them from the command line—handy for environments.
* **.PHONY**: a flag that says “this target is not a file on disk; always run it” (prevents “nothing to do” confusion).
* **DAG**: Directed Acyclic Graph — the structure formed by dependencies. `make` walks this graph in the correct order.

---

## 3) How `make` actually decides what to run (mental model)

Traditionally, make was about **files**: “build `app` from `app.o`, which comes from `app.c`”. If the **output file** exists and is **newer** than inputs, make skips the work.

In **task-runner mode** (common in ML/MLOps today), we usually mark targets as **.PHONY** so they **always run**, ignoring timestamps. Then dependencies simply define **order**:

```
publish  →  build  →  test  →  quality
```

Run `make publish`, and `make` executes the chain in the correct sequence.

---

## 4) One subtle but crucial behavior: “one shell per line”

By default, **each recipe line runs in its own fresh shell**. That means:

```make
target:
	echo "1"
	cd subdir
	pwd   # still shows the original dir – `cd` didn’t persist
```

**Why?** Because each line is a separate shell.

**Three ways to handle this:**

1. Put commands on **one line** using `&&`:

```make
target:
	cd subdir && pwd
```

2. Tell make to use **one shell for the whole recipe**:

```make
.ONESHELL:
SHELL := bash
target:
	cd subdir
	pwd        # works now
```

3. Use a separate script (`bash myscript.sh`) when the sequence is long.

---

## 5) Variables & evaluation timing (predictable tags)

You often want a **single build tag** reused across steps (build → test → publish). If you compute a tag with `date` multiple times, seconds or minutes can change mid-pipeline.

**Key concept: evaluate once, reuse everywhere.**

* `:=` assigns **now** (at parse time).
* `=` assigns **lazily** (re-evaluated when used).

```make
DATE := $(shell date +%Y%m%d-%H%M%S)  # one snapshot for this `make` run
TAG  ?= $(DATE)                       # default; can be overridden: `make TAG=dev build`
```

Now `TAG` is stable for all targets in this run → no “test used a different tag than build” bugs.

---

## 6) Dependencies as an execution plan (read it like a pipeline)

You can read dependency lines as **English**:

* `build: test quality`
  “Before **build**, run **test** and **quality**.”

* `publish: itest`
  “Before **publish**, run the **integration tests** (which themselves may depend on build).”

You’ve turned informal instructions into a deterministic **execution plan**.

---

## 7) Passing configuration around (without hard-coding)

Make variables can come from:

* the Makefile (defaults),
* your **environment** (`STREAM=pred make itest`),
* or the **CLI** (`make build IMAGE=myorg/trip-svc TAG=staging`).

Recipes can **export** variables into the shell they run, or you can pass them to called scripts:

```make
itest: build
	# pass a computed image name to your test runner
	LOCAL_IMAGE_NAME=$(IMAGE):$(TAG) bash integration-test/run.sh
```

In the script you just read `LOCAL_IMAGE_NAME`.

---

## 8) Where Make ends and scripts begin (good boundaries)

* Put **short glue** in the Makefile (variable wiring, dependency order).
* Put **procedural logic** (loops, conditionals, error handling) inside scripts (`.sh` or `.py`).
* Result: Make remains a readable **top-level playbook**, and scripts are testable standalone.

---

## 9) Pre-commit hooks vs Make (who does what?)

* **Pre-commit**: runs **fast, local checks** whenever you commit (formatting, import sorting, lint, quick tests).
* **Make**: runs **full workflows on demand** (integration tests with Docker/LocalStack, builds, publishes).

They **complement** each other:

* Pre-commit keeps diffs clean and catches obvious issues early.
* Make gives you reproducible pipelines you (and CI) can trigger explicitly.

---

## 10) Typical targets you’ll see (concepts, not code)

* `quality` → runs **formatter** (Black/Ruff), **import sorter** (isort/Ruff), and **linter** (pylint/Ruff).
* `test` → runs **unit tests** (pytest).
* `docker-up` / `docker-down` → start/stop dependencies (e.g., LocalStack + your service).
* `itest` → **end-to-end**: build an image, start services, run integration checks, tear down.
* `build` → builds a **tagged image** once per run (stable tag).
* `publish` → pushes the verified image to a registry (only after `itest` succeeds).
* `setup` → one-time project bootstrap (install deps, install pre-commit hook).

You can read the **graph** and understand your project’s **release policy** at a glance.

---

## 11) Common errors & quick fixes (you’ll hit these)

* **“missing separator.  Stop.”**
  Your recipe lines must start with a **tab**, not spaces.
* **Command needs multiple lines of state (like `cd`)**
  Use `.ONESHELL:` or join with `&&` on one line.
* **Variables not what you expect**
  Decide between `:=` (evaluate once) and `=` (lazy), and prefer `:=` for tags.
* **Hidden failures**
  In scripts, use `set -euo pipefail` so any failing command aborts the whole step.
* **Windows path/line endings**
  Prefer WSL for Docker-heavy flows; otherwise mind path syntax and CRLF.

---

## 12) 2025-friendly tooling notes (so you pick wisely)

* **Ruff** can replace **flake8 + isort + most pylint rules** and also do **formatting** (`ruff format`). It’s extremely fast.
  Many teams use **Ruff (lint + format)** + **pytest** + (optionally) **mypy/pyright** for types.
* **Black** is still great if you want its exact style, but don’t double-format with Ruff and Black at the same time.
* **Task runners** with friendlier syntax exist:

  * **`just`** (Justfile) — simple & popular.
  * **`task`** (Taskfile.yml) — YAML, cross-platform, parallelism.
    You can use these instead of Make; conceptually they do the same orchestration job.

---

## 13) How this fits your MLOps picture

* **Local integration**: Make drives Docker Compose + LocalStack to validate cloud flows (e.g., writing to Kinesis) **without real AWS**.
* **Reproducible images**: Compute tag once; the **same image** is tested and published.
* **CI/CD**: Your pipeline can simply call `make itest` and `make publish`. Same commands locally and in CI ⇒ fewer “works on my machine” issues.

---

## 14) Mini-Q&A (things juniors often ask)

**Q: Why not just put everything in shell scripts?**
A: You can, but you lose the **dependency graph** and **idempotent orchestration**. Make targets are discoverable (`make` can even print help) and compose cleanly.

**Q: Do I have to learn all Make syntax?**
A: No. For 95% of cases you need: targets, dependencies, variables, `.PHONY`, and maybe `.ONESHELL`. That’s it.

**Q: When should I choose Ruff vs pylint/isort/black?**
A: If your team is starting fresh, **Ruff** is a great all-in-one (lint + format + imports) and very fast. If you already use Black’s style strictly, keep Black and use Ruff for lint (`ruff`) without `ruff format`.

**Q: Can Make run in GitHub Actions/GitLab CI?**
A: Yes. Most CI configs end up as tiny wrappers like:

```yaml
- run: make itest
- run: make publish
```

Same commands locally and in CI.


