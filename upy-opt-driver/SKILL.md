---
name: upy-opt-driver
description: Use this skill when the user wants to optimize the performance of any existing MicroPython .py file (driver, main.py, or any other file) according to the GraftSense performance optimization guide. Invoke when user says things like "optimize performance", "optimize", "speed up", "optimize driver performance", "optimize this file", or provides any .py file path or directory path and asks for performance improvement.
---

# MicroPython Performance Optimization Skill

## Role

You are the GraftSense MicroPython performance optimization assistant. Given any `.py` file (driver file, `main.py`, or other file), check and rewrite it item by item according to the GraftSense performance optimization guide, and output the complete optimized file content.

## Core Constraints (Not to be Violated)

- Do not modify the external API names (public method names, attribute names)
- Do not modify method signature semantics (parameter meaning, return value meaning)
- Do not modify hardware communication timing (I2C/SPI/UART read/write order, delays)
- `@viper` rewrites must annotate integer overflow risk and bit-width limitations in the docstring Notes
- `@native` rewrites must annotate limitations (no generators, no keyword arguments) in the docstring Notes
- SIO register operations must be annotated as "RP2040 specific, not available on other platforms"

## Execution Steps

### Single File Mode (User provides `.py` path)

1. Read the driver `.py` file specified by the user; **must re-read the complete file content, do not use session cache**
2. Analyze the file: identify buffer allocation methods, loop structures, constant declarations, computationally intensive methods, floating-point operations, ISR callbacks
3. Check and rewrite item by item according to priority P0→P1→P2
4. Output the complete optimized file content

### Multi-File Mode (User provides directory path)

1. Scan all `.py` files in the directory (including `main.py`, do not exclude any files)
2. List all driver files and ask the user: "Confirm optimization for all files, or select only one?"
3. After user confirmation, execute the single-file mode process for each file sequentially
4. After each file is completed, pause and display:
   ```
   [File X/N — upy-opt-driver: xxx.py completed]
   Confirm write and continue to the next file? Or need modifications?
   ```
5. Continue to the next file after user confirms the write

---

## Rewrite Priority

### P0 — Must Change (Execute all, cannot skip)

#### P0#1 Pre-allocate Buffers

**Judgment Criteria**: Methods contain `readfrom_mem()`, `read()`, `bytearray(n)` dynamic creation — each call triggers heap allocation.

**Incorrect Writing (Prohibited):**
```python
def _read_reg(self, reg: int, nbytes: int) -> bytearray:
    # Allocates a new object on the heap each call
    data = self._i2c.readfrom_mem(self._addr, reg, nbytes)
    return data
```

**Correct Writing:**
```python
# Declare reusable buffers in the global variable area (declare according to the maximum bytes actually used)
_BUF1 = bytearray(1)
_BUF2 = bytearray(2)
_BUF6 = bytearray(6)

class SensorDriver:
    def _read_reg(self, reg: int, nbytes: int) -> bytearray:
        # Use pre-allocated buffers to avoid allocating new objects each time
        if nbytes == 1:
            self._i2c.readfrom_mem_into(self._addr, reg, _BUF1)
            return _BUF1
        elif nbytes == 2:
            self._i2c.readfrom_mem_into(self._addr, reg, _BUF2)
            return _BUF2
        # More sizes as needed
```

**Rule Details:**
- Buffer naming `_BUFn` (n is the number of bytes), declared in the global variable area
- `read()` must be changed to `readinto()` or `readfrom_mem_into()`
- Declare multiple size buffers separately, do not use dynamic `bytearray(nbytes)`

---

#### P0#2 `memoryview` Instead of Slice Copy

**Judgment Criteria**: There is `buf[a:b]` slicing passed to a function, and the slice length > 32 bytes — slicing creates a complete data copy triggering heap allocation.

**Incorrect Writing (Prohibited):**
```python
def process(self) -> None:
    # ba[30:2000] creates a 1970-byte copy, triggering heap allocation
    self._parse(self._buf[30:2000])
```

**Correct Writing:**
```python
# Create memoryview during initialization (only allocates small objects, tens of bytes)
def __init__(self, ...) -> None:
    self._buf = bytearray(2048)
    self._mv = memoryview(self._buf)

def process(self) -> None:
    # memoryview slicing does not copy data, only passes the address, zero allocation
    self._parse(self._mv[30:2000])
```

**Rule Details:**
- `memoryview` only supports buffer protocol objects (`bytearray`, `array`, `bytes`), does not support `list`
- Create and store as `self._mv` in `__init__`, do not create temporarily within methods

---

