# Topic D Self-Check Guide

This guide is student-facing. It explains how to check your work without using SWE-bench gold patches, hidden `test_patch` files, upstream fixing commits, or another team's solution.

## What You Can Use

- The assigned repository at the `base_commit` in `tasks.csv`.
- The issue statement in `problem_statements/`.
- Documentation and tests that already exist in that repository at `base_commit`.
- The test suggestions in `public_tests.csv`.
- Reproduction tests or scripts that you write yourself.
- AI assistance, if you verify its claims and submit transcripts.

## What You Must Not Use

- SWE-bench gold patches.
- SWE-bench `test_patch` files.
- Original upstream fixing PRs or commits.
- Public solution repositories or another team's patch/transcript.

## Step 1: Clone Each Repository

Create a local checkout for each instance under `repos/<instance_id>/`:

```bash
mkdir -p repos
git clone https://github.com/django/django.git repos/django__django-16485
cd repos/django__django-16485
git checkout 39f83765e12b0e5d260b7939fc3fe281d879b279
```

Repeat for each row in `tasks.csv`. Use the `repo` and `base_commit` columns; do not work from the latest branch.

## Step 2: Reproduce the Issue

Before writing a fix, create a small reproduction:

- a focused script;
- a local test you write yourself; or
- a command showing the buggy behavior from the issue statement.

Save the command and observed output in `analysis/root-cause.md` or the per-instance log.

## Step 3: Apply Your Patch Locally

After creating `patches/<instance_id>.patch`, check that it applies cleanly:

```bash
cd repos/<instance_id>
git apply --check ../../patches/<instance_id>.patch
```

This does not modify the checkout. It only verifies that the patch can apply to the assigned base commit.

## Step 4: Run Existing Public Tests

Use `public_tests.csv` as a starting point. These commands point to tests that already exist in the original repository or nearby modules:

```bash
python -m pytest tests/test_ext_autodoc_automodule.py
```

Some SWE-bench `FAIL_TO_PASS` tests are created by instructor-only `test_patch` files and may not exist in the base repository. If a named test does not exist, do not search for the hidden `test_patch`; write your own reproduction test from the issue statement.

## Step 5: Build the Prediction Package

Create the prediction file required by the course submission path. A common JSON shape is:

```json
[
  {
    "instance_id": "django__django-16485",
    "model_patch": "diff --git ..."
  }
]
```

## Step 6: Check Your Submission Package

Before submitting, check that every assigned instance has a patch file, reproduction evidence, root-cause
notes, local test evidence, transcript, and prediction entry.

## What The Instructor Will Run

The instructor may run:

- package validation;
- patch apply checks;
- selected public tests;
- hidden or official SWE-bench-style evaluation;
- report and transcript review.

Passing public tests is necessary but not sufficient. Your report must explain the root cause, the fix, and why your patch is general rather than overfit to one displayed symptom.
