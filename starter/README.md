# Topic D Starter Package

This package intentionally contains no helper scripts. It provides the assigned SWE-bench Verified metadata,
issue statements, public-test suggestions, and submission expectations. You are expected to clone the upstream
repositories, create patches, run local checks, and package your own predictions.

The package does not include hidden tests, gold patches, external repository checkouts, or `sb-cli`
credentials.

## Files

- `tasks.csv`: assigned instances, repositories, base commits, issue statement paths, and metadata.
- `tasks_throwaway.csv`: optional practice instance.
- `problem_statements/`: provided issue text for each instance.
- `public_tests.csv`: suggested existing tests or nearby modules to inspect.
- `SELF_CHECK.md`: guidance for checking your work without using hidden materials.
- `analysis/README.md` and `logs/README.md`: expected analysis and transcript artifacts.

## Student Responsibilities

For each assigned `instance_id`:

1. Clone the listed repository yourself.
2. Check out the exact `base_commit` from `tasks.csv`.
3. Read only the provided issue statement and allowed repository materials.
4. Reproduce the issue with your own script, command, or local test.
5. Develop and review a patch.
6. Save the patch as `patches/<instance_id>.patch`.
7. Save transcripts and analysis notes.
8. Build your own SWE-bench prediction file.

## Prediction Format

If submitting through `sb-cli`, create a JSON prediction file with one object per instance:

```json
[
  {
    "instance_id": "django__django-16485",
    "model_patch": "diff --git ..."
  }
]
```

Some local tools use JSONL with the same fields. Follow the exact format requested by the course submission
path you use.

## Required Patch Names

- `patches/django__django-16485.patch`
- `patches/django__django-14580.patch`
- `patches/sphinx-doc__sphinx-8595.patch`
- `patches/pylint-dev__pylint-7080.patch`
- `patches/pytest-dev__pytest-7571.patch`

## Academic Integrity

Allowed sources are the assigned repository at `base_commit`, documentation available from that checkout,
public API documentation needed to understand behavior, and the provided issue text.

Do not read or copy SWE-bench gold patches, hidden `test_patch` files, original upstream PRs, fixing commits,
public solution repositories, or another team's transcript or patch.
