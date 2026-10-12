# PyGhidra: property customizer shadows same-named static field (`PointerDataType.dataType`)

## Root cause
- `src/main/py/src/pyghidra/properties.py:24-54`: the `JImplementationFor("java.lang.Object")` class customizer
  (`__jclass_init__`) asks Java for bean properties (`PropertyUtils.getProperties`, line 36) and installs each one
  with `self._customize(field, property(fget, fset))` (line 54). It never checks whether the class already has a
  Java field of that name.
- `src/main/java/ghidra/pyghidra/property/PropertyUtils.java` (`getProperties` ~line 93, `doGetProperties` ~line 110):
  properties are derived only from `get*/is*/set*` methods; public fields are ignored.
- `PointerDataType` has `public final static PointerDataType dataType` and the inherited instance getter
  `getDataType()` (from `Pointer`), so a property `dataType` is installed on the class, replacing JPype's static
  field descriptor. Class-level access `PointerDataType.dataType` then fails (`Field 'dataType' is static`), while
  reflection (`class_.getField('dataType').get(None)`) still works. Byte/Word/DWord/PascalString255 have no
  `getDataType()`, so they are unaffected. **Hypothesis confirmed by source reading** (no JVM/Ghidra build was
  available here; nothing was executed).

## Minimal repro
```python
import pyghidra; pyghidra.start()
from ghidra.program.model.data import PointerDataType
PointerDataType.dataType  # AttributeError: Field 'dataType' is static
```

## Fix
`PropertyUtils.doGetProperties` now collects the names of public **static** fields (`cls.getFields()`) and drops any
bean property with the same name before merging. The property is simply not installed, so JPype's static field
descriptor stays visible. Instance getter methods (`getDataType()`) remain callable. Non-colliding getters are
unchanged. Instance (non-static) field collisions are deliberately left as before (property wins) to keep behaviour
changes minimal.

## How to test
Added `test_static_field_not_shadowed_by_property` in `src/main/py/tests/test_core.py` (needs a Ghidra install /
JVM; run in a fresh process: `pytest tests/test_core.py -k static_field`). **Not run here** — no JVM/Ghidra build.
Java change needs a Gradle build of the PyGhidra module (`gradle :PyGhidra:build`) or an installed Ghidra built from this branch.
Test assumption to verify: `PointerDataType.dataType.getDataType()` returns `None` (untyped pointer).

## Risks
- Anyone who relied on `SomeClass.x` being a property while a static field `x` exists (unlikely; it was broken at class level).
- Instance access `obj.dataType` on `PointerDataType` instances: the property is no longer installed, so JPype
  resolves it to the static field value (a `PointerDataType`) instead of the old getter result. Use `getDataType()`.
  This is a behaviour change for that name and worth a reviewer's eye.
- Python-side approach (check `inspect.getattr_static` for a JPype field) was rejected: static-ness is not reliably visible from Python.

## DRAFT upstream issue (not submitted)
**Title:** PyGhidra: `PointerDataType.dataType` raises AttributeError ("Field 'dataType' is static")

Ghidra 12.1.4, pyghidra 3.1.0, JPype 1.5.2, Python 3.13 (Windows). `ByteDataType.dataType` etc. work but
`PointerDataType.dataType` raises `AttributeError: Field 'dataType' is static`; reflection works. Cause: the PyGhidra
bean-property customizer installs a `dataType` property (from `Pointer.getDataType()`) that shadows the public static
field `dataType`. Property installation should skip names colliding with public static fields.

## DRAFT PR description (not submitted)
**PyGhidra: don't let bean properties shadow public static fields**

`PropertyUtils.getProperties` now skips properties whose name equals a public static field of the class, fixing
`PointerDataType.dataType` (static field + `getDataType()` getter). Adds a regression test in `test_core.py`.
Unchanged for all other getters. Untested locally against a JVM; see notes for risks.
