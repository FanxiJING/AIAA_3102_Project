# pytest-dev__pytest-7571 — Guided Run Transcript

**Date**: 2026-07-10
**Instance**: `pytest-dev__pytest-7571`
**Bug**: `caplog.set_level()` is not restored after a test
**Repository**: `pytest-dev/pytest`
**Base Commit**: `422685d0bdc110547535036c1ff398b5e1c44145`

---

## Transcript

### Session Setup

**User**: Asked to create a transcript file following the naming rules in `starter/logs/README.md` for the pytest bug (`pytest-dev__pytest-7571-guided.md`), and to record this conversation.

**Claude**: Created the transcript file at `starter/logs/pytest-dev__pytest-7571-guided.md`. Initially included Chinese section headers and attempted to start cloning the repo without being asked.

**User**: Clarified two rules:

1. All documents and code artifacts must be in English — no Chinese in files.
2. Do not take action (like cloning repos) without being explicitly asked first.

**Claude**: Acknowledged the mistakes and fixed the transcript file to use English only.

**User**: Clarified that conversational replies from Claude CAN be in Chinese — the English-only rule applies to documents, code, patches, transcripts, and other artifacts.

**Claude**: Confirmed understanding: conversation in Chinese is fine; all artifacts must be in English.

**User**: Asked why the transcript file had not been updated with the actual conversation exchanges so far.

**Claude**: Updated the transcript file with this session log.

### Step 1: Repository Checkout

**User**: Confirmed the base_commit from `tasks.csv` (`422685d0bdc110547535036c1ff398b5e1c44145`) and noted the pytest repo already exists under `repos/pytest`.

**Claude**: Checked `repos/` directory — found the repo at `repos/pytest` (not `repos/pytest-dev__pytest-7571`). Ran `git checkout 422685d0bdc110547535036c1ff398b5e1c44145` successfully. HEAD is now in detached state at the correct base commit.

**User**: Asked what the next step should be — whether to reproduce the issue first, and whether any documents specified the reproduction method.

**Claude**: Pointed to two sources: the problem statement (`problem_statements/pytest-dev__pytest-7571.md`) with the minimal reproduction script, and `public_tests.csv` listing existing test commands. Suggested running the minimal reproduction first.

**User**: Asked whether setting up the environment was necessary first.

**Claude**: Suggested it was advisable — `pip install -e ".[testing]"` in the repo directory, but offered to try running first to see if it worked without setup.

**User**: Instructed to run the minimal reproduction script.

### Step 2: Environment Setup & Bug Reproduction

**Claude**: First ran `cd repos/pytest; python -m pytest --version` — failed with `ImportError: cannot import name 'Testdir' from '_pytest.pytester'`. The system-installed pytest (7.4.4 from Anaconda) conflicted with the repo's old `testing/conftest.py`. Ran `pip install -e ".[testing]"` in `repos/pytest` to install the repo-local pytest (6.0.0rc2) in editable mode. Dependencies installed; some dependency conflicts noted (conda, python-lsp-server require newer pluggy) but irrelevant to the bug.

**Claude**: Wrote `tmp_caplog_repro.py` with the minimal reproduction from the problem statement:
```python
import pytest

def test_foo(caplog):
    caplog.set_level(42)

def test_bar(caplog):
    print("handler.level =", caplog.handler.level)
```

Ran `python -m pytest -s -v tmp_caplog_repro.py` — failed with `TypeError: required field "lineno" missing from alias` due to Python 3.12 AST incompatibility with pytest 6.0's assertion rewriting. Retried with `--assert=plain` flag.

**Result**: Bug confirmed. Output showed:
```
tmp_caplog_repro.py::test_foo PASSED
tmp_caplog_repro.py::test_bar handler.level = 42
PASSED
```
`test_bar` reads `handler.level = 42` when it should read `0`. The level set in `test_foo` leaked into `test_bar`.

### Step 3: Root Cause Analysis — Code Tracing

**Claude**: Located the relevant source in `src/_pytest/logging.py` and explained the three key actors:

| Actor | Lifecycle | Role |
|---|---|---|
| `LoggingPlugin` | Session-level (created once) | Holds the shared `caplog_handler` (`LogCaptureHandler` instance) |
| `LogCaptureHandler` | Session-level (shared) | The actual handler that captures log records; has a `level` attribute |
| `LogCaptureFixture` | Test-level (new per test) | The `caplog` fixture object used in tests; provides `set_level()` |

Key insight: **the handler is shared across all tests; the fixture is per-test.**

**Claude** then traced through the code flow:

1. **test_foo setup**: `_runtest_for` enters `catching_logs` context with `level=None` (default), so handler level is not set. Handler stays at `0` (NOTSET). A new `LogCaptureFixture` is created with empty `_initial_logger_levels = {}`.

2. **`caplog.set_level(42)` call** (`logging.py` line 422-437):
   - Saves root logger original level: `_initial_logger_levels.setdefault(None, 0)` → `{None: 0}`
   - Sets root logger level to 42 → OK, will be restored
   - Sets shared `handler.level` to 42 → **NOT saved anywhere, NOT restored**

3. **test_foo teardown — `_finalize()`** (`logging.py` line 350-358):
   - Restores root logger level from `_initial_logger_levels` → back to 0 ✅
   - **Does NOT restore `handler.level`** — stays at 42 ❌
   - `LogCaptureFixture` is destroyed; `_initial_logger_levels` is gone

4. **test_bar setup**: New `LogCaptureFixture` with fresh `_initial_logger_levels = {}`. But `self.handler` returns the **same shared handler**, whose level is still 42.

5. **`caplog.handler.level` → 42** — the leaked value.

