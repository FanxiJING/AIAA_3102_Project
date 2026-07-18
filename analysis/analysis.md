# Analysis

## Verification Table

| Instance | Patch | Reproduced | Root Cause Found | Local Test | `git apply --check` | Notes |
|---|---|---|---|---|---|---|
| `pytest-dev__pytest-7571` | ✅ | ✅ `handler.level = 42` leaked | ✅ `_finalize` missing handler level restore | ✅ `test_change_level` passed | ✅ | `test_change_level_undo` failed due to Python 3.12 / pytest 6.0 AST incompatibility (unrelated) |
| `django__django-16485` | ✅ | ✅ both supplied inputs raised invalid `Context(prec=0)` | ✅ derived precision violated `decimal.Context` minimum | ✅ F2P 1/1, P2P 9/9, public module 10/10 | ✅ | Original public baseline 10/10; official `sb-cli` not run |
| `django__django-14580` | ✅ | ✅ isolated generated migration raised `NameError` | ✅ `TypeSerializer` emitted `models.Model` without its required import | ✅ focused 1/1, writer 50/50, migrations 579 with 1 skip | ✅ | Test-only old-source check failed as expected; official `sb-cli` not run |
| `sphinx-doc__sphinx-8595` | ✅ | ✅ `not []` = `True` confirmed | ✅ truthiness check conflates `None` and `[]` | ✅ Manual trace + logic verification | ✅ | FAIL_TO_PASS test (`test_empty_all`) not present at base_commit; verified via reproduction script |
| `pylint-dev__pylint-7080` | ✅ | ✅ Windows-style `src\\gen\\about.py` was not matched by `^src/gen/.*$` before the fix | ✅ raw platform separators reached the full-path regex matcher | ✅ focused reproduction plus `test_ignore_path_recursive` and `test_ignore_recursive`: 3 passed | ✅ | The assigned FAIL_TO_PASS metadata name `test_ignore_path_recursive_current_dir` is absent from the base checkout; the focused reproduction follows the starter fallback guidance |

## Guided vs. One-Shot Comparison

### Instance 1: pytest-dev__pytest-7571

Both approaches produced a functionally correct patch (7 lines, single file). The guided run took ~1 hour with step-by-step interrogation, reproduction scripting, and intermediate artifacts (`orientation.md`, `root-cause.md`); the one-shot run took ~4 minutes from a single prompt. Key differences:

1. **Depth**: One-shot identified the missing handler-level restore in `_finalize` but did not independently explain *why* restoration is needed — the session-scoped handler vs. per-test fixture mismatch. Guided follow-up questions surfaced this mechanism.
2. **Checkpoints**: Guided produced a reproduction script and traced data flow before writing any code, enabling precise debugging had the patch failed. One-shot offered no intermediate verification.
3. **Artifact ownership**: Guided artifacts reflect our own synthesized understanding after challenging the agent's claims; one-shot output is raw agent prose.
4. **Constraint adherence**: The one-shot agent autonomously searched the web for "pytest-dev__pytest-7571 fix," retrieved the upstream PR and fix commit, and used this information — violating the project's prohibition on consulting upstream fixes. The guided run's human-in-the-loop enforcement prevented this. This incident exposes a structural weakness of one-shot use in assessed work: the LLM optimizes for task completion, not rule compliance, and constraints in project documentation are invisible unless a human enforces them at each decision point. We retain this in the transcript as an honest data point.

For a self-contained bug like this one, one-shot sufficed for correctness — but we expect the guided method's advantages to grow with task complexity (e.g., Django's migration serializer or Pylint's recursive ignore-path handling, where logic spans multiple files).

### Instance 2: django__django-14580

`django__django-14580` now provides a same-task comparison. The zero-orchestration one-shot ran in a new Codex session with no 16485 conversation context. Its single prompt supplied the instance, checkout, base commit, interpreter, issue path, allowed-source limits, and desired final outcomes, but no root-cause hint or follow-up guidance. It autonomously found the correct serializer-layer patch, but added the source and regression test together, inherited a wrong `PYTHONPATH` for its first target command, and did not record the unmodified public-module baseline.

