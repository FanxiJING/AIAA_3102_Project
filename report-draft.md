# Final Project Report — Topic C: SWE-bench Codebase Adoption

## 1. Project Problem and Goal

The central question of this project is: **can an AI-assisted developer localize and fix real bugs in unfamiliar, mature Python codebases, producing patches that are correct by mechanism — not merely by passing tests — and explain why each fix is correct?**

This matters because software maintenance in practice requires working on code you did not write, in libraries whose internals you have never studied. Issue reproduction, data-flow tracing across module boundaries, patch review, and evidence-based reasoning are the same skills needed when inheriting a legacy system or contributing to open-source. The AI adds complexity: it accelerates exploration but also introduces plausible-sounding errors that only careful verification can catch.

The concrete goal is to fix five real historical bugs from widely used Python libraries, using only the original GitHub issue text as the problem description. The gold patches, hidden tests, and upstream fixing commits are deliberately withheld — the task is to derive each fix from first principles using the source code, its documentation, and local experimentation.

| # | Instance | Library | Bug |
| --- | --- | --- | --- |
| 1 | `django__django-16485` | Django | `floatformat('0.00', 0)` raises `ValueError` |
| 2 | `django__django-14580` | Django | Migration references `models.Model` without importing `models` |
| 3 | `sphinx-doc__sphinx-8595` | Sphinx | Empty `__all__ = []` is ignored; all members documented |
| 4 | `pylint-dev__pylint-7080` | Pylint | `--recursive=y` ignores `ignore-paths` on Windows |
| 5 | `pytest-dev__pytest-7571` | Pytest | `caplog.set_level()` leaks handler state between tests |

---

## 2. Methodology

### 2.1 Overall Approach & Patch Discipline

The guiding principle was **treat every claim as a hypothesis until verified against the assigned source checkout and local test results**. This applied equally to our own reasoning and to AI-generated suggestions. Each instance progressed through five sequential phases — environment preparation, reproduction, localization, patch development, and verification — where each phase produced artifacts the next depended on.

All patches followed a **test-first, minimal-change** discipline: (1) add a regression test and confirm it fails on unmodified code, establishing that the test detects the defect rather than documenting already-working behavior; (2) identify the smallest change that makes the test pass — patches ranged from 1 to 7 lines with no refactoring or unrelated edits; (3) fix the general invariant violation, not the specific input (e.g., `max(..., 1)` rather than a zero-value special case); (4) apply test and production changes in separate, independently verified steps.

### 2.2 Environment, Reproduction & Localization

Each codebase was matched to a compatible environment (per-instance Python version, pinned dependencies, isolated checkouts) rather than forcing historical code into a modern runtime. Environment fixes were always applied and recorded separately from bug fixes.

Reproduction followed a **diagnostic staircase**: symptom confirmation → traceback to pinpoint the failure site → intermediate-value inspection to prove the mechanism → authoritative verification through the project's own test runner. When the named FAIL_TO_PASS test was absent from the base checkout, a focused reproduction was written from the issue statement.

Localization used **data-flow tracing**: forward from the user-visible entry point, backward from the failure to identify the violated assumption, then inspect the contract at the boundary where they meet. The symptom and the fix lived at different points in the call chain for all five bugs — keyword search would find the crash site but not the root cause.

### 2.3 Verification Protocol

Every patch was subjected to a seven-layer verification protocol:

| Layer | What | Purpose |
| --- | --- | --- |
| 1. Baseline | Run PASS_TO_PASS and nearby public tests on unmodified checkout | Confirm pre-existing functionality |
| 2. Red check | Run only the new regression test on old code | Prove the test detects the defect |
| 3. Green check | Apply the fix and re-run the regression test | Prove the fix resolves the defect |
| 4. Regression guard | Run the full PASS_TO_PASS set and nearby module | Prove no existing behavior was broken |
| 5. Broader suite | Run the surrounding subsystem test suite | Catch unintended side effects |
| 6. Patch integrity | `git apply --check` on the saved `.patch` | Ensure well-formed, cleanly applicable patch |
| 7. Patch-id stability | Verify saved patch-id matches working-tree diff | Ensure no unintended changes |

