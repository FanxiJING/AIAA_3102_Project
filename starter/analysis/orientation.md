# Orientation

Brief description of where the relevant logic lives for each task.

## pytest-dev__pytest-7571

The bug is in `src/_pytest/logging.py`, within the `LogCaptureFixture` class. Three methods are involved:

- **`__init__`** (line 344): Initializes `_initial_logger_levels`, a dict that saves original logger levels for restoration. Does not track the handler's original level.
- **`set_level`** (line 422): Mutates both the Python logger level (via `logger_obj.setLevel`) and the shared `LogCaptureHandler` level (via `self.handler.setLevel`). The logger's original level is saved via `_initial_logger_levels.setdefault()`, but the handler's original level is not saved.
- **`_finalize`** (line 350): Called at fixture teardown. Restores logger levels from `_initial_logger_levels`, but does not restore `handler.level`.

The shared handler is created once in `LoggingPlugin.__init__` (line 527) as `self.caplog_handler`. The `_runtest_for` method (line 674) stores this same handler into each test's `item._store` via `caplog_handler_key`. The `LogCaptureFixture.handler` property (line 360) retrieves it from the store. This shared-scope design means handler state persists across test boundaries.

Also relevant: the `caplog` fixture definition at line 461 creates a new `LogCaptureFixture` per test and calls `_finalize` in teardown.

## django__django-16485

*To be completed.*

## django__django-14580

*To be completed.*

## sphinx-doc__sphinx-8595

*To be completed.*

## pylint-dev__pylint-7080

*To be completed.*
