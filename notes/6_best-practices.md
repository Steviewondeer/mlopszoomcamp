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

Would you like me to make you a **simplified diagram** that shows how these pieces connect (from test → Docker → Lambda simulation → model → assert → CI)?
It’d help visualize how the full “testing loop” works.
