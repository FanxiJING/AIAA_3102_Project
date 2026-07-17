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

The bug is in `django/template/defaultfilters.py`, within the `floatformat()` template filter. The function normalizes its input to a `Decimal`, calculates a quantization exponent and a context precision from the requested decimal places and the number's internal representation, then calls `Decimal.quantize()`. The defect is in the precision calculation immediately before the quantize call: for zero values whose `Decimal` representation preserves a negative exponent (e.g., `Decimal("0.00")`), the derived precision can drop to zero, violating `decimal.Context`'s minimum of 1. Tests live in `tests/template_tests/filter_tests/test_floatformat.py`; `FunctionTests.test_zero_values` is the FAIL_TO_PASS target, and the other nine methods in `tasks.csv` form the PASS_TO_PASS set.

## django__django-14580

The bug is in `django/db/migrations/serializer.py`, within the `TypeSerializer` class. Migration source generation is coordinated by `django/db/migrations/writer.py`: `OperationWriter` deconstructs each operation and calls `MigrationWriter.serialize()`, which delegates to `serializer_factory()`. For a `CreateModel` with mixed bases (a custom class plus `models.Model`), `TupleSerializer` recursively serializes each base and unions their import sets. `TypeSerializer` handles class objects and has a special-case table for `models.Model` and `type(None)`. The defect is in the `models.Model` special case: it emits the expression `models.Model` but returns an empty import list, so the generated migration file references a name without importing it. `MigrationWriter` already merges `from django.db import models` into the imports when present, so only the serializer's contract needs repair. Focused tests live in `tests/migrations/test_writer.py`; `WriterTests.test_serialize_type_model` is the FAIL_TO_PASS target.

## sphinx-doc__sphinx-8595

The bug is in `sphinx/ext/autodoc/__init__.py`, within the `ModuleDocumenter` class (a subclass of `Documenter` that handles `automodule` directives). Three locations are involved:

- **`__init__`** (line 989): Initializes `self.__all__ = None` — the default value indicating "no `__all__` defined."
- **`import_object`** (line 1015): Calls `inspect.getall(self.object)` to read the documented Python module's actual `__all__` variable, storing the result in `self.__all__`. If the module has `__all__ = []`, this stores `[]`; if it has `__all__ = ['foo']`, it stores `['foo']`; if there is no `__all__`, the `AttributeError` is caught and `self.__all__` stays `None`.
- **`get_object_members`** (line 1074): The decision point. When `want_all` is `True` (i.e., `:members:` was used), it checks `if not self.__all__:` (line 1077) to decide whether to return all members or filter by `__all__`. The truthiness check treats both `None` (undefined) and `[]` (empty) as falsy, conflating two semantically distinct cases.

The `self.__all__` value is also used for sorting in `sort_members` (line 1102).

## pylint-dev__pylint-7080

The relevant logic begins in `pylint/lint/pylinter.py`, where `PyLinter._discover_files()` uses `os.walk()` to discover Python files and delegates filtering to `_is_ignored_file()` in `pylint/lint/expand_modules.py`; that helper applies the `ignore`, `ignore-patterns`, and `ignore-paths` configuration values, which are declared in `pylint/lint/base_options.py` and compiled during configuration parsing. The bug is located at this final path-matching boundary, where recursive Windows paths are compared with configured regular expressions. The surrounding recursive-ignore tests are in `tests/test_self.py` under `TestRunTC`, and the focused student reproduction is `tests/test_pylint_7080_reproduction.py`.
