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

## `django__django-16485`: `floatformat()` crashes for scaled zero values

### Assigned state and symptom

- Repository: `django/django`
- Base commit: `39f83765e12b0e5d260b7939fc3fe281d879b279`
- Environment used: Python 3.10.20 from `C:\Users\whliu\miniconda3\envs\swe_django\python.exe`
- Affected inputs: `floatformat("0.00", 0)` and `floatformat(Decimal("0.00"), 0)`

The complete PowerShell reproduction command was:

```powershell
$python = "C:\Users\whliu\miniconda3\envs\swe_django\python.exe"
Set-Location "D:\data_code\py for ai\final_project\final_prac_c\starter\repos\django__django-16485"

@'
from decimal import Decimal
import django
from django.conf import settings

if not settings.configured:
    settings.configure(
        SECRET_KEY="django-16485-reproduction",
        USE_I18N=False,
        USE_TZ=False,
    )
django.setup()

from django.template.defaultfilters import floatformat

for value in ["0.00", Decimal("0.00")]:
    try:
        print(f"{value!r} -> {floatformat(value, 0)!r}")
    except Exception as exc:
        print(f"{value!r} -> {type(exc).__name__}: {exc}")
'@ | & $python -
```

The pre-fix reproduction produced:

```text
'0.00' -> ValueError: valid range for prec is [1, MAX_PREC]
Decimal('0.00') -> ValueError: valid range for prec is [1, MAX_PREC]
```

The full traceback localized the failure to `django/template/defaultfilters.py:190`:

```text
Traceback (most recent call last):
  File "<stdin>", line 14, in <module>
  File "...\django\template\defaultfilters.py", line 190, in floatformat
    rounded_d = d.quantize(exp, ROUND_HALF_UP, Context(prec=prec))
ValueError: valid range for prec is [1, MAX_PREC]
```

### Mechanism

Both supplied forms converge on the same internal value because `floatformat()` executes:

```python
input_val = str(text)
d = Decimal(input_val)
```

For either input, the relevant state is:

| Expression | Value |
|---|---|
| `d` | `Decimal("0.00")` |
| `p` | `0` |
| `m = int(d) - d` | `Decimal("0.00")` |
| `bool(m)` | `False` |
| `d.as_tuple()` | `DecimalTuple(sign=0, digits=(0,), exponent=-2)` |
| initial `units` | `len((0,)) == 1` |
| exponent adjustment | `units += -2` |
| final `units` | `-1` |
| calculated `prec` | `abs(0) + (-1) + 1 == 0` |

The earlier fast path applies only when `not m and p < 0`. Here `not m` is true but `p == 0`, so execution continues. Python evaluates the arguments to `d.quantize()` before calling it; consequently, `Context(prec=0)` raises the exception and `quantize()` itself never starts. The precision formula assumed it would always produce a positive context precision, but a zero whose `Decimal` representation preserves a negative exponent violates that assumption.

### Regression test first

Two assertions were added to the existing `FunctionTests.test_zero_values` test before changing production code:

```python
self.assertEqual(floatformat("0.00", 0), "0")
self.assertEqual(floatformat(Decimal("0.00"), 0), "0")
```

Running only this test against the old implementation produced exit code 1 and the same `ValueError`:

```powershell
& $python tests\runtests.py template_tests.filter_tests.test_floatformat.FunctionTests.test_zero_values --parallel 1
```

```text
Found 1 test(s).
ERROR: test_zero_values (...FunctionTests)
ValueError: valid range for prec is [1, MAX_PREC]
Ran 1 test in 0.009s
FAILED (errors=1)
```

This establishes that the added test detects the original defect rather than merely documenting already-working behavior.

### Minimal patch

The production change places the invariant required by `decimal.Context` directly at the precision boundary:

```diff
-    prec = abs(p) + units + 1
+    prec = max(abs(p) + units + 1, 1)
```

This is not an input-specific special case. It preserves every previously positive precision and changes only invalid calculations below the API's minimum to the smallest valid context precision. The test changes add two assertions; the source change modifies one line. The complete diff is 3 insertions and 1 deletion across two files.

### Verification

After applying the one-line production change:

```text
Original base public module: 10/10 passed
FAIL_TO_PASS target: 1/1 passed
PASS_TO_PASS set:     9/9 passed
Nearby public module: 10/10 passed
git diff --check:     passed
git apply --check:    passed
```

The original reproduction now produces:

```text
'0.00' -> '0'
Decimal('0.00') -> '0'
```

The saved patch at `patches/django__django-16485.patch` has the same stable patch-id as the current working-tree diff (`b118a51af3543bf34b4efc042c2b9600e8565f8a`). Formal `sb-cli` evaluation has not yet been run, so no official SWE-bench result is claimed here.

## `django__django-14580`: generated migration omits the `models` import

### Assigned state and symptom

The assigned checkout is at base commit `36fa071d6ebd18a61c4d7f1b5c9d17106134bd44`. The issue describes a model with a custom field, an abstract model base, and a plain Python mixin. `makemigrations` serializes its concrete model bases as a custom class plus `models.Model`, but the generated file imports only the application module and `migrations`. Importing that migration then raises:

The reproduction and verification environment was:

- Operating shell: Windows PowerShell.
- Python: 3.10.20.
- Interpreter: `C:\Users\whliu\miniconda3\envs\swe_django\python.exe`.
- Checkout: `starter\repos\django__django-14580` at `36fa071d6ebd18a61c4d7f1b5c9d17106134bd44`.
- Import isolation: `$env:PYTHONPATH` was explicitly set to the assigned checkout after an inherited path initially selected the separate 16485 repository.

```text
NameError: name 'models' is not defined
```

Before changing code, an isolated reproduction constructed an equivalent `CreateModel` operation using an importable local custom class and `models.Model`, rendered it through `MigrationWriter`, and executed the result in an empty namespace:

```powershell
$env:PYTHONPATH = (Get-Location).Path
@'
import sys
sys.path.insert(0, 'tests')
import custom_migration_operations.operations
from django.db import migrations, models
from django.db.migrations.writer import MigrationWriter

migration = type('Migration', (migrations.Migration,), {
    'operations': [migrations.CreateModel(
        'MyModel',
        fields=[],
        bases=(custom_migration_operations.operations.TestOperation, models.Model),
    )],
    'dependencies': [],
})
source = MigrationWriter(migration, include_header=False).as_string()
print(source)
exec(source, {})
'@ | & 'C:\Users\whliu\miniconda3\envs\swe_django\python.exe' -
```

The pre-fix source contained:

```python
import custom_migration_operations.operations
from django.db import migrations

# ...
bases=(custom_migration_operations.operations.TestOperation, models.Model)
```

and failed with the expected `NameError`. This reproduces the mechanism without using a hidden test or upstream fix.

### Mechanism

`CreateModel.deconstruct()` exposes its `bases` tuple to `OperationWriter`. For each argument, `OperationWriter._write()` calls `MigrationWriter.serialize()`, which delegates to `serializer_factory()`. The tuple is handled by `TupleSerializer`, which recursively serializes both class objects and unions their import sets.

The custom class follows the normal `TypeSerializer` path and returns both a qualified expression and an import. `models.Model` takes this special-case branch instead:

```python
special_cases = [
    (models.Model, "models.Model", []),
    (type(None), 'type(None)', []),
]
```

The returned expression refers to the name `models`, but the returned import list is empty. Import aggregation therefore works exactly as implemented yet cannot include information the leaf serializer discarded. Existing `MigrationWriter` logic already merges `from django.db import models` into `from django.db import migrations, models` when that import is present. The general defect is thus a violated serializer contract, not a formatting problem or a custom-mixin special case.

Some existing writer tests execute generated source through a helper that passes the test module's globals to `exec()`. Because that module has already imported `models`, those tests can resolve `models.Model` even when the generated source is incomplete. The isolated reproduction uses an empty namespace and does not mask the missing import.

### Minimal patch and regression test

The production fix makes the special case report the import required by its rendered expression:

```diff
-(models.Model, "models.Model", []),
+(models.Model, "models.Model", ["from django.db import models"]),
```

This is general because every serializer consumer now receives a self-contained `(expression, imports)` result for `models.Model`. It does not inspect `CreateModel`, mixin order, field types, or application names. Existing import aggregation and writer merging perform the remaining work.

The focused regression test is:

```python
def test_serialize_type_model(self):
    self.assertSerializedResultEqual(
        models.Model,
        ("models.Model", {"from django.db import models"}),
    )
```

The test directly checks both halves of the serializer contract. During the one-shot, it was added in the same edit as the production fix and was not separately executed against the old source. The one-shot therefore was not test-first.

### Guided base-state red/green verification

The later guided review did not rewrite that history. Instead, it created a detached worktree at the assigned base commit, applied only the six-line test diff, confirmed that `django/db/migrations/serializer.py` remained unchanged, and ran the focused method. The old serializer returned an empty import set:

```text
AssertionError: Tuples differ:
('models.Model', set())
!=
('models.Model', {'from django.db import models'})

Ran 1 test in 0.002s
FAILED (failures=1)
exit code: 1
```

This is valid base-state evidence that the regression test detects the defect, but it is explicitly a post-one-shot guided validation, not a claim that the one-shot followed test-first development. After returning to the patched checkout, the same method passed 1/1.

### Verification

All corrected test commands explicitly set `PYTHONPATH` to the assigned checkout because the shell initially inherited a path to the other Django task:

```powershell
$env:PYTHONPATH = (Get-Location).Path
& 'C:\Users\whliu\miniconda3\envs\swe_django\python.exe' tests/runtests.py migrations.test_writer.WriterTests.test_serialize_type_model --verbosity 2
& 'C:\Users\whliu\miniconda3\envs\swe_django\python.exe' tests/runtests.py migrations.test_writer --verbosity 1
& 'C:\Users\whliu\miniconda3\envs\swe_django\python.exe' tests/runtests.py migrations --verbosity 1
```

Results:

```text
Original base writer module: 49/49 passed
Test-only on old source:     1/1 failed as expected
Focused regression:       1/1 passed
Public writer module:     50/50 passed
Broader migrations suite: 579 passed, 1 skipped
Post-fix isolated exec:    passed
git diff --check:          passed
```

The post-fix source imports `from django.db import migrations, models` and executes in an empty namespace. The saved patch at `patches/django__django-14580.patch` applies cleanly to the unchanged index and has the same stable patch-id as the working-tree diff (`e973e142a0146970d9d7687a458da809f50ec8db`). Formal `sb-cli` evaluation has not yet been run, so no official SWE-bench result is claimed.

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