### 2.4 AI Workflow Design

Two AI workflows were employed. **Guided** (all five instances): multi-turn, human-in-the-loop — the user provided metadata, directed the AI to read and trace before coding, challenged inconsistent claims, and enforced process constraints. **One-shot** (pytest-7571, django-14580): a single autonomous prompt with no follow-up, designed to test whether the same quality of evidence emerges without human intervention. Note: for django-14580, the guided run was a *review* of the one-shot's patch (auditing and supplementing evidence) rather than an independent from-scratch development; for pytest-7571, the guided and one-shot runs were independent. Results are compared in Section 3.3.

---

## 3. Main Evidence and Results

All five bugs share a common shape: each is a failure at a boundary where two components individually behave correctly but their contract is violated — a precision formula and `decimal.Context`, a serializer and its import contract, a truthiness check and `__all__` semantics, a session-scoped handler and a per-test fixture, a Windows path and a POSIX regex. Patches are small (1–7 lines) but localization required tracing data flow across modules, not merely grepping for the crash site.

| Instance | Fix | Key Evidence | Test Results |
| --- | --- | --- | --- |
| `django__django-16485` | `max(..., 1)` on precision | `Decimal("0.00")` drove calculated `prec` to 0, violating `Context(prec >= 1)` | F2P 1/1, P2P 9/9, module 10/10 |
| `django__django-14580` | Add `"from django.db import models"` to `TypeSerializer` special case | `TypeSerializer` returned `("models.Model", set())` — expression without import | Focused 1/1, writer 50/50, migrations 579/1sk |
| `sphinx-doc__sphinx-8595` | `if self.__all__ is None:` replacing `if not self.__all__:` | Empty `__all__` incorrectly documented a temporary `hidden()` function before the fix; identity check separates empty from undefined | Focused red/green reproduction; public `ignore_module_all` logic passed ¹ |
| `pylint-dev__pylint-7080` | Normalize path separators before regex matching | Windows `src\\gen\\about.py` unmatched by `^src/gen/.*$`; `os.sep` → `/` fixes it | Focused repro + 2 public tests: 3/3 ¹ |
| `pytest-dev__pytest-7571` | Save/restore `_initial_handler_level` in fixture lifecycle | `handler.level = 42` leaked to next test; `_finalize` only restored logger levels | Focused red/green 2/2; `test_change_level` 1/1 ² |

¹ Hidden or named FAIL_TO_PASS tests were unavailable in the base checkout or blocked by the local test runner; verification used focused reproductions from the public issue text plus nearby public test logic.  
² `test_change_level_undo` remained blocked at collection by Python 3.12 / pytest 6.0 AST incompatibility in the nested pytest process (environment limitation).

Remote `sb-cli` evaluation was attempted after generating `patches/predictions.jsonl`:

```text
sb-cli submit swe-bench_verified test --predictions_path .\patches\predictions.jsonl --run_id my_id_9
sb-cli get-report swe-bench_verified test my_id_9 --overwrite 1
```

The saved report is `results/sb-cli/swe-bench_verified__test__my_id_9.json`. It lists all five instances as submitted, but returns `completed_instances=0`, `failed_instances=5`, `resolved_instances=0`, and `unresolved_instances=0`. We therefore do not claim a resolved/unresolved official outcome from this run; it is recorded as an attempted remote evaluation that failed before normal per-instance results were produced.

### 3.1 Guided vs. One-Shot Comparison

Two instances received both treatments:

| Dimension | django-14580 One-Shot | django-14580 Guided Review | pytest-7571 One-Shot | pytest-7571 Guided |
| --- | --- | --- | --- | --- |
| Runtime | 9 min 29 s | 6 min 55 s | ~4 min | ~1 h |
| Turns | 1 | 1 (review) | 1 | Multi-turn |
| Patch correct? | ✅ | ✅ (same patch) | ✅ | ✅ |
| Baseline recorded? | ❌ | ✅ 49/49 | ❌ | ✅ |
| Red-check proof? | ❌ | ✅ Test-only diff failed 1/1 | ❌ | ✅ |
| Environment isolation? | ❌ Wrong `PYTHONPATH` | ✅ Checkout pinned | ❌ | ✅ |
| Constraint compliance? | ✅ | ✅ | ❌ Web-searched upstream fix | ✅ |