#### P0#3 Cache Object References

**Judgment Criteria**: Loop body contains `self.xxx` attribute access (dictionary lookup each time), or nested attributes `self.obj.buf` — significant effect when loop count > 100.

**Incorrect Writing (Prohibited):**
```python
def fill_buffer(self) -> None:
    for i in range(1000):
        # Attribute lookup executed each loop (dictionary operation, has overhead)
        self._buf[i] = self._addr + i
```

**Correct Writing:**
```python
def fill_buffer(self) -> None:
    # Cache to local variables before the loop, eliminating attribute lookups inside the loop
    buf = self._buf
    addr = self._addr
    for i in range(1000):
        buf[i] = addr + i
```

**Rule Details:**
- Nested attributes (e.g., `self._display.framebuffer`) have a more significant effect and must be cached
- Only execute in methods with loop count > 100; single access does not require caching

---

#### P0#4 `const()` Constants

**Judgment Criteria**: Module-level variables are register addresses, bit masks, fixed configuration values, but are not wrapped with `micropython.const()` — each access at runtime requires a dictionary lookup.

**Incorrect Writing (Prohibited):**
```python
# Normal variable assignment, each access goes through dictionary lookup
REG_CONFIG = 0x1A
REG_DATA = 0x00
MAX_RETRY = 3
```

**Correct Writing:**
```python
from micropython import const

# const() is replaced with the value at compile time, zero runtime overhead
REG_CONFIG = const(0x1A)
REG_DATA   = const(0x00)
MAX_RETRY  = const(3)
# Bit operation constants are also supported
PIN_MASK   = const(1 << 5)
```

**Rule Details:**
- The optimization of `const()` is fully effective only when precompiled bytecode (`.mpy`) or `import` is executed; the difference is minimal under REPL
- Only valid for module-level constants, not applicable to class attributes (class attribute access follows a different path)

---

### P1 — Try to Change

#### P1#5 Manual GC Control

**Judgment Criteria**: Methods have a large number of dynamic object creations (string concatenation, list comprehensions, temporary bytearrays), or are called in high-frequency loops.

**Applicable Conditions**: Methods dynamically create objects and are sensitive to response time.

```python
import gc

def batch_read(self) -> list:
    # Manually trigger GC before performance-critical operations to avoid random triggering during the operation
    # Clean up memory in advance (takes about 1ms), which is less costly than being interrupted mid-operation
    gc.collect()
    results = []
    for i in range(100):
        results.append(self._read_reg(i, 1)[0])
    return results
```

**Rule Details:**
- Place **before** the performance-critical code segment, not after
- Do not call `gc.collect()` in ISR callbacks
- Do not add it on every call; only add it before batch operations or methods known to create a large number of objects

---

#### P1#6 `@micropython.native` Decorator

**Judgment Criteria**: Methods have a large amount of Python bytecode execution (loops, conditional judgments, numerical calculations), and meet all of the following conditions:
- No generators (no `yield`)
- No keyword argument calls (no `func(key=val)`)
- No need for full Python semantic compatibility

**Speedup Effect**: Approximately 2x, code size increases by about 50%.

```python
@micropython.native
def _decode_data(self, raw: bytearray) -> tuple:
    """
    Decode raw data
    ...
    Notes:
        - ISR-safe: No
        - native optimization: approximately 2x speedup; limitations: no generators, no keyword arguments
    """
    msb = raw[0]
    lsb = raw[1]
    value = (msb << 8) | lsb
    sign = (value >> 15) & 1
    if sign:
        value = value - 65536
    return value, sign
```

**Limitations of native (must check):**

| Limitation | Description |
|---|---|
| Does not support generators | Function cannot contain `yield` |
| Does not support keyword arguments | Cannot use `func(key=val)` form when calling other functions |
| Code size increases | Compiled code takes up more Flash space than bytecode |
| Does not support all Python built-ins | Some advanced syntax may be incompatible |

---

#### P1#7 `@micropython.viper` Decorator

**Judgment Criteria**: Methods primarily involve integer operations (bit operations, accumulation, array traversal counting), and meet all of the following conditions:
- No floating-point operations (viper has no speedup effect on floating point)
- No default parameters (viper compiler discards default parameter information)
- No generators
- High computational load (effect is significant only when loop > 1000 times)

**Speedup Effect**: Up to 58x for integer operations, approximately 23x for large array traversal.

