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

The relevant behavior is implemented by `floatformat()` in `django/template/defaultfilters.py`. The filter parses its precision argument and optional `g`/`u` suffixes, converts the input to `Decimal`, determines whether the value has a fractional component, derives a quantization exponent and local decimal-context precision, calls `Decimal.quantize()`, reconstructs a non-scientific decimal string, and finally delegates localization and grouping to `django.utils.formats.number_format()`. The closest public tests are in `tests/template_tests/filter_tests/test_floatformat.py`, especially `FunctionTests.test_zero_values()`, `test_negative_zero_values()`, and `test_low_decimal_precision()`. The reported failure occurs before localization, when `floatformat()` constructs the local `decimal.Context` used by `quantize()`.

## django__django-14580

Migration source generation is split between `django/db/migrations/writer.py` and serializers in `django/db/migrations/serializer.py`. `MigrationWriter.serialize()` selects a serializer for each Python value. Each serializer returns both a Python source expression and the import statements required to evaluate it. `TypeSerializer` contains special handling for `models.Model` and `type(None)`, while `MigrationWriter` later aggregates and orders the returned imports when rendering a migration. The relevant public tests are in `tests/migrations/test_writer.py`; `WriterTests.assertSerializedResultEqual()` is the appropriate helper when both expression text and imports must be verified. The defect is localized to the `models.Model` special case in `TypeSerializer`, not to the writer's general import-rendering logic.

## sphinx-doc__sphinx-8595

The bug is in `sphinx/ext/autodoc/__init__.py`, within the `ModuleDocumenter` class (a subclass of `Documenter` that handles `automodule` directives). Three locations are involved:

- **`__init__`** (line 989): Initializes `self.__all__ = None` — the default value indicating "no `__all__` defined."
- **`import_object`** (line 1015): Calls `inspect.getall(self.object)` to read the documented Python module's actual `__all__` variable, storing the result in `self.__all__`. If the module has `__all__ = []`, this stores `[]`; if it has `__all__ = ['foo']`, it stores `['foo']`; if there is no `__all__`, the `AttributeError` is caught and `self.__all__` stays `None`.
- **`get_object_members`** (line 1074): The decision point. When `want_all` is `True` (i.e., `:members:` was used), it checks `if not self.__all__:` (line 1077) to decide whether to return all members or filter by `__all__`. The truthiness check treats both `None` (undefined) and `[]` (empty) as falsy, conflating two semantically distinct cases.

The `self.__all__` value is also used for sorting in `sort_members` (line 1102).

## pylint-dev__pylint-7080

*To be completed.*
