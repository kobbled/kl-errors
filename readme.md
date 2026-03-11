# errors

FANUC Karel error handling, status checking, and variable initialization utilities.

## Overview

Karel has no exceptions and FANUC built-in calls signal failure via integer status codes. This module gives you three things:

1. **`CHK_STAT`** — one-line guard after any FANUC native call; aborts the program if the status code is bad.
2. **`karelError`** — formats and displays a human-readable message on the teach pendant, then posts a FANUC error at the severity you choose (warning, warning+stop, or abort).
3. **`SET_UNINIT_*`** — forcibly uninitializes a named variable at runtime using `SET_VAR`. Used in class templates to reset object fields during `clear()` / `delete()`.

Every other Ka-Boost module depends on this one directly or transitively.

---

## Files

| File | Purpose |
|------|---------|
| `include/errors.klt` | 52 named error code `#define`s + severity and success constants |
| `include/errors.klh` | Forward declarations for all 9 public routines (include in callers) |
| `src/errors.kl` | Implementation |
| `test/test_errors.kl` | Single test — exercises `karelError` with a line-wrap message |
| `package.json` | Rossum manifest; depends on `Strings` |

---

## API Reference

### Severity Constants (`errors.klt`)

| Constant | Value | Effect |
|----------|-------|--------|
| `ER_WARN` | `0` | Display message; task execution continues |
| `ER_WARN_STP` | `1` | Display message; pause all tasks and stop motion |
| `ER_ABORT` | `2` | Display message; abort all tasks and cancel |
| `SUCCESS` | `0` | Standard "no error" return value |
| `EXISTS` | `7015` | FANUC "already exists" — not treated as an error by `CHK_STAT` |
| `MAX_DISP_LNG` | `40` | Max characters per TP display line |

### Named Error Codes (`errors.klt`)

All constants are integers matching FANUC native error codes or Ka-Boost-defined codes.

**Array:** `ARR_LEN_MISMATCH`, `INVALID_INDEX`, `VAR_TOO_BIG`, `COULD_NOT_CLEAR_ARRAY`

**Path:** `PATH_INDEX_OUT_OF_RANGE`, `PATH_INVALID_INDEX`

**Value:** `VAL_OUT_OF_RNG`, `VAR_UNINIT` (12311), `INVALID_TYPE_CODE`, `STR_TOO_LONG`, `INVALID_STRING`, `PORT_NOT_SIMULATED`, `UNSUPPORTED_ITEM`

**Type:** `EXPECTED_REAL`, `EXPECTED_INT`, `VECTOR_INVALID`

**Position:** `POS_TYPE_MISMATCH`, `POS_REG_NOT_SET`, `INVALID_CONFIG`, `GROUP_MASK_BAD`, `SEARCH_MOTION_FAILED`, `CANT_CONVERT_TO_JPOS`, `INVALID_POSITION_TYPE`

**File:** `FILE_NOT_OPEN`, `CANT_READ`, `CANT_WRITE`

**Task:** `RUN_TASK_FAILED`, `PROGRAM_TIMEDOUT`

**TPE:** `TPE_UPDATE_FAILED`, `TPE_PROGRAM_ALREADY_EXISTS`, `TPE_POS_DOES_NOT_EXIST`, `TPE_POS_ALREADY_EXISTS`, `TPE_PROGRAM_DOES_NOT_EXIST`, `TPE_OUT_OF_MEMORY`, `TPE_NAME_INVALID_CHAR`, `TPE_NAME_HAS_SPACE`, `TPE_NAME_STARTS_WITH_NUM`

**Config:** `INVALID_MOTION_GROUP`

**Queue:** `QUEUE_IS_FULL`, `QUEUE_IS_EMPTY`, `QUEUE_BAD_SEQUENCE_NO`, `QUEUE_BAD_SKIP_VALUE`

---

### Routines (`errors.klh` / `src/errors.kl`)

#### `CHK_STAT(rec_stat : INTEGER)`

Aborts if `rec_stat` is not `SUCCESS` (0) and not `EXISTS` (7015). Call immediately after any FANUC built-in that returns a status code.

```karel
GET_INT_REG(reg_no, val, STATUS)
CHK_STAT(STATUS)
```

The `EXISTS` carve-out exists because operations like `CREATE_TPE` legitimately return 7015 when the program already exists, which is often harmless.

---

#### `karelError(stat : INTEGER; errStr : STRING; errorType : INTEGER)`

Displays `errStr` on the teach pendant (line-wrapped to 40 chars), forces the TP USER panel to show, then posts `stat` to the FANUC error log at severity `errorType`.

```karel
-- Abort with a named error code
karelError(INVALID_TYPE_CODE, 'expected INT or REAL type', ER_ABORT)

-- Warning only — execution continues after this call
karelError(VAR_UNINIT, 'delimiter not set, using default', ER_WARN)
```

**Important:** `karelError` does **not** stop execution by itself. For warnings (`ER_WARN`), code continues running after the call. If you need to stop execution, add `RETURN` or an appropriate `GOTO` after the call.

---

#### `SET_UNINIT_I(progname, varname)` / `SET_UNINIT_R` / `SET_UNINIT_B` / `SET_UNINIT_V` / `SET_UNINIT_F` / `SET_UNINIT_S`

Marks a named variable as uninitialized using FANUC's `SET_VAR` with an uninitialized dummy value. FANUC's error 12311 ("uninitialized data") is intentionally suppressed.

| Routine | Target Type |
|---------|-------------|
| `SET_UNINIT_I` | INTEGER |
| `SET_UNINIT_R` | REAL |
| `SET_UNINIT_B` | BOOLEAN |
| `SET_UNINIT_V` | VECTOR |
| `SET_UNINIT_F` | XYZWPR |
| `SET_UNINIT_S` | STRING[64] |

