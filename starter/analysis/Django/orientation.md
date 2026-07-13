# Codebase Orientation

## django__django-16485

The relevant behavior is implemented by `floatformat()` in `django/template/defaultfilters.py`. The filter first parses its argument and optional `g`/`u` suffixes, converts the input to `Decimal`, and determines whether the value has a fractional component. It then builds a quantization exponent, derives a local decimal-context precision from `Decimal.as_tuple()`, calls `Decimal.quantize()`, reconstructs a non-scientific decimal string, and finally delegates localization and grouping to `django.utils.formats.number_format()`. The closest public tests are in `tests/template_tests/filter_tests/test_floatformat.py`, particularly `FunctionTests.test_zero_values()`, `test_negative_zero_values()`, and `test_low_decimal_precision()`. The failure occurs before localization, at the construction of the local `decimal.Context` used by `quantize()`.

## django__django-14580

Migration source generation is split between `django/db/migrations/writer.py` and serializers in `django/db/migrations/serializer.py`. `MigrationWriter.serialize()` selects a serializer for a Python value. Each serializer returns both a Python source expression and a set of import statements required to evaluate that expression. `TypeSerializer` contains special handling for `models.Model` and `type(None)`. `MigrationWriter` later aggregates and orders the returned imports when it renders the migration file. Tests are located in `tests/migrations/test_writer.py`; `WriterTests.assertSerializedResultEqual()` is the appropriate helper when both the expression and its imports must be checked. The defect is localized to the `models.Model` entry in `TypeSerializer.special_cases`, not to the writer's general import-rendering logic.

## sphinx-doc__sphinx-8595

_Reserved for the teammate responsible for Sphinx._

## pylint-dev__pylint-7080

_Reserved for the teammate responsible for Pylint._

## pytest-dev__pytest-7571

_Reserved for the teammate responsible for pytest._
