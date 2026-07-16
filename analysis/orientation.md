# Orientation

Brief description of where the relevant logic lives for each task.

## pytest-dev__pytest-7571

The bug is in `src/_pytest/logging.py`, within the `LogCaptureFixture` class. Three methods are involved:

- **`__init__`** (line 344): Initializes `_initial_logger_levels`, a dict that saves original logger levels for restoration. Does not track the handler's original level.
- **`set_level`** (line 422): Mutates both the Python logger level (via `logger_obj.setLevel`) and the shared `LogCaptureHandler` level (via `self.handler.setLevel`). The logger's original level is saved via `_initial_logger_levels.setdefault()`, but the handler's original level is not saved.
- **`_finalize`** (line 350): Called at fixture teardown. Restores logger levels from `_initial_logger_levels`, but does not restore `handler.level`.

The shared handler is created once in `LoggingPlugin.__init__` (line 527) as `self.caplog_handler`. The `_runtest_for` method (line 674) stores this same handler into each test's `item._store` via `caplog_handler_key`. The `LogCaptureFixture.handler` property (line 360) retrieves it from the store. This shared-scope design means handler state persists across test boundaries.

Also relevant: the `caplog` fixture definition at line 461 creates a new `LogCaptureFixture` per test and calls `_finalize` in teardown.

## `django__django-16485`

This instance uses `django/django` at base commit `39f83765e12b0e5d260b7939fc3fe281d879b279`. The failing public API is the `floatformat()` template filter in `django/template/defaultfilters.py`. Its relevant path is: normalize the input with `str()`, construct a `Decimal`, parse the requested decimal places as `p`, determine whether the number has a fractional component through `m = int(d) - d`, calculate a quantization exponent and a local decimal-context precision, call `Decimal.quantize()`, then localize the formatted result. The defect is localized to the precision calculation immediately before the quantize call. Tests for both direct function calls and template rendering live in `tests/template_tests/filter_tests/test_floatformat.py`; `FunctionTests.test_zero_values` is the FAIL_TO_PASS target, while the other nine methods listed in `tasks.csv` form the PASS_TO_PASS set. The student-visible nearby regression command is `python tests/runtests.py template_tests.filter_tests.test_floatformat`.

### Adoption sequence

After checking out the assigned commit, the codebase was adopted in this order: (1) `problem_statements/django__django-16485.md` established the two failing public calls; (2) the `django__django-16485` row in `tasks.csv` identified the FAIL_TO_PASS/PASS_TO_PASS scope, and `public_tests.csv` identified the nearby module; (3) `tests/template_tests/filter_tests/test_floatformat.py` showed existing zero, negative-zero, localization, infinity, and low-precision behavior; (4) `django/template/defaultfilters.py` exposed the full `floatformat()` conversion and precision path; (5) the import at the top of that module confirmed that `Context` is Python's `decimal.Context`; and (6) a direct traceback plus printed `Decimal.as_tuple()`, `m`, `units`, and `prec` values connected the public symptom to line 190. This order moved from contract and regression surface to implementation and finally to runtime proof, rather than searching for a likely patch first.

## `django__django-14580`

This instance uses `django/django` at base commit `36fa071d6ebd18a61c4d7f1b5c9d17106134bd44`. Migration source generation is coordinated by `django/db/migrations/writer.py`: `OperationWriter` deconstructs each operation and asks `MigrationWriter.serialize()` to serialize arguments such as a `CreateModel` base tuple; the writer unions every returned import and finally combines `from django.db import models` with its mandatory migrations import. The value serializers live in `django/db/migrations/serializer.py`. `TupleSerializer` recursively serializes each base, while `TypeSerializer` handles class objects and has special representations for `models.Model` and `type(None)`. The defect is in the `models.Model` special case: it emits the expression `models.Model` without declaring the import needed by that expression. Focused serializer and generated-migration tests live in `tests/migrations/test_writer.py`; the provided regression target is `WriterTests.test_serialize_type_model`, and the student-visible nearby command is `python tests/runtests.py migrations.test_writer`.

### Adoption sequence

The one-shot agent inspected the assigned checkout in this recorded order: (1) the problem statement, base commit, worktree status, `tasks.csv`, and `public_tests.csv`; (2) a broad local search for `MigrationWriter`, `CreateModel`, `bases`, and `models.Model`; (3) `django/db/migrations/writer.py` to understand operation deconstruction and import aggregation; (4) the relevant serializer declarations in `django/db/migrations/serializer.py`, especially `TupleSerializer` and `TypeSerializer`; (5) `tests/migrations/test_writer.py` to understand serializer assertions, generated-source execution, and existing import-omission coverage; (6) nearby local migration test modules to find an importable custom base for reproduction; and (7) an isolated `MigrationWriter(...).as_string()` plus `exec(source, {})` reproduction. The guided review then returned to the task metadata, a clean base worktree, the focused test contract, and the existing `test_models_import_omitted` boundary before accepting the one-line serializer fix.

## sphinx-doc__sphinx-8595

The bug is in `sphinx/ext/autodoc/__init__.py`, within the `ModuleDocumenter` class (a subclass of `Documenter` that handles `automodule` directives). Three locations are involved:

- **`__init__`** (line 989): Initializes `self.__all__ = None` — the default value indicating "no `__all__` defined."
- **`import_object`** (line 1015): Calls `inspect.getall(self.object)` to read the documented Python module's actual `__all__` variable, storing the result in `self.__all__`. If the module has `__all__ = []`, this stores `[]`; if it has `__all__ = ['foo']`, it stores `['foo']`; if there is no `__all__`, the `AttributeError` is caught and `self.__all__` stays `None`.
- **`get_object_members`** (line 1074): The decision point. When `want_all` is `True` (i.e., `:members:` was used), it checks `if not self.__all__:` (line 1077) to decide whether to return all members or filter by `__all__`. The truthiness check treats both `None` (undefined) and `[]` (empty) as falsy, conflating two semantically distinct cases.

The `self.__all__` value is also used for sorting in `sort_members` (line 1102).

## pylint-dev__pylint-7080

*To be completed.*