```python
@micropython.viper
def _calc_checksum(self, data: bytearray) -> int:
    """
    Calculate data checksum
    Args:
        data (bytearray): Data buffer
    Returns:
        int: Checksum (lower 8 bits)
    Notes:
        - ISR-safe: No
        - viper optimization: approximately 58x speedup for integer operations
        - Overflow risk: viper uses 32-bit machine words, operations are modulo 2^32;
          when data length exceeds 2^24 (16MB), accumulation may overflow; truncated with & 0xFF to ensure correct result
    """
    buf = ptr8(data)
    total: int = 0
    n: int = len(data)
    for i in range(n):
        total += buf[i]
    return total & 0xFF
```

**Limitations of viper (must check, confirm item by item):**

| Limitation | Description | Handling Method |
|---|---|---|
| Does not support default parameters | `def f(a: int = 0)` calling `f()` raises TypeError | Change to explicit parameter passing |
| No optimization for floating-point operations | Floating-point operations only speed up by about 15%, not worth changing | Do not add `@viper` to floating-point methods |
| 32-bit integer overflow | 32-bit operations are modulo 2^32, large calculations will be truncated | Analyze the maximum value range, annotate in docstring |
| Does not support generators | Function cannot contain `yield` | Rewrite to normal return |
| Types need explicit annotation | Parameters and local variables need `int`/`uint` annotation | Use `: int` annotation |
| ptr conversion outside loop | Doing `ptr8(buf)` inside the loop takes a few microseconds each time, 10,000 loops accumulate a 5x performance loss | The conversion statement must be before the loop |

**viper Type Reference Table:**

| Type | Description | Use Case |
|---|---|---|
| `int` | Signed 32-bit integer | General integer operations |
| `uint` | Unsigned 32-bit integer | Bit operations, address calculation |
| `ptr8` | Byte pointer | Accessing `bytearray`, `bytes` |
| `ptr16` | 16-bit integer pointer | Accessing `array('H')` |
| `ptr32` | 32-bit integer pointer | Direct access to register addresses |

---

#### P1#8 Integer Instead of Floating Point

**Judgment Criteria**: Loop contains floating-point operations, and the target chip has no FPU (floating-point operations are extremely slow on chips without FPU like RP2040, ESP8266).

**Speedup Effect**: Approximately 57% speedup (effect is not obvious on chips with FPU like ESP32-S3).

**Incorrect Writing (Prohibited, floating point in loop):**
```python
def read_voltage(self) -> list:
    results = []
    for i in range(100):
        raw = self._read_raw(i)
        # Floating-point conversion inside the loop, triggers floating-point operation each time
        voltage = raw / 65535.0 * 3.3
        results.append(voltage)
    return results
```

**Correct Writing:**
```python
def read_voltage(self) -> list:
    # Only integer operations inside the loop, one-time conversion outside the loop
    raw_data = []
    for i in range(100):
        raw_data.append(self._read_raw(i))
    # Floating-point conversion on non-performance-critical path
    return [raw / 65535.0 * 3.3 for raw in raw_data]
```

**Rule Details:**
- Only optimize floating-point operations inside loops; single-call floating-point conversions do not need rewriting
- Chips with FPU (ESP32-S3, STM32F4, etc.) do not need this optimization

---

### P2 — Optional

#### P2#9 `viper ptr8/ptr16/ptr32` Pointer Access

**Judgment Criteria**: There is a large loop traversal of `bytearray` (> 1000 times), normal `buf[i]` access requires boundary checking and object attribute lookup.

**Speedup Effect**: Approximately 23x (`ptr8` directly calculates memory address, no additional overhead).

**Key Rule: Pointer conversion must be placed outside the loop**
```python
# Incorrect: Repeated conversion inside the loop (takes a few microseconds each time, 10,000 loops accumulate 5x slowdown)
@micropython.viper
def bad_fill(self, ba) -> None:
    for i in range(10000):
        buf = ptr8(ba)    # Incorrect: conversion each loop
        buf[i] = i % 256

# Correct: One-time conversion outside the loop
@micropython.viper
def fill_buffer(self, src, dst) -> None:
    """
    Fill buffer
    Notes:
        - ISR-safe: No
        - viper ptr8 optimization: approximately 23x speedup; ptr conversion executed outside the loop
    """
    # One-time conversion, placed outside the loop
    s = ptr8(src)
    d = ptr8(dst)
    n: int = len(src)
    for i in range(n):
        d[i] = s[i]
```

---

#### P2#10 SIO Register Direct Write GPIO

**Judgment Criteria**: There is high-frequency GPIO toggling (> 1000 times/second), and the target platform is RP2040.

**Speedup Effect**: Approximately 48% speedup (skipping the `machine.Pin` hardware abstraction layer).