For pytest-7571, the one-shot was dramatically faster (~4 min vs. ~1 h), consistent with the expectation that autonomous runs trade evidence for speed. For django-14580, however, the Guided Review (29 min) was faster than the One-Shot (40 min) — a counterintuitive result that reflects the difference between discovery and audit. The One-Shot had to autonomously localize the bug, generate a fix, run tests, and recover from an inherited wrong `PYTHONPATH`; the Guided Review started from a known correct patch and only needed to verify it, add missing baseline and red-check evidence, and confirm environment isolation. The Guided Review accepted the one-shot's production patch unchanged — the two columns reflect different evidence-gathering processes applied to the same code change. The pytest one-shot additionally violated the prohibition on consulting upstream fixes, demonstrating that academic integrity constraints require human enforcement at each decision point.


---

## 4. Case Analysis

### 4.1 django-16485: The Value of Diagnostic Staircase

**Why this case matters.** This was the most complete example of the diagnostic staircase method: each layer of investigation removed one level of uncertainty, and the final fix — `max(abs(p) + units + 1, 1)` — is a single function call that encodes a constraint the codebase had always assumed but never enforced.

**The reasoning chain.** The issue statement reported that `floatformat("0.00", 0)` raises `ValueError`. A bare Python script reproduced the error, but catching `Exception` and printing only its type told us *that* it failed, not *where*. Adding `traceback.print_exc()` pinned the crash to `defaultfilters.py:190` — the `d.quantize(exp, ROUND_HALF_UP, Context(prec=prec))` call. Printing `d.as_tuple()`, `m`, `units`, and the calculated `prec` then revealed the mechanism:

| Variable | Value | How |
| --- | --- | --- |
| `d` | `Decimal("0.00")` | Normalized from input |
| `d.as_tuple()` | `(sign=0, digits=(0,), exponent=-2)` | Zero with scale |
| `units` | `-1` | `len((0,)) + (-2)` |
| `prec` | `0` | `abs(0) + (-1) + 1` |

A precision of zero violates `decimal.Context`'s invariant (`prec >= 1`), and Python evaluates function arguments before calling the function — so `Context(prec=0)` raised the exception before `quantize()` ever ran. The fix was not a zero-value special case; `max(..., 1)` enforces the API's documented minimum for all inputs.

**What made it hard.** The symptom and the fix are separated by three layers of arithmetic — the crash at line 190 is a consequence of a calculation three lines earlier, which itself depends on `Decimal`'s internal representation of zero. Grepping for the error message would find the crash site but not the root cause. This distance between symptom and cause, not the complexity of any individual component, is what made localization non-trivial — and it exemplifies why data-flow tracing was necessary across all five instances.

### 4.2 pytest-7571: Lifecycle Mismatch and the One-Shot Constraint Failure

**Why this case matters.** This is the clearest illustration of the boundary-contract pattern described in Section 3: three actors with correctly implemented individual behavior, but mismatched lifecycles that cause shared state to leak. It also produced the project's most important AI governance finding.

**The reasoning chain.** The `caplog` fixture involves three actors with different scopes:

| Actor | Scope | Lifecycle |
| --- | --- | --- |
| `LogCaptureHandler` | Session | Created once in `LoggingPlugin.__init__`, shared across all tests |
| `LogCaptureFixture` | Per-test | New instance per test via the `caplog` fixture |
| `_initial_logger_levels` dict | Per-fixture | Fresh empty dict per `LogCaptureFixture.__init__` |

When `caplog.set_level(42)` is called, it mutates the session-shared handler to 42 and saves the *logger*'s original level — but never saves the *handler*'s original level. At teardown, `_finalize` restores only logger levels; the handler remains at 42. The next test's fixture receives the same session-shared handler, still at 42 instead of the expected 0 (NOTSET). The fix adds three lines: track `_initial_handler_level` in `__init__`, save it in `set_level`, restore it in `_finalize`.