The later guided review did not replace the working patch merely to create a different answer. It treated the one-shot diff as a candidate to audit. Guidance explicitly required a clean-base public baseline, a test-only old-source failure, correct checkout isolation, focused and nearby public tests, broader migrations coverage, isolated generated-source execution, patch applicability, and patch-id consistency. No production-code change beyond the one-shot's one-line fix was necessary.

| Dimension | One-shot `django__django-14580` | Guided review `django__django-14580` |
|---|---|---|
| User requests | 1 autonomous implementation prompt | 1 explicit review/correction request in the continuing main session |
| Measured interval | `00:09:28.620` | `00:06:54.548` through verification cutoff |
| Recorded tokens | `767,166` total | `1,674,434` incremental (`1,518,848` cached input) |
| Tool calls | 18 | 11 through verification cutoff |
| Root cause | Correctly localized to `TypeSerializer` | Rechecked through serializer/writer import contract |
| Old-code public baseline | Not recorded | 49/49 passed in a detached base worktree |
| Old-code focused test | Not run separately | Test-only diff failed 1/1 with the expected empty import set |
| Environment isolation | First target command used inherited wrong `PYTHONPATH`; then corrected | Assigned checkout pinned before accepted test runs |
| Post-fix target | 1/1 passed | 1/1 passed |
| Post-fix public module | 50/50 passed | 50/50 passed |
| Broader suite | 579 passed, 1 skipped | 579 passed, 1 skipped |
| Patch review | Minimal diff and local tests | Also apply check, stable patch-id equality, isolated `exec()`, and no unrelated files |
| Production outcome | Correct one-line fix | Same fix accepted; no extra production change needed |
| Official outcome | `sb-cli` not run | `sb-cli` still not run |

The same-task result is more defensible than comparing two different bugs. The one-shot was efficient and technically correct, but guidance improved evidence quality rather than code size: it supplied the missing red/green contract proof, the original public baseline, and stronger environment and packaging checks. Raw token totals are not a clean efficiency comparison because the guided review occurred in a long continuing session and `1,518,848` of its `1,670,833` input-token increment was cached context.

The earlier 16485 conversation remains a separate example of multi-turn guided development. It is not needed to manufacture the formal one-shot comparison now that 14580 has both an autonomous run and a guided audit.

## Cross-Task Observations

All five bugs share a common shape: each is a failure at a boundary where two components individually behave correctly but their contract is violated. In django-16485, valid arithmetic produces a precision that `decimal.Context` rejects (`prec >= 1`). In django-14580, `TypeSerializer` emits a correct expression but an empty import list — the two halves of a return value diverged. In Sphinx, Python truthiness (`not []` is `True`) collapses three semantically distinct `__all__` states into two. In pytest, a session-scoped handler and a per-test fixture have mismatched lifecycles: `set_level` mutates shared state, but `_finalize` restores only local state. In Pylint, `os.walk()` produces correct paths that lose their meaning when Windows separators meet a POSIX-convention regex. None are algorithmic errors; each is an invariant that was assumed but never enforced at the boundary where it mattered.

This pattern has two practical consequences. First, patches are small (1–7 lines across all five tasks) but localization requires tracing data flow across modules, not just grepping for the crash site — the symptom and the fix live at different points in the call chain. Second, a passing test is not automatically useful evidence. The original django-16485 module passed 10/10 because the exact zero edge case was uncovered by existing tests; the first django-14580 regression passed on old code because the test module's own globals accidentally supplied the missing import. A regression that does not fail before the fix for the right reason provides no causal evidence — it only documents already-working behavior.

## Difficulties and Solutions

### Difficulty 1: Reconstructing compatible environments for historical codebases