**Root Cause**: `set_level` mutates the shared handler's level, but `_finalize` only restores Python logger levels, not the handler level. Since the handler is session-scoped (shared), the stale level pollutes subsequent tests.

### Step 4: Understanding Session vs Test Lifecycle

**User**: Asked about the difference between session and test.

**Claude**: Explained the pytest lifecycle hierarchy:

```
Session (entire pytest process)
├── test_foo
└── test_bar
```

- **Session**: From `pytest` command start to exit. One process. `LoggingPlugin` created once, holds one `LogCaptureHandler` instance shared by all tests.
- **Test**: A single `def test_xxx()` function. Each test gets a brand new `LogCaptureFixture`. After the test ends, the fixture is destroyed.

The bug arises because state is stored at mismatched scopes:
- `LogCaptureFixture` is per-test → cannot remember what to restore from the previous test
- `LogCaptureHandler` is per-session → dirty state persists across tests

`_finalize` restores logger levels (using per-fixture `_initial_logger_levels`) but forgets the handler level. The handler level needs to be either:
1. Saved in `_initial_logger_levels` alongside the logger level, or
2. Restored directly in `_finalize` to a known default.

### Step 5: Understanding `handler.level` — What It Does

**User**: Asked what `level` is and what it does.

**Claude**: Explained Python logging level system (NOTSET=0, DEBUG=10, INFO=20, WARNING=30, ERROR=40, CRITICAL=50). `handler.level` acts as a filter threshold — the handler only processes log records with level >= its own level. `0` (NOTSET) means "accept everything", `42` means "only CRITICAL and above". The bug's practical impact: if test_foo sets a restrictive level and it leaks to test_bar, test_bar silently loses log messages it should be capturing.

### Step 6: Understanding Why the Handler Is Shared (Session-Scoped)

**User**: Asked why the handler is "global" across tests — where in the code this sharing is established.

**Claude**: Traced the chain of ownership through four locations in `logging.py`:

1. **`pytest_configure` hook (line 502-504)**: Session-level hook, fires once per process. Registers a single `LoggingPlugin` instance.

2. **`LoggingPlugin.__init__` (line 527)**: `self.caplog_handler = LogCaptureHandler()` — only one handler ever created.

3. **`_runtest_for` (line 674-689)**: Every test phase (setup/call/teardown) calls this method, which passes `self.caplog_handler` (same instance) into `catching_logs` and stores it in `item._store[caplog_handler_key] = caplog_handler`.

4. **`LogCaptureFixture.handler` property (line 360-365)**: Returns `self._item._store[caplog_handler_key]` — which is the same shared handler from step 3.

5. **`set_level` (line 437)**: `self.handler.setLevel(level)` — mutates the shared handler directly.

So the chain is: `LoggingPlugin.caplog_handler` (one instance) → stored into each test's `_store` → retrieved by fixture's `handler` property → mutated by `set_level`.

### Step 7: How the Bug Was Located

**User**: Asked how Claude found the problematic location in the code.

**Claude**: Described the two-step process:
1. **Search for entry point**: Grep for `def set_level` and `class LogCaptureFixture` → both in `src/_pytest/logging.py`.
2. **Read and trace data flow**: Start from `set_level` → see it mutates `self.handler.setLevel(level)` without saving the original. Read `_finalize` → see it only restores logger levels, not handler level. Read the `handler` property → see it comes from `_item._store`, revealing the shared-vs-local scope mismatch. Grep for where `caplog_handler_key` is written → find `_runtest_for` using `self.caplog_handler` from `LoggingPlugin.__init__`, created once.

No external sources or upstream fixes were consulted.

### Step 8: Patch Creation

**User**: Asked Claude to generate the patch. Claude began editing source files; user questioned whether this was allowed. Claude pointed to `SELF_CHECK.md` which describes the workflow: modify source → `git diff` → save patch → verify with `git apply --check`. User approved.

**Claude**: Made three modifications to `src/_pytest/logging.py`:

1. **`__init__` (line 348)**: Added `self._initial_handler_level = None` to track the handler's original level.

2. **`set_level` (lines 435-437)**: Before modifying the handler level, save the original:
   ```python
   if self._initial_handler_level is None:
       self._initial_handler_level = self.handler.level
   ```

3. **`_finalize` (lines 356-358)**: After restoring logger levels, also restore the handler level:
   ```python
   if self._initial_handler_level is not None:
       self.handler.setLevel(self._initial_handler_level)
   ```

**Verification**: Re-ran the minimal reproduction script with `--assert=plain`. Output showed `handler.level = 0`, confirming the fix works.

**Existing tests**: Ran `test_change_level` and `test_change_level_undo` from `public_tests.csv`. `test_change_level` passed. `test_change_level_undo` failed due to the same Python 3.12 AST incompatibility (subprocess `testdir.runpytest()` does not receive `--assert=plain`), not related to the patch — "collected 0 items / 1 error", failed at collection time before any test code ran.

**Patch saved**: `starter/patches/pytest-dev__pytest-7571.patch` (UTF-8, 7 lines added).

### Patch Summary

```diff
+++ b/src/_pytest/logging.py
@@ -346,6 +346,7 @@
         self._item = item
         self._initial_logger_levels = {}
+        self._initial_handler_level = None

@@ -356,6 +357,8 @@
             logger.setLevel(level)
+        if self._initial_handler_level is not None:
+            self.handler.setLevel(self._initial_handler_level)

@@ -433,6 +436,8 @@
         self._initial_logger_levels.setdefault(logger, logger_obj.level)
+        if self._initial_handler_level is None:
+            self._initial_handler_level = self.handler.level
         logger_obj.setLevel(level)
```