**What made it hard.** The lifecycle mismatch is invisible at any single point in the code. `set_level` alone is correct (it mutates the handler and records the logger level). `_finalize` alone is correct (it restores what was recorded). Only by tracing across test boundaries — from teardown through the next setup — does the leak become visible. No individual component is wrong, but the system-level invariant is violated.

**The one-shot constraint failure.** When given the instance ID in a single autonomous prompt, the pytest one-shot agent independently searched the web for the upstream PR and used that information to produce its patch — despite the prompt explicitly prohibiting this. The patch was technically correct, but the process was not compliant. We retained this transcript as an honest data point: it shows that academic integrity constraints in project documentation are invisible to an AI agent unless a human enforces them at each decision point.

---

## 5. Difficulties and Solutions

### 5.1 Reconstructing Compatible Environments for Historical Codebases

**Difficulty.** Three of five tasks were initially blocked by incompatibilities between historical revisions and the modern runtime — pytest 6.0.0rc2 vs. Python 3.12's AST, Sphinx 3.x vs. Jinja2 3.1's removed API, Django vs. wrong Python versions. A secondary risk was cross-contamination between the two Django checkouts via inherited `PYTHONPATH`.

**Solution.** Match the environment to the codebase, not vice versa: per-instance Python versions, pinned historical dependencies, and isolated checkouts with explicit `PYTHONPATH`. Every environment fix was recorded and applied separately from the bug fix, so patches contained only the logic change. A broader lesson emerged: bare reproduction scripts are for diagnosis; the project's own test runner is for verdicts. The two can produce different failures, and distinguishing an environment artifact from a real regression requires understanding the codebase's bootstrap sequence.

### 5.2 Replacing an Unavailable FAIL_TO_PASS Test Without Using Hidden Material

**Difficulty.** The Pylint FAIL_TO_PASS test was absent from the assigned base checkout, and the hidden `test_patch` files are prohibited by the starter rules. Without a pre-existing regression, there was no off-the-shelf way to demonstrate the bug.

**Solution.** A focused reproduction was written directly from the issue statement per the starter fallback guidance: a Windows path (`src\\gen\\about.py`) checked against the POSIX-style regex (`^src/gen/.*$`) failed before normalization and passed after it. This reproduction, combined with the two student-visible recursive-ignore tests, produced three passing tests. The missing test was recorded as a limitation.

### 5.3 Restoring Test-First Evidence After a Combined AI Edit

**Difficulty.** When asked to fix django-16485, the AI inserted the production change and regression assertions in a single edit, making it impossible to observe the test fail on old code. A test never observed to fail provides no causal evidence — it could be documenting already-working behavior.

**Solution.** The process was redone from a clean checkout: add only the assertions, observe `test_zero_values` fail with exit code 1, then apply the one-line source change. The final patch was unchanged — same fix, same assertions — but the corrected sequence supplied causal evidence that a combined edit cannot provide. This was more instructive than any technical bug: AI agents optimize for task completion, not evidentiary rigor, and the constraints that matter most exist only in the human's process requirements.

---

## 6. AI Usage Declaration

### 6.1 Usage Summary

| Workstream | Model | Tokens | Turns | Runtime | Correctable Mistakes |
| --- | --- | --- | --- | --- | --- |
| pytest guided | `deepseek-v4-pro` | ~134k * | — | ~1 h | — |
| pytest one-shot | `deepseek-v4-pro` | ~71.8k * | 1 | ~4 min | 1 (constraint violation) |
| sphinx guided | `deepseek-v4-pro` | ~85.8k * | — | ~40 min | — |
| django-16485 guided | `gpt-5.6-sol` | 4,699,950 | 11 | 1 h 15 min | 6 (wrong Python, wrong Conda, hidden traceback, non-test-first edit, SQLite artifacts, overly detailed one-shot prompt) |
| django-14580 one-shot | `gpt-5.6-sol` | 767,166 | 1 | 9 min 29 s | 1 (wrong `PYTHONPATH`) |
| django-14580 guided review | `gpt-5.6-sol` | 1,674,434 ‡ | 1 | 6 min 55 s | 1 (PowerShell pipe failure on test patch) |
| pylint guided | `gpt-5.5` | 6,095,563 | 8 | ~30 min | 1 (absent FAIL_TO_PASS test) |

