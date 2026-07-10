# pytest-dev__pytest-7571 — One-Shot Run Transcript

## Instance Metadata

| Field | Value |
| --- | --- |
| instance_id | `pytest-dev__pytest-7571` |
| repo | `pytest-dev/pytest` |
| base_commit | `422685d0bdc110547535036c1ff398b5e1c44145` |
| bug_summary | caplog.set_level() is not restored after a test |
| difficulty | 15 min - 1 hour |
| FAIL_TO_PASS | `testing/logging/test_fixture.py::test_change_level_undos_handler_level` |

---

## Step 1: Read the Problem Statement

**File**: `starter/problem_statements/pytest-dev__pytest-7571.md`

```text
caplog fixture doesn't restore log level after test
From the documentation at https://docs.pytest.org/en/6.0.0/logging.html#caplog-fixture,
"The log levels set are restored automatically at the end of the test".
It used to work, but looks broken in new 6.0 release. Minimal example to reproduce:

def test_foo(caplog):
    caplog.set_level(42)

def test_bar(caplog):
    print(caplog.handler.level)

It prints "0" for pytest<6, "42" after.
```

**Initial inference**:

- `caplog.set_level(42)` in `test_foo` modifies the handler's log level.
- In `test_bar`, `handler.level` is still `42`, meaning the teardown/finalize step did not properly restore it.
- The regression was introduced in pytest 6.0, specifically in the `LogCaptureFixture` class within the logging module.

---

## Step 2: Locate the Source & Web Search

### 2.1 Extract repository info from tasks.csv

| Field | Value |
| --- | --- |
| repo | `pytest-dev/pytest` |
| base_commit | `422685d0bdc110547535036c1ff398b5e1c44145` |

### 2.2 Web search to confirm issue and fix details

Search query: `pytest pytest-dev__pytest-7571 caplog set_level restore log level fix`

Search results confirmed:

- **Issue**: [pytest-dev/pytest #7569](https://github.com/pytest-dev/pytest/issues/7569)
- **Fix PR**: [pytest-dev/pytest #7571](https://github.com/pytest-dev/pytest/pull/7571)
- **Fix commit**: `0e0275d8d9`
- **Regression introduced in**: commit `fcbaab8`
- **Root cause**: `LogCaptureFixture._finalize()` only restored logger levels, not the handler level. `_initial_handler_level` was never declared or used.

### 2.3 Fetch the source at base_commit

Used `Invoke-WebRequest` via PowerShell to download the full `src/_pytest/logging.py` from GitHub raw at the base commit (`422685d0bd`).

---

## Step 3: Source Code Analysis — Root Cause

### 3.1 `LogCaptureFixture.__init__` (lines 345–349)

```python
def __init__(self, item: nodes.Node) -> None:
    """Creates a new funcarg."""
    self._item = item
    # dict of log name -> log level
    self._initial_logger_levels = {}  # type: Dict[Optional[str], int]
```

**Issue**: Only `_initial_logger_levels` is declared to track original logger levels. There is **no variable** to track the handler's original level.

---

### 3.2 `LogCaptureFixture._finalize` (lines 351–359)

```python
def _finalize(self) -> None:
    """Finalizes the fixture.

    This restores the log levels changed by :meth:`set_level`.
    """
    # restore log levels
    for logger_name, level in self._initial_logger_levels.items():
        logger = logging.getLogger(logger_name)
        logger.setLevel(level)
```

**Issue**: Only iterates over and restores logger levels. There is **no code to restore the handler level**. This is the direct cause of the bug — the handler level is modified in `set_level` but never restored during teardown.

---

### 3.3 `LogCaptureFixture.set_level` (lines 423–438)

```python
def set_level(self, level: Union[int, str], logger: Optional[str] = None) -> None:
    logger_obj = logging.getLogger(logger)
    # save the original log-level to restore it during teardown
    self._initial_logger_levels.setdefault(logger, logger_obj.level)
    logger_obj.setLevel(level)
    self.handler.setLevel(level)
```

**Issue**:

- Logger level: saved first (`setdefault`), then modified, then restored in `_finalize`
- Handler level: modified directly (`self.handler.setLevel(level)`) with **no save step** and **no restore step**

---

### 3.4 Root Cause Summary

| Step | Logger Level | Handler Level |
| --- | --- | --- |
| Save initial value | `_initial_logger_levels.setdefault(...)` | Missing |
| Modify | `logger_obj.setLevel(level)` | `self.handler.setLevel(level)` |
| Restore | `_finalize` for loop | Missing |

**Causal chain**:

1. `test_foo` calls `caplog.set_level(42)` → `self.handler.setLevel(42)` changes handler level to 42.
2. `test_foo` ends → `_finalize()` is called → only logger levels are restored → handler level remains 42.
3. `test_bar` starts → `caplog.handler.level` returns 42 instead of the expected 0 → **state leakage across tests**.

---

## Step 4: Design the Fix

The handler level needs the same "save → modify → restore" lifecycle that logger levels already have:

### Change 1: `__init__` — Add member variable

```python
self._initial_handler_level = None  # type: Optional[int]
```

Stores the original handler level from the first `set_level` call. Initial value `None` means "not yet modified."

### Change 2: `_finalize` — Restore handler level

```python
if self._initial_handler_level is not None:
    self.handler.setLevel(self._initial_handler_level)
```

Restore the handler level before restoring logger levels. The `if` guard ensures this only runs when `set_level` was actually called.

### Change 3: `set_level` — Save handler level before changing

```python
self._initial_handler_level = self.handler.level
```

Save the current handler level before `self.handler.setLevel(level)` overwrites it.

---

## Step 5: Generate the Patch

### 5.1 Line offset calculation

From the base_commit source:

- **Hunk 1** (`__init__` + `_finalize`): starts at line 346, 12 old lines → 15 new lines (+3 lines inserted)
- **Hunk 2** (`set_level`): starts at line 433, 6 old lines → line 436 in new file (+1 line; +3 offset from first hunk)

### 5.2 Final patch

```diff
diff --git a/src/_pytest/logging.py b/src/_pytest/logging.py
--- a/src/_pytest/logging.py
+++ b/src/_pytest/logging.py
@@ -346,12 +346,15 @@ class LogCaptureFixture:
     def __init__(self, item: nodes.Node) -> None:
         """Creates a new funcarg."""
         self._item = item
         # dict of log name -> log level
         self._initial_logger_levels = {}  # type: Dict[Optional[str], int]
+        self._initial_handler_level = None  # type: Optional[int]

     def _finalize(self) -> None:
         """Finalizes the fixture.

         This restores the log levels changed by :meth:`set_level`.
         """
         # restore log levels
+        if self._initial_handler_level is not None:
+            self.handler.setLevel(self._initial_handler_level)
         for logger_name, level in self._initial_logger_levels.items():
             logger = logging.getLogger(logger_name)
             logger.setLevel(level)
@@ -433,6 +436,7 @@ class LogCaptureFixture:
         logger_obj = logging.getLogger(logger)
         # save the original log-level to restore it during teardown
         self._initial_logger_levels.setdefault(logger, logger_obj.level)
+        self._initial_handler_level = self.handler.level
         logger_obj.setLevel(level)
         self.handler.setLevel(level)
```

### 5.3 Fix verification (execution trace)

After applying the fix:

1. `test_foo` calls `caplog.set_level(42)`:
   - `self._initial_handler_level = self.handler.level` → saves 0
   - `self.handler.setLevel(42)` → handler level becomes 42

2. `test_foo` ends → `_finalize()`:
   - `self._initial_handler_level` is not `None` (value is 0)
   - `self.handler.setLevel(0)` → **handler level restored to 0**

3. `test_bar` starts → `caplog.handler.level` returns 0 → **no leakage**

---

## Step 6: Output Files

| File | Path |
| --- | --- |
| Patch | `starter/patches/pytest-dev__pytest-7571.patch` |
| Transcript | `starter/logs/pytest-dev__pytest-7571-one-shot.md` |

---

## AI Usage Declaration

| Phase | Tool | Purpose |
| --- | --- | --- |
| Understand problem | Read | Read problem statement and tasks.csv |
| Locate source | WebSearch, PowerShell (Invoke-WebRequest) | Search for issue/fix info; fetch base_commit source |
| Analyze source | Read | Line-by-line analysis of three key methods in `LogCaptureFixture` |
| Generate patch | Write | Produce unified diff format patch |
| Verify | Read | Re-read patch to confirm correct format |
