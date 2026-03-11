# errors — AI Context File

## Purpose

`errors` is the foundational error-handling module for Ka-Boost. It provides three capabilities: (1) a uniform wrapper for FANUC's `POST_ERR` and `TPDISPLAY` for human-readable error output, (2) a `CHK_STAT` shorthand that aborts on any bad FANUC status code, and (3) six `SET_UNINIT_*` helpers that forcibly initialize typed variables by name using `SET_VAR` — the standard pattern for clearing object fields in Karel class templates. Every module at Layer 1 or above depends on this module directly or transitively.

---

## Repository Layout

```
lib/errors/
├── package.json            rossum manifest; single dep: Strings
├── include/
│   ├── errors.klh          public routine declarations (header guard: errors_h)
│   └── errors.klt          error code #defines and severity constants (header guard: errors_t)
├── src/
│   └── errors.kl           implementation of all 9 public routines
└── test/
    └── test_errors.kl      single test: karelError with a line-wrap message
```

---

## Full API Reference

### errors.klt — Constants and Error Codes

Include with `%include errors.klt` or `%from errors.klt %import <name>`.

#### Severity Constants (passed as `errorType` to `karelError`)

| Constant | Value | Meaning |
|----------|-------|---------|
| `ER_WARN` | `0` | Warning only — task continues |
| `ER_WARN_STP` | `1` | Warning + pause all tasks + stop motion |
| `ER_ABORT` | `2` | Abort all tasks and cancel |
| `SUCCESS` | `0` | Standard success return value |
| `EXISTS` | `7015` | FANUC "already exists" — treated as non-error by CHK_STAT |
| `MAX_DISP_LNG` | `40` | Max characters per TP display line |

#### Named Error Codes (52 total)

**Array**
- `ARR_LEN_MISMATCH` = 12304
- `INVALID_INDEX` = 16024
- `VAR_TOO_BIG` = 16032
- `COULD_NOT_CLEAR_ARRAY` = 7169

**Path**
- `PATH_INDEX_OUT_OF_RANGE` = 12383
- `PATH_INVALID_INDEX` = 16041

**Value**
- `VAL_OUT_OF_RNG` = 16028
- `VAR_UNINIT` = 12311 — the "uninitialized variable" FANUC code; also used as a semantic error
- `INVALID_TYPE_CODE` = 12319
- `STR_TOO_LONG` = 22047
- `INVALID_STRING` = 12333
- `PORT_NOT_SIMULATED` = 13017
- `UNSUPPORTED_ITEM` = 40014

**Type**
- `EXPECTED_REAL` = 22032
- `EXPECTED_INT` = 22033
- `VECTOR_INVALID` = 12360

**Position**
- `POS_TYPE_MISMATCH` = 60032
- `POS_REG_NOT_SET` = 117368
- `INVALID_CONFIG` = 15121
- `GROUP_MASK_BAD` = 21018
- `SEARCH_MOTION_FAILED` = 30255
- `CANT_CONVERT_TO_JPOS` = 15291
- `INVALID_POSITION_TYPE` = 12259

**File**
- `FILE_NOT_OPEN` = 21004
- `CANT_READ` = 21016
- `CANT_WRITE` = 21015

**Task**
- `RUN_TASK_FAILED` = 24011
- `PROGRAM_TIMEDOUT` = 24028

**TPE (Teach Pendant Editor)**
- `TPE_UPDATE_FAILED` = 112080
- `TPE_PROGRAM_ALREADY_EXISTS` = 7015
- `TPE_POS_DOES_NOT_EXIST` = 7071
- `TPE_POS_ALREADY_EXISTS` = 7072
- `TPE_PROGRAM_DOES_NOT_EXIST` = 7073
- `TPE_OUT_OF_MEMORY` = 9036
- `TPE_NAME_INVALID_CHAR` = 9038
- `TPE_NAME_HAS_SPACE` = 9032
- `TPE_NAME_STARTS_WITH_NUM` = 9031

**Config**
- `INVALID_MOTION_GROUP` = 112032

**Queue**
- `QUEUE_IS_FULL` = 61001
- `QUEUE_IS_EMPTY` = 61002
- `QUEUE_BAD_SEQUENCE_NO` = 61003
- `QUEUE_BAD_SKIP_VALUE` = 61004

---

### errors.klh — Routine Declarations

```
ROUTINE CHK_STAT(rec_stat: INTEGER) FROM errors
ROUTINE karelError(stat: INTEGER; errStr : STRING; errorType : INTEGER) FROM errors
ROUTINE SET_UNINIT_I(progname : STRING; varname : STRING) FROM errors
ROUTINE SET_UNINIT_R(progname : STRING; varname : STRING) FROM errors
ROUTINE SET_UNINIT_B(progname : STRING; varname : STRING) FROM errors
ROUTINE SET_UNINIT_V(progname : STRING; varname : STRING) FROM errors
ROUTINE SET_UNINIT_F(progname : STRING; varname : STRING) FROM errors
ROUTINE SET_UNINIT_S(progname : STRING; varname : STRING) FROM errors
ROUTINE SET_UNI_ARRS(progname : STRING; varname : STRING; start_i : INTEGER; stop_i : INTEGER) FROM errors
```