\* Claude Code context-window snapshot; not a full input+output count.  
‡ Incremental: 1,518,848 of 1,670,833 input tokens were cached from prior turns.

Three AI tools were used: **Claude Code** with `deepseek-v4-pro` (pytest, Sphinx), **OpenAI Codex** with `gpt-5.6-sol` at `xhigh` effort (both Django instances), and **OpenAI Codex** with `gpt-5.5` at `medium` effort (Pylint). Across all seven workstreams, the AI made 10 correctable mistakes — all caught by human verification before reaching a patch. Transcripts are in `logs/<instance_id>-guided.md` and `logs/<instance_id>-one-shot.md`.

### 6.2 Constraint Compliance

No task consulted upstream fixing commits, SWE-bench gold patches, or hidden tests — with one disclosed exception: the pytest one-shot agent autonomously web-searched the instance ID and retrieved the upstream PR (see Section 4.2). The django-16485 guided transcript also documents an accidental `git log` inspection that did not reveal the fix.

---

## 7. Discussion and Limitations

### 7.1 Evaluation Coverage & Generality

The most significant limitation is incomplete formal evaluation: `sb-cli` was attempted for all five instances in `my_id_9`, but the service report produced no completed resolved/unresolved results. Two FAIL_TO_PASS-style tests were also unavailable or blocked locally: one named Pylint test was absent from the base checkout, and pytest's nested `test_change_level_undo` was blocked by Python 3.12 / pytest 6.0 assertion-rewriter incompatibility. Local test results cannot substitute for the hidden SWE-bench test suite; reproduction scripts demonstrate mechanism but lack the coverage of a vetted test suite.

The generality of findings is also limited. All five bugs are boundary-contract violations in mature Python libraries — deterministic, reproducible, with isolated fixes. Non-deterministic bugs (race conditions, memory corruption) would require different approaches not tested here. All patches were small because each root cause was a single violated invariant — bugs requiring architectural restructuring would stress-test the methodology differently. The boundary-contract pattern itself may be an artifact of SWE-bench selection bias (the benchmark favors bugs with isolated, testable fixes) rather than a property of real-world defects in general.

### 7.2 Comparison Confounds

The guided vs. one-shot comparison has multiple structural confounds. Three different AI models were used across workstreams, entangling model capability with workflow type. Only two instances received both treatments, and django-14580's guided run was a *review* rather than independent from-scratch development. All guided runs were conducted by the same operator, so the methodology's effectiveness is partially attributable to operator skill — reproducibility across operators is unmeasured. The findings are therefore indicative rather than conclusive.

The pytest one-shot violation exposes a further structural concern: the current academic integrity safeguard relies on the human auditing the transcript and recognizing violations. An operator who does not review the transcript would submit non-compliant work without knowing it. Structural safeguards — such as logging every external information source accessed or running agents without web access — deserve consideration for future AI-assisted coursework.

### 7.3 What the Method Does Not Capture

The evidence-first methodology prioritizes correctness and causal proof over speed, token efficiency, and cost — guided runs consumed more resources than one-shot alternatives for the same patches. Whether the additional evidence justifies that cost depends on context: in production or coursework, the evidence is the point; in high-throughput triage, the guided method would be uneconomical. A tiered approach — one-shot for triage, guided for high-risk fixes — would balance both needs.

---

## 8. Conclusion

The most transferable lesson from this project is a principle, not a technique: evidence quality matters more than patch correctness in isolation. A correct patch without a red-check is indistinguishable from a test that documents already-working behavior. A correct patch without a baseline is vulnerable to hidden regressions. A correct patch produced by an AI that violated academic integrity rules is not a valid submission. In AI-assisted development, the human's role is not to write code — the AI can do that — but to enforce the evidentiary standards that give the code meaning.

---

## 9. References and Appendix

### References

1. SWE-bench: <https://www.swebench.com/>
2. Django repository: <https://github.com/django/django>
3. Sphinx repository: <https://github.com/sphinx-doc/sphinx>
4. Pylint repository: <https://github.com/pylint-dev/pylint>
5. Pytest repository: <https://github.com/pytest-dev/pytest>
