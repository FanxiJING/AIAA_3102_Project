# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## Project Overview

This is a university course project (AIAA 3102, Topic C — SWE-bench Codebase Adoption). The task is to fix five real bugs in mature Python libraries, produce patches verified through the SWE-bench harness, and write analysis explaining the reasoning behind each fix. The repository itself is a **meta-project** — it contains no application code, only metadata, problem statements, and submission scaffolding. All actual code work happens in external upstream repositories cloned from GitHub.

## Five Assigned SWE-bench Instances

| Instance | Library | Summary |
|---|---|---|
| `django__django-16485` | Django | `floatformat('0.00', 0)` raises `ValueError` |
| `django__django-14580` | Django | Generated migration references `models.Model` without importing `models` |
| `sphinx-doc__sphinx-8595` | Sphinx | Empty `__all__ = []` is ignored; autodoc documents all members |
| `pylint-dev__pylint-7080` | Pylint | `--recursive=y` ignores `ignore-paths` |
| `pytest-dev__pytest-7571` | Pytest | `caplog.set_level()` is not restored after a test |

There is also one optional practice instance: `astropy__astropy-12907` (in `tasks_throwaway.csv`).

## Key Data Files

- **`starter/tasks.csv`** — Assigned instances: `instance_id`, `repo`, `base_commit`, `FAIL_TO_PASS` and `PASS_TO_PASS` test lists, difficulty estimates. This is the source of truth for which commit to check out and which tests matter.
- **`starter/problem_statements/<instance_id>.md`** — The original bug report / issue text for each instance. These are the only allowed problem descriptions; do not search for upstream fixes.
- **`starter/public_tests.csv`** — Suggested existing test files and commands for locally verifying patches without hidden SWE-bench tests.
- **`starter/SELF_CHECK.md`** — Step-by-step guide for cloning repos, applying patches, running tests, and packaging predictions.
- **`starter/tasks_throwaway.csv`** — Optional practice instance metadata (same schema as `tasks.csv`).

## Workflow for Each Instance

1. Clone the upstream repo: `git clone https://github.com/<repo>.git repos/<instance_id>`
2. Check out the exact `base_commit` from `tasks.csv`.
3. Read the issue statement from `problem_statements/<instance_id>.md`.
4. Reproduce the bug locally (write a script or use existing tests).
5. Develop and review a patch.
6. Save the patch: `starter/patches/<instance_id>.patch`
7. Verify the patch applies: `cd repos/<instance_id> && git apply --check ../../starter/patches/<instance_id>.patch`
8. Run public tests from `public_tests.csv` to confirm no regressions.

## Commands for Running Tests

SWE-bench evaluation uses `sb-cli` (configured by the course instructor). For local self-checks, use the test commands in `public_tests.csv`:

```bash
# Django instances — use runtests.py
cd repos/django__django-16485
python tests/runtests.py template_tests.filter_tests.test_floatformat

cd repos/django__django-14580
python tests/runtests.py migrations.test_writer

# Sphinx
cd repos/sphinx-doc__sphinx-8595
python -m pytest tests/test_ext_autodoc_automodule.py

# Pylint
cd repos/pylint-dev__pylint-7080
python -m pytest tests/test_self.py::TestRunTC::test_ignore_path_recursive tests/test_self.py::TestRunTC::test_ignore_recursive

# Pytest
cd repos/pytest-dev__pytest-7571
python -m pytest testing/logging/test_fixture.py::test_change_level testing/logging/test_fixture.py::test_change_level_undo
```

Note: Some SWE-bench `FAIL_TO_PASS` tests exist only in hidden `test_patch` files and won't be present at `base_commit`. If a test doesn't exist, write your own reproduction test from the issue statement.

## Repository Structure

```
starter/
├── tasks.csv              # Assigned instances metadata
├── tasks_throwaway.csv    # Optional practice instance
├── public_tests.csv       # Suggested local test commands
├── SELF_CHECK.md          # Self-verification guide
├── problem_statements/    # Issue texts (one .md per instance)
├── patches/               # Your .patch files go here
├── analysis/              # orientation.md, root-cause.md, analysis.md
├── results/sb-cli/        # sb-cli output artifacts
└── logs/                  # AI interaction transcripts
```

## Academic Integrity Rules

- **Allowed**: source code at base_commit, project docs from that checkout, public API docs, provided issue text, AI assistance (with verification).
- **Forbidden**: SWE-bench gold patches, hidden `test_patch` files, upstream fixing PRs/commits, web searches combining instance_id with "fix"/"patch", copying another team's work.
- Overlap with the real fix is fine if you can explain your derivation trail.

## Prediction Format

When packaging for `sb-cli`, create a JSON file with one object per instance:

```json
[
  {
    "instance_id": "django__django-16485",
    "model_patch": "diff --git ..."
  }
]
```

## Deliverables Checklist

- `starter/patches/<instance_id>.patch` for all 5 instances
- `starter/patches/predictions.jsonl`
- `starter/analysis/orientation.md` — where the relevant logic lives per task
- `starter/analysis/root-cause.md` — mechanism-level explanation of each bug
- `starter/analysis/analysis.md` — verification table, guided-vs-one-shot comparison, cross-task observations, difficulties, AI usage declaration
- `starter/logs/<instance_id>-guided.md` and at least one `-one-shot.md` transcript
- `starter/results/sb-cli/` — evaluation artifacts
- `report.pdf` — self-contained project report