```karel
-- In a class clear() method (progname is the GPP macro 'class_name'):
SET_UNINIT_F('class_name', 'origin')
SET_UNINIT_V('class_name', 'centroid')
SET_UNINIT_R('class_name', 'radius')
```

---

#### `SET_UNI_ARRS(progname, varname, start_i, stop_i)`

Initializes a range of array elements by building the index string `varname[i]` for each i from `start_i` to `stop_i`.

```karel
-- Uninitialize elements 1 through 5 of an array
SET_UNI_ARRS('myprog', 'data_arr', 1, 5)
```

**Limitation:** Only works for single-digit indices (0–9) due to the single-character index conversion. Do not use for arrays with more than 10 elements.

---

## Common Patterns

### 1. Wrap every FANUC native call with CHK_STAT

FANUC built-ins like `GET_INT_REG`, `OPEN_FILE`, `SET_VAR`, `CNV_STR_CONF`, etc. return a status integer. Rather than checking it manually every time, delegate to `CHK_STAT`:

```karel
%include errors.klt
%from errors.klh %import CHK_STAT

ROUTINE my_routine
  VAR
    val : INTEGER
    STATUS : INTEGER
  BEGIN
    GET_INT_REG(1, val, STATUS)
    CHK_STAT(STATUS)

    SET_INT_REG(2, val + 1, STATUS)
    CHK_STAT(STATUS)
  END my_routine
```

### 2. Validate inputs, then karelError for logic failures

Use named error codes and an appropriate severity. Prefer `ER_ABORT` for invalid inputs (the caller has a bug). Use `ER_WARN` only when the program can meaningfully continue.

```karel
%include errors.klt
%from errors.klh %import karelError

ROUTINE process_type(typ : INTEGER)
  BEGIN
    IF (typ <> C_INT) AND (typ <> C_REAL) THEN
      karelError(INVALID_TYPE_CODE, 'process_type: expected C_INT or C_REAL', ER_ABORT)
      RETURN
    ENDIF
    -- proceed
  END process_type
```

### 3. Use SELECT for multi-case file/system error handling

When an operation can fail in multiple distinct ways that need different messages:

```karel
%include errors.klt
%from errors.klh %import karelError

OPEN_FILE(filename, 'RO', fl, ec)
SELECT ec OF
  CASE (FILE_NOT_OPEN):
    karelError(ec, 'File ' + filename + ' not found', ER_ABORT)
  CASE (FILE_MULTIPLE_ACCESS):
    karelError(ec, 'File ' + filename + ' already open', ER_WARN)
  ELSE:
    IF ec <> 0 THEN
      karelError(ec, 'Unexpected error opening ' + filename, ER_ABORT)
    ENDIF
ENDSELECT
```

### 4. Clear object fields with SET_UNINIT_* in class templates

Karel `.klc` class templates use GPP macro `class_name` which expands to the actual program name at compile time. Use this as `progname` in `SET_UNINIT_*` calls:

```karel
-- In a .klc class template, delete() or clear() routine:
%from errors.klh %import SET_UNINIT_F, SET_UNINIT_R, SET_UNINIT_V

ROUTINE clear
  BEGIN
    SET_UNINIT_F('class_name', 'this.origin')   -- XYZWPR field
    SET_UNINIT_V('class_name', 'this.normal')   -- VECTOR field
    SET_UNINIT_R('class_name', 'this.radius')   -- REAL field
  END clear
```

After this, callers can use Karel's `UNINIT()` built-in to check whether a field has been set before using it.

### 5. Mark a value as "no valid reading" instead of erroring

Rather than aborting when a value is out of valid range, mark it as uninitialized so callers can check with `UNINIT()`:

```karel
-- Sensor or computed value out of valid range:
IF (reading > MAX_VALID) THEN
  SET_UNINIT_R('class_name', 'this.last_reading')
  RETURN
ENDIF

-- Caller:
IF NOT UNINIT(sensor.last_reading) THEN
  use_value(sensor.last_reading)
ENDIF
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Calling `karelError` with `stat=0` | Message appears on TP but no error is posted to FANUC history | Pass the actual FANUC status code or a meaningful named constant — not `SUCCESS` |
| Assuming `karelError` stops execution | Code runs with bad state after a `ER_WARN` call | Add an explicit `RETURN` after `karelError` if execution must stop |
| Using `SET_UNI_ARRS` for index > 9 | Wrong variable name constructed; wrong variable cleared or `SET_VAR` errors | Only use for indices 0–9; write a manual `FOR` loop for larger arrays |
| Using a literal class name in `SET_UNINIT_*` inside a `.klc` file | Wrong variable targeted if the class is instantiated with a different name | Always use the GPP macro `'class_name'` as `progname` in class templates |
| Including `errors.klh` without `errors.klt` | Named error codes like `INVALID_INDEX` are undeclared | Always include both: `%include errors.klt` before `%include errors.klh` |
| Forgetting the `EXISTS` carve-out in `CHK_STAT` | Idempotent create operations abort when they return 7015 | Either use `CHK_STAT` and accept the carve-out, or check status manually if you need to distinguish SUCCESS from EXISTS |

---

## Build Flow

`errors` is compiled as a standalone Karel program (`PROGRAM errors`). Other modules declare its routines with `FROM errors` in their `.klh` headers. At build time, rossum resolves `errors` as a dependency and ensures it appears in `build.ninja` before any module that `%include`s `errors.klh`.

The selective import form is preferred across the codebase to keep declarations minimal:
```
%from errors.klh %import karelError, CHK_STAT
```

See the [Ka-Boost readme](../../readme.md) for full build instructions.