Three tasks were initially blocked by incompatibilities between historical code and the available modern environment. For pytest, the assigned revision is 6.0.0rc2, while the available interpreter was Python 3.12 and the system installation initially exposed pytest 7.4.4. The version mixture first caused `ImportError: cannot import name 'Testdir' from '_pytest.pytester'`; installing the assigned checkout in editable mode selected the repository's own pytest implementation. Its assertion rewriter then failed during collection with `TypeError: required field "lineno" missing from alias`, because this historical pytest release predates Python 3.12's AST behavior. Running the focused reproduction with `--assert=plain` bypassed assertion rewriting and verified the handler-level repair. The public `test_change_level_undo` path launches a nested pytest process without that flag and therefore remained blocked at collection; this is reported as an environment limitation rather than a product-test pass. For Django, the available system interpreters were Python 3.5.2 and 3.12.13 — neither a suitable environment for the two assigned revisions, whose classifiers target 3.10–3.11. A dedicated Python 3.10.20 Conda environment was created from `conda-forge`, and each Django instance used an independent checkout at its exact base commit. This isolation proved essential when the first django-14580 targeted test command inherited a `PYTHONPATH` pointing at the separate django-16485 checkout and failed before collection with `ImportError: cannot import name 'default_test_processes'`; the traceback exposed the wrong absolute path, and subsequent commands explicitly pinned `PYTHONPATH` to the correct repository before results were accepted. For Sphinx, the assigned 3.x checkout imports `environmentfilter` from Jinja2, but that symbol was removed in Jinja2 3.1. The system environment contained Jinja2 3.1.4, so importing Sphinx failed with `ImportError: cannot import name 'environmentfilter' from 'jinja2'` before the autodoc behavior could be reproduced. Pinning the historical dependency with `pip install "jinja2<3.1"` installed Jinja2 3.0.3 and restored the API expected by this Sphinx revision; this dependency correction was kept separate from the one-line autodoc logic patch.

### Difficulty 2: Making standalone reproductions diagnostically faithful

The first django-16485 reproduction caught `Exception` and printed only its type and message — enough to confirm the symptom but not to localize the cause. A second run used `traceback.print_exc()`, which pinned the failure to `defaultfilters.py:190`. A separate diagnostic then printed `d.as_tuple()`, `m`, `units`, and `prec`, proving that both `"0.00"` and `Decimal("0.00")` drive the calculated precision to zero. After the quantization bug was repaired, the same ad-hoc reproduction progressed past the original crash site and raised `ImproperlyConfigured`: Django's settings had never been initialized because the earlier `ValueError` stopped execution before localization code was reached. This was not a new product defect but a consequence of using a bare script rather than the configured test runner. The progression from symptom confirmation to traceback capture to intermediate-value inspection removed one layer of uncertainty at each step, while the later settings failure demonstrated why reproduction-environment errors must be separated from the reported bug. The fix was therefore verified authoritatively through Django's `runtests.py`, where the full settings infrastructure was available and both focused and module-level tests passed.

### Difficulty 3: Restoring test-first evidence after a combined AI edit

When asked to fix django-16485, the AI inserted both the production change and the regression assertions in a single edit before demonstrating that the new test failed on the old code. The user explicitly required regression-first development. The process was redone from a clean old-code state: add only the assertions, observe `test_zero_values` fail with the expected `ValueError` (exit code 1), apply only the one-line source change, then rerun the target and nearby public tests. The final patch was unchanged — the same one-line fix and two assertions — but the corrected sequence supplied the causal evidence that a combined edit cannot provide. This was the most instructive difficulty of the project: the AI optimized for task completion, not evidentiary rigor, and the constraint that mattered most existed only in the human's process requirements, not in the code.

### Difficulty 4: Replacing an unavailable FAIL_TO_PASS test without using hidden material

The Pylint metadata names `TestRunTC.test_ignore_path_recursive_current_dir` as FAIL_TO_PASS, but that method is not present in the assigned base checkout. Searching for the instructor-only test patch would have violated the starter rules. Instead, the missing-test fact was recorded and a focused reproduction was written from the issue statement: a Windows-style path such as `src\\gen\\about.py` was checked against the POSIX-style regular expression `^src/gen/.*$`. The reproduction demonstrated the failure before normalization and passed after the fix, and it was then run with the two student-visible recursive-ignore tests from `public_tests.csv`, producing three passing tests.

## AI Usage Declaration

AI suggestions were treated as hypotheses and checked against the assigned source checkout and local tests. No API or dollar cost is reported where the local records do not provide it. All claims below are supported by the submitted transcripts and task records.

### Usage summary