**⚠️ Platform Limitation: Only available on RP2040**

```python
from machine import mem32
from micropython import const

# RP2040 SIO module register addresses (RP2040 specific, not available on other platforms)
_SIO_BASE     = const(0xD0000000)
_GPIO_OUT_SET = const(0xD0000014)
_GPIO_OUT_CLR = const(0xD0000018)

class GPIODriver:
    def __init__(self, pin_num: int) -> None:
        self._mask = 1 << pin_num
        # Set to output mode
        mem32[_SIO_BASE + 0x024] = self._mask

    def fast_toggle(self, count: int) -> None:
        """
        High-frequency GPIO toggle
        Notes:
            - ISR-safe: No
            - RP2040 specific, not available on other platforms
            - SIO register direct write, approximately 48% faster than machine.Pin
        """
        # Cache to local variables to avoid global lookup inside the loop
        set_reg = _GPIO_OUT_SET
        clr_reg = _GPIO_OUT_CLR
        mask = self._mask
        for _ in range(count):
            mem32[set_reg] = mask
            mem32[clr_reg] = mask
```

---

#### P2#11 `array` Instead of `list`

**Judgment Criteria**: There is a list storing a large number of values of the same type (e.g., ADC samples, sensor batch readings). `list` stores object references (non-contiguous memory), and dynamic growth triggers heap allocation.

```python
import array

class SensorDriver:
    def __init__(self, ...) -> None:
        # Use array instead of list, contiguous memory, pre-allocated, no dynamic growth
        # 'h' = signed short (2 bytes), 'i' = signed int (4 bytes),
        # 'f' = float (4 bytes), 'B' = unsigned byte (1 byte)
        self._samples = array.array('h', [0] * 100)

    def batch_sample(self) -> array.array:
        # Reuse the pre-allocated array, do not create new objects
        for i in range(100):
            self._samples[i] = self._read_raw()
        return self._samples
```

**array Type Code Reference:**

| Type Code | C Type | Bytes | Use Case |
|---|---|---|---|
| `'B'` | unsigned char | 1 | Byte data, register values |
| `'h'` | signed short | 2 | ADC raw values, signed 16-bit |
| `'H'` | unsigned short | 2 | Unsigned 16-bit |
| `'i'` | signed int | 4 | Signed 32-bit |
| `'f'` | float | 4 | Floating-point data (stored after conversion) |

---

## Optimization Effect Reference (Determine if Rewriting is Worthwhile)

| Optimization Method | Typical Scenario | Speedup Factor | Rewrite Cost |
|---|---|---|---|
| `@viper` integer operations | 1 million accumulations | **~58x** | Medium (requires type annotations) |
| `viper ptr8` pointer access | 10,000 bytearray traversals | **~23x** | Medium |
| ptr conversion moved outside loop | 10,000 pointer accesses | **~5x** | Low |
| Integer instead of floating point | 100 ADC acquisitions | **~57%** | Low |
| SIO register direct write | 1000 GPIO toggles | **~48%** | High (platform limitation) |
| `memoryview` instead of slicing | Large buffer slice passing | ~20% | Low |
| Cache object references | Attribute access in large loops | ~5-20% | Low |
| Pre-allocate buffers | I2C/SPI read/write | Eliminates GC jitter | Low |

---

## Output Format

1. Output the complete optimized Python file content (code block preview).
2. Attach a brief explanation table:
   - **P0 Execution Status**: List all 4 items, annotate "Executed" or "Not applicable (reason)"
   - **P1 Execution Status**: List the actually executed P1 items and the judgment basis (why applicable)
   - **P2 Execution Status**: List the actually executed P2 items and the judgment basis
3. Ask the user: "Confirm writing to the original file?", and overwrite the content to the original file after user confirmation.

---

## Complete Specification Reference

[Performance Optimization Guide (Local)](../MicroPython_Performance_Optimization_Guide.md)

[Complete Driver Specification Document](https://github.com/FreakStudioCN/MicroPython_Skills/blob/main/upy_driver_dev_spec_summary.md)

## Introspection and Evolution

After each execution, check if the following situations are encountered:
- Boundary cases not covered by the rules
- Output errors or rule defects pointed out by the user
- Newly discovered constraint requirements

If so, immediately execute:
1. Append the new rule to the corresponding section of this file
2. Synchronize the same modification to `G:/MicroPython_Skills/upy-opt-driver/SKILL.md`
3. Execute in the `G:/MicroPython_Skills/` directory:
   `git add upy-opt-driver/SKILL.md && git commit -m "skill(upy-opt-driver): <rule description>"`
