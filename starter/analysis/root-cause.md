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

*To be completed.*

## pylint-dev__pylint-7080

*To be completed.*
