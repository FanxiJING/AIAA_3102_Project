# Orientation

## pytest-dev__pytest-7571

The relevant logic is in `src/_pytest/logging.py`, primarily in the `LogCaptureFixture` class. `__init__()` stores the original logger levels in `_initial_logger_levels`, `set_level()` changes both the selected logger and the shared `LogCaptureHandler`, and `_finalize()` restores state at test teardown. The handler is created once by `LoggingPlugin.__init__()` and reused by each test's `caplog` fixture through the item store, so the interaction between the session-scoped handler and the per-test fixture is the key area to inspect. The related tests are in `testing/logging/test_fixture.py`.

## sphinx-doc__sphinx-8595

The relevant logic is in `sphinx/ext/autodoc/__init__.py`, in the `ModuleDocumenter` class. `__init__()` initializes `self.__all__`, `import_object()` reads the module's actual `__all__` value, and `get_object_members()` decides which members are documented when `:members:` is requested; `sort_members()` also uses the stored value. The important distinction is between `None` for an undefined `__all__` and `[]` for an explicitly empty one. The related test module is `tests/test_ext_autodoc_automodule.py`.

## django__django-16485

The relevant logic is the `floatformat()` filter in `django/template/defaultfilters.py`. It parses the precision argument, converts the input to `Decimal`, calculates the quantization precision, calls `Decimal.quantize()` with a local context, and then formats and localizes the result through Django's number-formatting utilities. The reported failure is in the precision calculation before localization, so the closest tests are in `tests/template_tests/filter_tests/test_floatformat.py`, especially the zero-value, negative-zero, and low-precision cases.

## django__django-14580

The relevant logic is split between `django/db/migrations/serializer.py` and `django/db/migrations/writer.py`. `MigrationWriter.serialize()` selects a serializer, `TypeSerializer` handles special values such as `models.Model`, and each serializer returns both a source expression and the imports required to evaluate it; the writer later aggregates and orders those imports when rendering the migration. The defect is localized to the `models.Model` special case in `TypeSerializer`, while the appropriate regression tests and `assertSerializedResultEqual()` helper are in `tests/migrations/test_writer.py`.

## pylint-dev__pylint-7080

The relevant logic begins in `pylint/lint/pylinter.py`, where `PyLinter._discover_files()` uses `os.walk()` to discover Python files and delegates filtering to `_is_ignored_file()` in `pylint/lint/expand_modules.py`; that helper applies the `ignore`, `ignore-patterns`, and `ignore-paths` configuration values, which are declared in `pylint/lint/base_options.py` and compiled during configuration parsing. The bug is located at this final path-matching boundary, where recursive Windows paths are compared with configured regular expressions. The surrounding recursive-ignore tests are in `tests/test_self.py` under `TestRunTC`, and the focused student reproduction is `tests/test_pylint_7080_reproduction.py`.
