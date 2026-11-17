# Issue Resolution: Fix allow_dash for Path parameters

## Metadata
- **Target Repository**: `tiangolo/typer`
- **Private Repository**: `rajanjava/typer-private`
- **Upstream Issue / PR**: [#2016: Fix allow_dash for Path parameters](https://github.com/fastapi/typer/pull/2016)
- **Status**: Resolved & Verified

---

## Problem Description & Root Cause
In `typer._param_types.coerce_cli_path()`, when validating and coercing a path value:
```python
if path_type is None or path_type is str or path_type is bytes:
    rv: Any = value
else:
    assert lenient_issubclass(path_type, Path)
    if isinstance(value, path_type):
        rv = value
    else:
        # validate into path_type, producing e.g. Path("-")
        rv = adapter.validate_python(value)

is_dash = (
    parameter_info.file_okay and parameter_info.allow_dash and rv in (b"-", "-")
)
```

When a CLI parameter has annotation `pathlib.Path` with `allow_dash=True`, `adapter.validate_python("-")` converts the input string `"-"` into a `pathlib.Path` instance (`Path("-")`).

Python's `pathlib.Path` objects compare unequal to plain strings and bytes (`Path("-") == "-"` evaluates to `False`). Therefore, the check `rv in (b"-", "-")` always evaluated to `False` for `Path` instances.

Because `is_dash` evaluated to `False`:
- Typer did not recognize `"-"` as the stdin/stdout dash sentinel.
- If `parameter_info.exists=True` was specified, Typer executed `os.stat(rv)` on `"-"`, which raised `FileNotFoundError`.
- Typer aborted execution with:
  `Invalid value for '--config': path '-' does not exist.`

---

## Solution
Use `os.fspath(rv)` in the dash check:
```python
is_dash = (
    parameter_info.file_okay
    and parameter_info.allow_dash
    and os.fspath(rv) in (b"-", "-")
)
```
`os.fspath()` seamlessly handles `str`, `bytes`, and `os.PathLike` (including `pathlib.Path`), normalizing `Path("-")` to `"-"` while preserving `bytes` (`b"-"`) and `str` (`"-"`).

---

## Code Changes
```diff
diff --git a/typer/_param_types.py b/typer/_param_types.py
index 3f6b190..e391a45 100644
--- a/typer/_param_types.py
+++ b/typer/_param_types.py
@@ -241,7 +241,9 @@ def coerce_cli_path(
                 raise BadParameter(get_error_msg(exc), ctx=ctx, param=param) from exc
 
     is_dash = (
-        parameter_info.file_okay and parameter_info.allow_dash and rv in (b"-", "-")
+        parameter_info.file_okay
+        and parameter_info.allow_dash
+        and os.fspath(rv) in (b"-", "-")
     )
 
     if not is_dash:
```

---

## Verification
- Added reproduction tests `test_coercion_path_allow_dash` and `test_coercion_path_allow_dash_with_resolve_path` in `tests/test_coercion.py`.
- Verified reproduction failure prior to fix:
  `AssertionError: Invalid value for '--config': path '-' does not exist.`
- Verified all tests pass after fix:
  `14 passed in 0.70s`
- Verified `ruff check` on modified files:
  `All checks passed!`
