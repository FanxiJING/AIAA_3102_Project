# Verification and Comparative Analysis

## Verification status

| Instance | Reproduced before fix | Focused regression | Public module after fix | Patch applies at base | `sb-cli` |
|---|---|---|---|---|---|
| `django__django-16485` | Yes: invalid Decimal context precision | Pass after fix | 10/10 test methods passed | Yes | Not run |
| `django__django-14580` | Yes: expression/import-set mismatch | Pass after fix | 50/50 tests passed | Yes | Not run |
| `sphinx-doc__sphinx-8595` |  |  |  |  |  |
| `pylint-dev__pylint-7080` |  |  |  |  |  |
| `pytest-dev__pytest-7571` |  |  |  |  |  |

The Django results above are local evidence only. They must not be presented as official SWE-bench results until `sb-cli` or the course evaluation path has actually completed.

## Guided-run observations

The guided workflow used short feedback cycles: confirm the assigned commit, reproduce before editing, inspect the exact computation or serialization contract, add a focused regression, observe it fail, implement a minimal patch, run the nearby public module, review the diff, and apply the exported patch to a clean worktree. This process was particularly valuable for `django__django-14580`, where the first regression test passed for the wrong reason. Reviewing the helper implementation exposed leakage from the test module's global namespace, and the assertion was changed to inspect the serializer's explicit return value.

## One-shot versus guided comparison

No independent one-shot run has yet been completed or supplied for these Django instances. This section is deliberately left incomplete rather than reconstructing a baseline after seeing the solution.

| Dimension | One-shot baseline | Guided Django run |
|---|---|---|
| Prompt/context | _Not yet supplied_ | Issue text, assigned checkout, local source, public tests, and observed command output |
| Reproduction before patch | _Not yet supplied_ | Required and recorded |
| Test review | _Not yet supplied_ | Detected a false-negative regression test in task 14580 |
| Patch review | _Not yet supplied_ | Minimal source changes plus focused tests; clean diff and apply checks |
| Outcome | _Not yet supplied_ | Both public modules pass locally |

The team must either add a genuine independent one-shot transcript for one task or merge a comparison performed by another teammate.

## Cross-task observations

Both Django defects are failures of internal invariants rather than large algorithmic errors. In `floatformat()`, a derived value crossed the valid lower boundary of an external API (`Context.prec >= 1`). In migration serialization, two halves of a return contract became inconsistent: the generated expression referenced a name that the accompanying import set did not provide. In both cases, the patch is small, but confidence depends on tracing the surrounding subsystem and designing a test that observes the correct contract.

The tasks also show why a passing pre-existing suite is insufficient. The 16485 module passed because it did not cover exponent-bearing zero strings at precision zero. The first new 14580 test also passed because its execution environment supplied the missing name accidentally. A useful regression must fail for the intended mechanism before the implementation is changed.

## Difficulties and solutions

### 1. Reproducing historical Django versions on the available machine

The system Python was 3.5.2, while another available environment used Python 3.12.13. A dedicated Python 3.10.20 Conda environment was created from `conda-forge`. Two independent Django checkouts were used because the tasks require different base commits. Import paths and commit hashes were recorded before testing.

### 2. Distinguishing the bug from an unconfigured standalone Django script

After the 16485 precision fix, an ad-hoc script progressed past `quantize()` and then raised `ImproperlyConfigured` because localization accessed settings. This was not treated as a regression in the patch. The official Django test runner supplies settings, and the focused and full filter tests passed there. The distinction prevented an environment error from being confused with the original defect.

### 3. A regression test that passed despite the defect

For 14580, `assertSerializedEqual()` evaluated code with the test module's globals, which already contained `models`. The missing import was therefore hidden. Reading the helper revealed the contamination. The final test uses `assertSerializedResultEqual()` to compare the expression and import set directly, and it fails against the unpatched base for the intended reason.

### 4. Keeping generated artifacts out of patches

Django's test runner created temporary SQLite databases. `git status`, `git diff --check`, and explicit cleanup were used so the exported patches contained only the implementation and regression-test files. Each patch was then applied in a clean worktree at the assigned commit.

## AI usage declaration

An AI coding assistant was used interactively for task decomposition, environment troubleshooting, source localization, candidate patch reasoning, test design, command review, and preparation of these written artifacts. The student executed the commands and supplied the actual outputs. AI suggestions were checked against the assigned repository revisions and public tests. A notable AI-assisted test suggestion for 14580 was not accepted merely because it passed; the helper implementation was inspected, the false negative was identified, and the test was redesigned to observe the import-set contract directly. No SWE-bench gold patch, hidden test patch, upstream fixing commit, original fixing pull request, public solution repository, or another team's transcript was consulted.

## Remaining work

- Run the course `sb-cli` path for the two Django predictions if credentials and infrastructure are available.
- Add a genuine one-shot-versus-guided comparison, either for a Django task or from the teammate responsible for the team baseline.
- Merge the Sphinx, Pylint, and pytest sections without overwriting the completed Django material.
- Incorporate these results into the unified report and presentation.
