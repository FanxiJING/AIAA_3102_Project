# AIAA 3102 — Topic C: SWE-bench Codebase Adoption

HKUST(GZ) AIAA 3102 course project: fix five real bugs in mature Python open-source libraries,
verify patches through the SWE-bench harness, and write analysis explaining the reasoning behind
each fix.

## Project Overview

This project involves developing patches for five historical bugs from well-known Python libraries:

| # | Instance ID | Library | Bug Summary | Difficulty |
| --- | --- | --- | --- | --- |
| 1 | `django__django-16485` | Django | `floatformat('0.00', 0)` raises `ValueError` | 15 min – 1 h |
| 2 | `django__django-14580` | Django | Generated migration references `models.Model` without importing `models` | <15 min |
| 3 | `sphinx-doc__sphinx-8595` | Sphinx | Empty `__all__ = []` is ignored; autodoc documents all members | <15 min |
| 4 | `pylint-dev__pylint-7080` | Pylint | `--recursive=y` ignores `ignore-paths` | 15 min – 1 h |
| 5 | `pytest-dev__pytest-7571` | Pytest | `caplog.set_level()` is not restored after a test | 15 min – 1 h |

There is also one optional practice instance: `astropy__astropy-12907`.

## Repository Structure

```text
.
├── README.md                 # This file — project overview
├── CLAUDE.md                 # Claude Code guidance
├── report.pdf                # Final project report
├── topic-c-handout.md        # Course project handout (English)
├── topic-c-handout-zh.md     # Course project handout (Chinese)
├── topic-c-handout.pdf       # Course project handout (PDF)
├── patches/                  # Patch files & predictions
│   ├── django__django-16485.patch
│   ├── django__django-14580.patch
│   ├── sphinx-doc__sphinx-8595.patch
│   ├── pylint-dev__pylint-7080.patch
│   ├── pytest-dev__pytest-7571.patch
│   └── predictions.jsonl
├── analysis/                 # Analysis documents
│   ├── orientation.md        # Where the relevant logic lives per task
│   ├── root-cause.md         # Mechanism-level bug explanations
│   └── analysis.md           # Verification table, comparisons, difficulties, AI declaration
├── logs/                     # AI interaction transcripts
│   ├── django__django-16485-guided.md
│   ├── django__django-14580-guided.md
│   ├── django__django-14580-one-shot.md
│   ├── sphinx-doc__sphinx-8595-guided.md
│   ├── pylint-dev__pylint-7080-guided.md
│   ├── pytest-dev__pytest-7571-guided.md
│   └── pytest-dev__pytest-7571-one-shot.md
├── results/sb-cli/           # SWE-bench evaluation artifacts
├── starter/                  # Course-provided starter package
│   ├── README.md             # Starter instructions & academic integrity rules
│   ├── tasks.csv             # Task metadata (base_commit, test lists, etc.)
│   ├── tasks_throwaway.csv   # Optional practice task
│   ├── public_tests.csv      # Suggested local test commands
│   ├── SELF_CHECK.md         # Local self-check guide
│   └── problem_statements/   # Issue texts (one .md per instance)
└── repos/                    # Upstream repo clones (gitignored)
    ├── pytest/               # pytest-dev/pytest
    └── sphinx/               # sphinx-doc/sphinx
```

## Current Progress

| Instance | Reproduce | Patch | Local Tests | sb-cli | Analysis |
| --- | --- | --- | --- | --- | --- |
| django__django-16485 | ✅ | ✅ | ✅ | ✅ | ✅ |
| django__django-14580 | ✅ | ✅ | ✅ | ✅ | ✅ |
| sphinx-doc__sphinx-8595 | ✅ | ✅ | ✅ | ✅ | ✅ |
| pylint-dev__pylint-7080 | ✅ | ✅ | ✅ | ✅ | ✅ |
| pytest-dev__pytest-7571 | ✅ | ✅ | ✅ | ✅ | ✅ |

## Deliverables Checklist

- [✅️] `patches/<instance_id>.patch` ×5
- [✅️] `patches/predictions.jsonl`
- [✅️] `analysis/orientation.md`
- [✅️] `analysis/root-cause.md`
- [✅️] `analysis/analysis.md`
- [✅️] `logs/<instance_id>-guided.md` ×5
- [✅️] `logs/<instance_id>-one-shot.md` ×2 (django-14580, pytest-7571)
- [✅️] `results/sb-cli/` evaluation artifacts
- [✅️] `report.pdf`

## Academic Integrity

- **Allowed**: source code at base_commit, project docs from that checkout, public API docs, provided issue text, AI assistance (with verification)
- **Forbidden**: SWE-bench gold patches, hidden `test_patch` files, upstream fixing PRs/commits, web searches combining instance_id with "fix"/"patch", copying another team's work
- Overlap with the real fix is fine as long as you can explain your derivation trail

## Resources

- [SWE-bench](https://www.swebench.com/) — official benchmark site
- Course handouts and submission requirements: see `topic-c-handout.pdf`
