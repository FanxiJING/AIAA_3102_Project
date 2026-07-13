# Root Cause Analysis

Mechanism-level explanation of each bug and the fix applied.

## pytest-dev__pytest-7571

### Bug Mechanism

The `caplog` fixture (`caplog.set_level()`) does not restore the log handler's level after a test ends, despite documentation stating otherwise.

**Three actors with mismatched lifecycles:**

| Actor | Scope | Created |
|---|---|---|
| `LogCaptureHandler` | Session | Once in `LoggingPlugin.__init__` |
| `LogCaptureFixture` | Per-test | New instance per test via `caplog` fixture |
| `_initial_logger_levels` dict | Per-fixture | Fresh dict per `LogCaptureFixture` |

**Trace of the leak:**

1. **test_foo calls `caplog.set_level(42)`**: The handler's `setLevel(42)` is called on the session-shared handler. The fixture's `_initial_logger_levels` saves the *logger* level (`{None: 0}`) but does not save the handler's current level.

2. **test_foo teardown — `_finalize()`**: Iterates `_initial_logger_levels`, restores root logger to 0. The handler level (42) is never restored. The fixture is then destroyed.

3. **test_bar setup**: A new `LogCaptureFixture` is created with empty `_initial_logger_levels`. Its `handler` property returns the same session-shared handler — which still has level 42.

4. **test_bar reads `caplog.handler.level`**: Returns 42 instead of the expected 0 (NOTSET).

### Fix

Three changes in `LogCaptureFixture`:

1. **`__init__`**: Add `self._initial_handler_level = None` to track the handler's original level.

2. **`set_level`**: Before mutating the handler level, save the original value under `_initial_handler_level` (only on the first call, mirroring the `setdefault` pattern used for logger levels).

3. **`_finalize`**: After restoring logger levels, also restore the handler level from `_initial_handler_level` if it was saved.

### Evidence

- Reproduction script confirmed `handler.level = 42` leaked from test_foo to test_bar before the fix.
- After the fix, `handler.level = 0` as expected.
- Existing test `test_change_level` passes.

## django__django-16485

*To be completed.*

## django__django-14580

*To be completed.*

## sphinx-doc__sphinx-8595

### Bug Mechanism

When `__all__ = []` (an empty list) is defined in a Python module, Sphinx autodoc with `:members:` still documents all public functions, instead of showing nothing. The bug is a truthiness check that fails to distinguish `None` (no `__all__` defined) from `[]` (explicitly empty `__all__`).

**Trace of the logic error:**

1. **Module defines `__all__ = []`**: The developer explicitly declares that this module has no public API. Python's own `from module import *` respects this — importing nothing.

2. **`import_object()` reads `__all__`**: `inspect.getall(self.object)` returns `[]`, which is stored in `self.__all__`. This is correct.

3. **`get_object_members()` checks the condition** (line 1077): `if not self.__all__:` evaluates `not []`, which is `True`. The code enters the branch intended for "no `__all__` defined" and returns all module members unfiltered.

4. **The else-branch (filtering) is never reached**: The code that marks members outside `__all__` as skipped is bypassed entirely.

**Three semantically distinct states collapsed into two:**

| `self.__all__` | Developer intent | `not self.__all__` | `self.__all__ is None` |
|---|---|---|---|
| `None` | "No `__all__` — use default rules" | `True` | `True` |
| `['foo', 'bar']` | "These names are public" | `False` | `False` |
| `[]` | "Nothing is public" | `True` ← bug | `False` |

The root cause is that Python truthiness treats both `None` and `[]` as falsy, but only `None` signals "undefined" — `[]` is a deliberate, meaningful value.

### Fix

A one-line change at `sphinx/ext/autodoc/__init__.py:1077`:

```diff
-            if not self.__all__:
+            if self.__all__ is None:
```

This replaces a truthiness check with an identity check against `None`. After the fix, `[]` no longer matches the "undefined" branch — it falls through to the else-branch where the empty `__all__` list causes every member to be marked as skipped, producing zero documented members as intended.

### Evidence

- Reproduction script confirmed `not []` evaluates to `True` at the Python level, demonstrating the logic bug.
- The source code of `get_object_members()` was read and traced; the truthiness check at line 1077 is the sole decision point controlling whether members are filtered.
- Manual trace confirmed all three cases behave correctly after the fix: `None` → show all, `['foo']` → filter to `foo`, `[]` → show none.
- The existing test `test_autodoc_ignore_module_all` (which tests `sort_by_all` with a non-empty `__all__`) is unaffected by the change.

## pylint-dev__pylint-7080

*To be completed.*