---

### errors.kl — Routine Implementations

#### `CHK_STAT(rec_stat : INTEGER)`

Posts `POST_ERR(rec_stat, '', 0, ER_ABORT)` if `rec_stat <> SUCCESS` AND `rec_stat <> EXISTS`. The EXISTS carve-out prevents false-positive aborts when FANUC "already exists" codes (7015 / TPE_PROGRAM_ALREADY_EXISTS) are returned by operations that are idempotent by design.

```
CHK_STAT implementation body:
  IF (rec_stat <> SUCCESS) AND (rec_stat <> EXISTS) THEN
    POST_ERR(rec_stat, '', 0, ER_ABORT)
  ENDIF
```

#### `karelError(stat : INTEGER; errStr : STRING; errorType : INTEGER)`

1. Clears TP I/O status: `CLR_IO_STAT(TPDISPLAY)`
2. Writes two CRs to pad display
3. Calls `str_parse(errStr, MAX_DISP_LNG)` (from `Strings`) to line-wrap message at 40 chars
4. Writes wrapped message + CRs to `TPDISPLAY`
5. `FORCE_SPMENU(TP_PANEL, SPI_TPUSER, 1)` — brings TP USER panel into view
6. `POST_ERR(stat, '', 0, errorType)` — posts to FANUC error history

Note: `stat` is the FANUC error code posted. `errStr` is display-only.

#### `SET_UNINIT_I/R/B/V/F/S(progname : STRING; varname : STRING)`

All six follow identical logic, differing only in the local dummy array type used:
- `SET_UNINIT_I` → `ARRAY[1] OF INTEGER`
- `SET_UNINIT_R` → `ARRAY[1] OF REAL`
- `SET_UNINIT_B` → `ARRAY[1] OF BOOLEAN`
- `SET_UNINIT_V` → `ARRAY[1] OF VECTOR`
- `SET_UNINIT_F` → `ARRAY[1] OF XYZWPR`
- `SET_UNINIT_S` → `ARRAY[1] OF STRING[64]`

Mechanism: `SET_VAR(entry, progname, varname, uninit_dummy[1], STATUS)` — FANUC `SET_VAR` with an uninitialized dummy value intentionally triggers error 12311 ("uninitialized data"), which is then suppressed. The net effect: the target variable is written with an uninitilialized state, which is the goal. Any other status code is a real error and causes `POST_ERR`.

#### `SET_UNI_ARRS(progname : STRING; varname : STRING; start_i : INTEGER; stop_i : INTEGER)`

Iterates from `start_i` to `stop_i`, constructing the index string:
```
var_idx = varname + '[' + CHR(i + ORD('0', 1)) + ']'
```
Then calls `SET_VAR` with the same uninit pattern. Note: this uses `CHR(i + ORD('0',1))` which converts integer to digit character — this only works correctly for single-digit indices (0–9).

---

## Core Patterns

### Pattern 1: CHK_STAT After Every FANUC Native Call

Every FANUC built-in that returns a `STATUS` integer should be followed by `CHK_STAT`.

```karel
%include errors.klt
%from errors.klh %import CHK_STAT

-- After GET_REG / SET_REG / OPEN_FILE / etc.
GET_INT_REG(reg_no, val, STATUS)
CHK_STAT(STATUS)

SET_INT_REG(reg_no, new_val, STATUS)
CHK_STAT(STATUS)
```

Real usage: `lib/registers/src/registers.kl`, `lib/pose/lib/poselib/pose.kl`

### Pattern 2: karelError for Semantic / Logic Errors

Use when a caller has passed invalid data or an unsupported code path is reached.

```karel
%include errors.klt
%from errors.klh %import karelError

-- Type validation guard
IF (typ <> C_INT) AND (typ <> C_REAL) THEN
  karelError(INVALID_TYPE_CODE, 'expected INT or REAL type code', ER_ABORT)
  RETURN
ENDIF

-- Warning that allows continuation
IF UNINIT(delimeter) THEN
  karelError(VAR_UNINIT, 'delimiter not set', ER_WARN)
ENDIF
```

Real usage: `lib/csv/src/csv.kl`, `lib/pose/lib/poselib/pose.kl`, `lib/files/src/files.kl`

### Pattern 3: SELECT on Error Code for File / System Operations

When multiple distinct error outcomes need different messages:

```karel
SELECT ec OF
  CASE (FILE_NOT_OPEN):
    karelError(ec, 'File ' + filename + ' not opened', ER_ABORT)
  CASE (FILE_MULTIPLE_ACCESS):
    karelError(ec, 'File ' + filename + ' already opened', ER_WARN)
  CASE (FILE_RAM_FULL):
    WRITE TPDISPLAY(filename, ' EOF', CR)
  ELSE:
    karelError(ec, 'Unhandled problem opening ' + filename, ER_ABORT)
ENDSELECT
```

Real usage: `lib/files/src/files.kl`

### Pattern 4: SET_UNINIT_* in Class clear() / delete() Methods

Karel class templates (`.klc`) must explicitly uninitialize fields on clear. Use the `class_name` macro (GPP-expanded to the class instance name) as the `progname` argument.

```karel
-- In a .klc class template, clear() method:
SET_UNINIT_F('class_name', 'origin')       -- uninitialize XYZWPR field
SET_UNINIT_V('class_name', 'centroid')     -- uninitialize VECTOR field
SET_UNINIT_R('class_name', 'radius')       -- uninitialize REAL field
rast_bounds = FALSE
poly_count = 0
```

Real usage: `lib/draw/lib/canvas/canvas.klc`, `lib/shapes/include/class/box.klc`, `lib/shapes/include/class/cylinder.klc`

### Pattern 5: Selective VAR_UNINIT Suppression

When a value being out of range should mark it as "no valid reading" rather than error out:

```karel
-- Sensor reading out of range → uninitialize instead of error
IF (AIN[this.pin] > this.max) THEN
  SET_UNINIT_R('class_name', 'last_range')
ENDIF

-- Later, caller checks UNINIT() before using the value:
IF NOT UNINIT(sensor.last_range) THEN
  -- safe to use
ENDIF
```

Real usage: `lib/sensors/tof/include/tof_sensor.klc`

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Calling `karelError` with `stat=0` | No error posted on controller, but display still shown | Pass the actual FANUC status or a meaningful named code, not `SUCCESS` |
| Using `SET_UNI_ARRS` for indices > 9 | Incorrect variable name constructed (`CHR(i + ORD('0',1))` only works for 0–9) | Only use for arrays with ≤ 10 elements; write a manual loop for larger ranges |
| Forgetting the `EXISTS` carve-out in `CHK_STAT` | Assumes `CHK_STAT` only passes on `STATUS=0`; may misinterpret "already exists" as success in calling code | Remember `CHK_STAT` also passes on `EXISTS=7015` — if you need to distinguish, check STATUS explicitly before calling CHK_STAT |
| Passing a literal string as `progname` in `SET_UNINIT_*` for a class template | Runtime error or wrong variable targeted | In `.klc` files, always use the GPP macro `'class_name'` (which expands to the instantiated class program name) |
| `karelError` does not halt execution | After calling `karelError` with `ER_WARN`, code continues; unexpected state if callers assume it returns | `karelError` does NOT `RETURN` — add explicit `RETURN` or `GOTO` after `karelError` with `ER_WARN` if control flow must stop |
| Including `errors.klt` without `errors.klh` | Compiler error: error code constants resolve but routines are undeclared | Always include both: `%include errors.klt` + `%include errors.klh` or `%from errors.klh %import ...` |

---

## Dependencies

### What errors depends on
- `Strings` — uses `str_parse()` inside `karelError` to line-wrap messages to 40 chars for TP display

### What depends on errors (directly or transitively)
Every Ka-Boost module at Layer 1 and above. Direct consumers at each layer:
- Layer 1: `system`, `Strings` (circular — errors uses Strings, Strings uses errors)
- Layer 2: `math`, `matrix`
- Layer 3: `hash`, `queue`, `iterator`, `graph`
- Layer 4: `files`, `csv`, `xmlib`, `socket`
- Layer 5: `registers`, `display`, `multitask`, `TPE`, `forms`
- Layer 6: `shapes`, `pose`, `sensors`
- Layer 7: `draw`, `layout`, `paths/*`

---

## Build / Integration Notes

- `errors` is compiled as a standalone program (`PROGRAM errors`). All other modules link to it via `FROM errors` in routine declarations.
- The header guard convention: `%ifndef errors_h` / `%ifndef errors_t` — both guards must be respected.
- The standard import idiom used across the codebase is `%from errors.klh %import karelError, CHK_STAT` — selective import avoids declaring unused routines.
- `errors.kl` has `%NOLOCKGROUP` since it performs no motion.
- The module has no TP interfaces (no `tp-interfaces` in `package.json`).
- The single test (`test/test_errors.kl`) requires `%include klersys.kl` — the FANUC system constants header.
- Because `Strings` is a dependency of `errors`, and `errors` is a dependency of `Strings`, the rossum build must resolve this circular dependency correctly. In practice, `errors` is compiled before `Strings` because `errors.kl` only calls `str_parse` at runtime, not at compile time — the `%include strings.klh` provides only the declaration.