| Workstream | Model | Tokens | Turns | Runtime | Failures |
|---|---|---|---|---|---|
| pytest guided | `deepseek-v4-pro` | ~134k * | — | ~1 h | — |
| pytest one-shot | `deepseek-v4-pro` | ~71.8k * | 1 | ~4 min | 1 (constraint violation: web-searched upstream fix) |
| sphinx guided | `deepseek-v4-pro` | ~85.8k * | — | ~40 min | — |
| django-16485 guided | `gpt-5.6-sol` | 4,699,950 | 11 | 1 h 15 min | 6 (wrong Python, wrong Conda, hidden traceback, non-test-first edit, SQLite artifacts, overly detailed one-shot prompt) |
| django-14580 one-shot | `gpt-5.6-sol` | 767,166 | 1 | 9 min 29 s | 1 (inherited wrong `PYTHONPATH`) |
| django-14580 guided review | `gpt-5.6-sol` | 1,674,434 ‡ | 1 | 6 min 55 s | 1 (PowerShell pipe failed to apply test patch) |
| pylint guided | `gpt-5.5` | 6,095,563 § | — § | — § | 1 (`test_ignore_path_recursive_current_dir` absent from checkout) |

\* Claude Code context-window snapshot; not a full input+output count.  
‡ Incremental: the guided review ran in a continuing session; 1,518,848 of 1,670,833 input tokens were cached from prior turns.    
§ Session-level figures from a continuing Pylint workstream, not an isolated per-task run; turns and runtime not separable.

### Constraint-compliance disclosure

No task consulted upstream fixing commits, pull requests, SWE-bench gold patches, or hidden tests. Two exceptions are disclosed: the pytest one-shot transcript records that the agent inspected Git history and retrieved the upstream PR despite the intended restriction — the LLM optimized for task completion rather than rule compliance. The django-16485 guided transcript documents an accidental `git log` inspection that did not reveal or supply the fix. The final team report must not claim that every task followed an identical tool policy.

### Workflow notes

**Claude Code (pytest, sphinx).** Claude Code with `deepseek-v4-pro` was used for pytest and Sphinx in both guided and one-shot modes. Token figures are context-window snapshots from `/context`, not full input+output counts; turns and runtime are estimated from the Guided vs. One-Shot section above. Transcripts are in `logs/pytest-dev__pytest-7571-guided.md`, `logs/pytest-dev__pytest-7571-one-shot.md`, and `logs/sphinx-doc__sphinx-8595-guided.md`. No structured usage record exists for the Sphinx one-shot run.

**OpenAI Codex (django-16485, django-14580).** OpenAI Codex model `gpt-5.6-sol` with `xhigh` reasoning effort (CLI version `0.144.5`) was used for both Django instances. Figures are from local Codex rollout records (`%USERPROFILE%\.codex\sessions\...`), with cutoffs chosen to exclude the measurement turn itself. The django-16485 guided run spanned 11 turns; the AI made six correctable mistakes (wrong Python recommendation, wrong Conda assumption, hidden traceback, non-test-first edit, SQLite artifacts, overly detailed one-shot prompt), each corrected by the user before patch acceptance. The django-14580 one-shot run completed in a single autonomous turn; the later guided review supplied test-only red-check evidence and environment isolation that the one-shot omitted. Neither instance used web search, git history to discover a fix, upstream sources, or hidden tests. Detailed rollout records are preserved in the local Codex session files referenced above.

**OpenAI Codex (pylint).** OpenAI Codex model `gpt-5.5` with `medium` reasoning effort was used in a continuing guided workflow. The session-level token figure (6,095,563) covers the full workstream and is not separable into per-task turns or runtime. The main correction was recognizing that the named FAIL_TO_PASS test (`test_ignore_path_recursive_current_dir`) was absent from the base checkout and using a student-written reproduction instead, per the starter fallback guidance.

**Verification.** Across all instances, AI output was treated as hypothesis, not authority. Every patch was checked against the assigned base checkout: the issue was reproduced independently, the regression test was confirmed to fail on old code before the fix was accepted, PASS_TO_PASS and nearby public tests were run, and `git apply --check` plus diff consistency checks were applied. No official `sb-cli` result is claimed where it has not been run.
