---
name: upy-slim-driver
description: Use this skill when the user wants to reduce the memory footprint of any existing MicroPython .py file (driver, main.py, or any other file) according to the GraftSense memory minimization guide. Invoke when user says things like "减少内存占用", "slim", "降低RAM使用", "对文件做内存优化", "优化内存", or provides any .py file path or directory path and asks for memory reduction.
---

# MicroPython Memory Footprint Optimization Skill

## Role

You are the GraftSense MicroPython memory optimization assistant. Given any `.py` file (driver file, `main.py`, or other file), check and rewrite it item by item according to the GraftSense memory footprint minimization guide, and output the fully optimized file content.

This Skill focuses on **RAM usage** (peak heap memory, number of runtime objects), complementing `upy-opt-driver` (which focuses on execution speed). There is one overlap: pre-allocated buffers (P0#1) — this Skill handles it from the perspective of "avoiding heap allocation, reducing peak RAM", while `upy-opt-driver` handles it from the perspective of "eliminating GC jitter, improving speed"; the rewrite result is the same, so it is not executed twice.

## Core Constraints (Not to be Violated)

- Do not modify the external API names (public method names, property names)
- Do not modify method signature semantics (parameter meaning, return value meaning)
- Do not modify hardware communication timing (I2C/SPI/UART read/write order, delays)
- The `_CONST` private constant rewrite only applies to constants **used internally by the module**; if a constant is directly referenced by external code (e.g., `driver.REG_CONFIG`), keep the public name and prompt the user in the description table
- `gc.disable()` intervals must be short and bounded; prohibited within potentially blocking I/O operations
- `const()` optimization is only fully effective in `.mpy` or frozen bytecode; its effect is minimal in the REPL

## Execution Steps

### Single File Mode (User provides `.py` path)

1. Read the user-specified driver `.py` file; **must re-read the complete file content, do not use session cache**
2. Analyze the file: identify constant declaration methods, buffer allocation locations, string concatenation methods, register table data structures, GC control status, `struct` usage, class attribute storage methods
3. Check and rewrite item by item according to priority P0→P1→P2
4. Output the fully optimized file content

### Multi-File Mode (User provides directory path)

1. Scan all `.py` files in the directory (including `main.py`, do not exclude any files)
2. List all driver files and ask the user: "Confirm memory optimization for all files, or select only a specific one?"
3. After user confirmation, execute the single-file mode process for each file sequentially
4. Pause after each file is completed, displaying:
   ```
   [File X/N — upy-slim-driver: xxx.py completed]
   Confirm write and continue to the next file? Or need modifications?
   ```
5. Continue to the next file after user confirms the write

---

## Rewrite Priority

### P0 — Mandatory (Execute all, cannot skip)

#### P0#1 Pre-allocated Buffers (Avoid Heap Allocation)

**Judgment Criteria**: Methods contain `readfrom_mem()`, `read()`, `bytearray(n)` dynamic creation — each call allocates a new object on the heap, increasing peak RAM and triggering GC.

**Incorrect (Prohibited):**
```python
def _read_reg(self, reg: int, nbytes: int) -> bytearray:
    # Each call heap-allocates nbytes bytes, 100 calls = 100 heap allocations
    # Peak RAM = sum of all unreclaimed objects, can lead to memory fragmentation
    data = self._i2c.readfrom_mem(self._addr, reg, nbytes)
    return data
```

**Correct:**
```python
# Declare reusable buffers in the global variable area (declare according to actual maximum bytes, fixed RAM usage)
_BUF1 = bytearray(1)
_BUF2 = bytearray(2)
_BUF6 = bytearray(6)

class SensorDriver:
    def _read_reg(self, reg: int, nbytes: int) -> bytearray:
        # Reuse pre-allocated buffers, heap allocations reduced from N times to 0 times
        # Peak RAM = buffer size (fixed), no fragmentation risk
        if nbytes == 1:
            self._i2c.readfrom_mem_into(self._addr, reg, _BUF1)
            return _BUF1
        elif nbytes == 2:
            self._i2c.readfrom_mem_into(self._addr, reg, _BUF2)
            return _BUF2
        elif nbytes == 6:
            self._i2c.readfrom_mem_into(self._addr, reg, _BUF6)
            return _BUF6
```

**Rule Details:**
- Buffer naming `_BUFn` (n is the number of bytes), declared in the global variable area
- `read()` must be changed to `readinto()` or `readfrom_mem_into()`
- Declare separate buffers for different sizes, do not use dynamic `bytearray(nbytes)`
- If this item has already been executed by `upy-opt-driver`, skip it and note "Already handled by upy-opt-driver" in the description table

#### P0#2 Private `_CONST` Constants (Zero RAM Usage)

**Judgment Criteria**: Module-level constants use public names (e.g., `REG_CONFIG = const(0x1A)`) and are only used internally within the module — public `const` still occupies an entry in the module's global dictionary (about 40 bytes/entry); private `_CONST` is not written to the global dictionary, RAM usage is zero.

**Incorrect (Prohibited, using public names for internal use only):**
```python
from micropython import const

# Public constants: each occupies about 40 bytes in the global dictionary
# 5 constants = about 200 bytes RAM
REG_CONFIG  = const(0x1A)
REG_DATA    = const(0x00)
REG_STATUS  = const(0x02)
MAX_RETRY   = const(3)
TIMEOUT_MS  = const(500)

class SensorDriver:
    def _read_status(self) -> int:
        # Used internally by the module, external code does not access these constants
        return self._read_reg(REG_STATUS, 1)[0]
```

**Correct:**
```python
from micropython import const

# Private names (underscore prefix): not written to the global dictionary, RAM usage is zero
# 5 constants save about 200 bytes RAM
_REG_CONFIG  = const(0x1A)
_REG_DATA    = const(0x00)
_REG_STATUS  = const(0x02)
_MAX_RETRY   = const(3)
_TIMEOUT_MS  = const(500)

class SensorDriver:
    def _read_status(self) -> int:
        # Use private constants, replaced with numeric values at compile time
        return self._read_reg(_REG_STATUS, 1)[0]
```

**Rule Details:**
- Only rewrite constants **used internally by the module**; public constants referenced externally (e.g., `driver.REG_CONFIG`) keep their original names, prompt the user in the description table
- Synchronously update all references to the constant within the file (`REG_CONFIG` → `_REG_CONFIG` in method bodies)
- If the file already uses the `_CONST` private form entirely, mark "Already compliant, skip"
- **Key Limitation**: `const()` optimization is only fully effective in `.mpy` or frozen bytecode; its effect is minimal in the REPL

**Edge Case Handling:**

| Scenario | Handling Method |
|---|---|
| Constant referenced by external code (e.g., `from driver import REG_CONFIG`) | Keep public name, note "Public API, cannot be made private" in description table |
| Constant used in bitwise expressions (e.g., `const(1 << 5)`) | Supported, `const()` can handle compile-time integer expressions |
| Constant referencing another constant (e.g., `COLS = const(0x10 + ROWS)`) | Error, must change to literal value `const(0x10 + 33)` |

#### P0#3 Avoid String `+` Concatenation in Loops

**Judgment Criteria**: String `+` concatenation inside a loop body — each `+` creates a new string object, N loops = N heap allocations, and old objects wait for GC collection.

**Incorrect (Prohibited):**
```python
def build_report(self, readings: list) -> str:
    result = ""
    for i, val in enumerate(readings):
        # Each + creates a new string object, 100 loops = 100 heap allocations
        # 1st: "" + "ch" = "ch" (creates object 1)
        # 2nd: "ch" + "0" = "ch0" (creates object 2, object 1 becomes garbage)
        # 3rd: "ch0" + "=" = "ch0=" (creates object 3, object 2 becomes garbage)
        # ...cumulatively creates hundreds of temporary objects, peak RAM is very high
        result = result + "ch" + str(i) + "=" + str(val) + "\n"
    return result
```

**Correct Method 1 (`.join()` + Generator, Recommended):**
```python
def build_report(self, readings: list) -> str:
    # Generator yields string fragments one by one, join() allocates the final string at once
    # Only creates one final string object, intermediate objects are collected in the same GC cycle
    # Peak RAM = final string size + generator overhead (about 100 bytes)
    return "\n".join("ch{}={}".format(i, val) for i, val in enumerate(readings))
```

**Correct Method 2 (`.format()` Pre-allocation, Suitable for Fixed Formats):**
```python
def build_report(self, readings: list) -> str:
    # Pre-allocate list to avoid dynamic growth
    lines = []
    for i, val in enumerate(readings):
        # format() only creates one string object, no intermediate concatenation
        lines.append("ch{}={}".format(i, val))
    return "\n".join(lines)
```

**Correct Method 3 (Static Concatenation, Compile-time Merging):**
```python
# Adjacent string literals are merged into one object at compile time, zero runtime allocation
_MSG_INIT = "SensorDriver " "init " "ok"
_MSG_ERR  = "I2C " "read " "failed"
_HEADER   = "GraftSense " "v1.0 " "2026"
```

**Rule Details:**
- Only rewrite string `+` **inside loops**; single concatenation (outside loops) does not need rewriting
- Prefer `.format()` or f-strings (MicroPython 1.20+ supports); use adjacent literal merging for multi-segment static strings
- Log/debug strings executed only in `DEBUG` mode can be exempted
- If the loop count is < 10 and the total string length is < 100 bytes, the rewrite benefit is minimal and can be skipped

**Memory Comparison (100 loops):**

| Method | Temporary Objects | Estimated Peak RAM |
|---|---|---|
| `+` concatenation in loop | ~400 | ~10KB (severe fragmentation) |
| `.join()` + generator | ~1 | ~2KB (contiguous memory) |
| Static literal merging | 0 | 0 (compile-time) |

#### P0#4 `bytes`/`bytearray` Instead of Register Lists

**Judgment Criteria**: Module-level or class-level `list` storing register addresses, configuration sequences, lookup tables — `list` stores object references (about 8 bytes pointer per element + object header), `bytes` stores raw bytes (1 byte per element), 100 register addresses save about 700 bytes RAM.

**Incorrect (Prohibited):**
```python
# list storage: 8 elements occupy about 64+ bytes RAM (each integer object 4 bytes + pointer 8 bytes)
_REG_TABLE = [0x00, 0x01, 0x02, 0x03, 0x10, 0x11, 0x12, 0x20]
_INIT_SEQ  = [0xAE, 0x00, 0x10, 0x40, 0xA1, 0xC8, 0xA6]

class DisplayDriver:
    def _init_display(self) -> None:
        for reg in _INIT_SEQ:
            self._write_reg(reg)
```

**Correct (`bytes` Read-Only Table):**
```python
# bytes storage: 1 byte per element, 8 elements only occupy 8 bytes RAM (saves about 90%)
_REG_TABLE = b'\x00\x01\x02\x03\x10\x11\x12\x20'
_INIT_SEQ  = b'\xAE\x00\x10\x40\xA1\xC8\xA6'

class DisplayDriver:
    def _init_display(self) -> None:
        # bytes supports iteration and indexing, usage is the same as list
        for reg in _INIT_SEQ:
            self._write_reg(reg)
```

**Correct (`bytearray` Modifiable Table):**
```python
# Use bytearray when elements need to be modified at runtime
_CAL_TABLE = bytearray(b'\x00\x80\xFF\x40')

class SensorDriver:
    def _update_calibration(self, idx: int, val: int) -> None:
        # bytearray supports element assignment
        _CAL_TABLE[idx] = val
```

**`struct` Storage for Multi-byte Values (e.g., 16-bit Register Addresses):**
```python
import struct

# list storing 16-bit addresses: 10 elements about 80+ bytes
# struct storage: 10 x 2 bytes = 20 bytes (saves about 75%)
_REG16_TABLE = struct.pack('10H', 0x0100, 0x0200, 0x0300, 0x0400, 0x0500,
                                   0x0600, 0x0700, 0x0800, 0x0900, 0x0A00)

class AdvancedDriver:
    def _get_reg(self, idx: int) -> int:
        # unpack_from unpacks from a specified offset, no slicing needed (slicing creates a copy)
        return struct.unpack_from('H', _REG16_TABLE, idx * 2)[0]
```

**Rule Details:**
- Only applies to containers of **homogeneous numeric types**; `list` with mixed types (strings + numbers) is not rewritten
- Element values must be in the range 0–255 to use `bytes`/`bytearray`; use `struct` for values outside this range
- Use `bytes` for read-only tables, `bytearray` for tables that need runtime modification
- `struct.pack()` format codes: `'B'` = unsigned byte, `'H'` = unsigned 16-bit, `'I'` = unsigned 32-bit

**Applicable Scenario Reference Table:**

| Data Type | Recommended Solution | Memory Usage (100 elements) | Usage Restrictions |
|---|---|---|---|
| 8-bit register address (0-255) | `bytes` | ~100 bytes | Read-only |
| 8-bit configuration sequence (needs modification) | `bytearray` | ~100 bytes | Modifiable |
| 16-bit register address | `struct.pack('100H', ...)` | ~200 bytes | Requires `unpack_from` for access |
| Mixed types (strings + numbers) | `list` (keep as-is) | ~800+ bytes | No restrictions |

---

### P1 — Try to Modify

#### P1#5 `gc.collect()` Before Batch Operations

**Judgment Criteria**: Methods contain batch dynamic object creation (multiple `bytearray`, list comprehensions, string concatenation) and are sensitive to response time — triggering GC in advance can avoid random triggering during the operation (random trigger timing is uncontrollable and may pause in the middle of an I2C transmission).

```python
import gc

def batch_read(self, count: int) -> list:
    """
    Batch read sensor data
    Args:
        count (int): Number of reads
    Returns:
        list: List of readings
    Notes:
        - Manually trigger GC before batch operations to clean up fragmentation and reduce the probability of interruption during the operation
        - gc.collect() takes about 1ms, which is more controllable than random triggering during the operation
    """
    # Manually trigger GC before batch operations to clean up fragmentation
    gc.collect()
    results = []
    for i in range(count):
        results.append(self._read_reg(i, 2))
    return results
```

**Rule Details:**
- Place it **before** the batch operation, not after (cleaning up after cannot prevent triggering during the operation)
- Do not call `gc.collect()` in ISR callbacks (ISRs require very low latency)
- Do not add for single small operations; only add before methods known to create many temporary objects
- Applicable scenarios: batch I2C reads, large data frame construction, multi-channel sampling

#### P1#6 `gc.disable()`/`gc.enable()` Protecting Critical Sections

**Judgment Criteria**: There are timing-sensitive continuous operation sequences (e.g., multi-step I2C writes, SPI frame transmission), and triggering GC in the middle would cause timing violations or data errors.

**Key Constraint**: Dynamic memory allocation is prohibited within the section (otherwise, running out of memory causes a direct crash, with no GC fallback)

```python
import gc

def _send_frame(self, data: bytearray) -> None:
    """
    Send a complete data frame
    Args:
        data (bytearray): Frame data (must be pre-allocated, creation is prohibited within the gc.disable section)
    Notes:
        - ISR-safe: No
        - gc.disable() protected section: Prevents GC from triggering during frame transmission, avoiding timing violations
        - No dynamic memory allocation operations (bytearray, list, str concatenation, etc.) are allowed within the section
        - The section must be short and bounded (microsecond to millisecond level), and must not contain potentially blocking I/O
    """
    # Disable GC to ensure frame transmission is not interrupted
    gc.disable()
    try:
        # Critical timing operations: CS low → SPI write → CS high
        # No dynamic memory allocation is allowed within this section
        self._cs.value(0)
        self._spi.write(data)  # data must be a pre-allocated bytearray
        self._cs.value(1)
    finally:
        # Must restore in finally to prevent GC from being permanently disabled due to an exception
        gc.enable()
```

**Incorrect Example (Prohibited, dynamic allocation within the section):**
```python
def _send_frame_bad(self, reg: int, value: int) -> None:
    gc.disable()
    try:
        # Error: Creating bytearray within the section, if memory is insufficient, it will crash directly
        data = bytearray([reg, value])  # Prohibited!
        self._spi.write(data)
    finally:
        gc.enable()
```

**Correct Example (Pre-allocated Buffer):**
```python
# Global pre-allocated buffer
_FRAME_BUF = bytearray(64)

def _send_frame_good(self, reg: int, value: int) -> None:
    # Prepare data outside the section
    _FRAME_BUF[0] = reg
    _FRAME_BUF[1] = value
    
    gc.disable()
    try:
        # Only operate on the pre-allocated buffer within the section, zero dynamic allocation
        self._spi.write(memoryview(_FRAME_BUF)[:2])
    finally:
        gc.enable()
```

**Rule Details:**
- The `gc.disable()` section must be **short and bounded** (microsecond to millisecond level), and must not contain potentially blocking I/O
- Must be wrapped with `try/finally` to ensure recovery even on exceptions
- Dynamic memory allocation is prohibited within the section (otherwise, running out of memory causes a direct crash, with no GC fallback)
- Not applicable to ordinary read/write methods; only for frame-level operations with clear timing requirements
- Not applicable to ISR callbacks (ISRs themselves already disable interrupts, no additional protection needed)

**Applicable Scenario Judgment:**

| Scenario | Is `gc.disable()` Applicable? | Reason |
|---|---|---|
| SPI continuous frame transmission (CS held low) | Yes | Mid-operation GC would cause frame interval timeout |
| I2C multi-byte write (single transaction) | No | I2C hardware handles timing automatically, no protection needed |
| Ordinary sensor read | No | Single operation is short, GC impact is negligible |
| High-frequency GPIO toggling (> 1kHz) | Yes | GC pause would cause waveform distortion |

#### P1#7 `struct.pack_into()` Reusing Buffers

**Judgment Criteria**: Methods have repeated calls to `struct.pack()` — `struct.pack()` returns a new `bytes` object each time (heap allocation); `struct.pack_into()` writes to a pre-allocated buffer, zero heap allocation.

**Incorrect (Prohibited):**
```python
import struct

def _build_cmd(self, reg: int, value: int) -> None:
    # Each call creates a new bytes object, triggering heap allocation
    # 100 calls = 100 heap allocations + 100 temporary objects waiting for GC
    cmd = struct.pack('>BH', reg, value)
    self._i2c.writeto(self._addr, cmd)
```

**Correct:**
```python
import struct

# Pre-allocate command buffer in the global variable area (1 byte reg + 2 bytes value = 3 bytes)
_CMD_BUF = bytearray(3)

class SensorDriver:
    def _build_cmd(self, reg: int, value: int) -> None:
        # pack_into writes to the pre-allocated buffer, zero heap allocation
        # 100 calls = 0 heap allocations, peak RAM = buffer size (3 bytes)
        struct.pack_into('>BH', _CMD_BUF, 0, reg, value)
        self._i2c.writeto(self._addr, _CMD_BUF)
```

**Complex Example (Multiple Command Formats):**
```python
import struct

# Pre-allocate buffers for multiple command formats
_CMD_SHORT = bytearray(2)   # Format 'BB'
_CMD_LONG  = bytearray(5)   # Format 'BHH'

class AdvancedDriver:
    def _send_short_cmd(self, reg: int, val: int) -> None:
        # Format 'BB': 2 unsigned bytes
        struct.pack_into('BB', _CMD_SHORT, 0, reg, val)
        self._i2c.writeto(self._addr, _CMD_SHORT)
    
    def _send_long_cmd(self, reg: int, val1: int, val2: int) -> None:
        # Format 'BHH': 1 byte + 2 16-bit unsigned integers
        struct.pack_into('BHH', _CMD_LONG, 0, reg, val1, val2)
        self._i2c.writeto(self._addr, _CMD_LONG)
```

**Rule Details:**
- The buffer size must match the `struct` format string (verify with `struct.calcsize()`)
- Naming follows semantic names like `_CMD_BUF`, `_PKT_BUF`, declared in the global variable area
- If the method already uses pre-allocated buffers (P0#1), check if they can be merged and reused
- Declare separate buffers for different command formats, do not use dynamic `bytearray(struct.calcsize(fmt))`

**Buffer Size Verification:**
```python
import struct

# Verify that the buffer size matches the format string
assert len(_CMD_BUF) == struct.calcsize('>BH'), "Buffer size mismatch"
# '>BH' = big-endian + 1 byte + 2 bytes = 3 bytes
```

**Memory Comparison (100 calls):**

| Method | Heap Allocations | Estimated Peak RAM | Temporary Objects |
|---|---|---|---|
| `struct.pack()` | 100 | ~300 bytes (fragmented) | 100 |
| `struct.pack_into()` | 0 | 3 bytes (fixed) | 0 |

---

### P2 — Optional

#### P2#8 `__slots__` Limiting Instance Attributes

**Judgment Criteria**: Driver classes have a fixed set of instance attributes (all declared in `__init__`, not dynamically added at runtime) — default Python classes use `__dict__` to store instance attributes (dictionary overhead about 200+ bytes/instance); `__slots__` uses a fixed array instead, saving about 50–200 bytes/instance.

**Key Constraint**: If a subclass inherits this class, the subclass must also declare `__slots__ = ()` (otherwise the subclass will still have `__dict__`)

```python
class SensorDriver:
    # Declare a fixed set of attributes, disable __dict__, save about 50-200 bytes/instance
    __slots__ = ('_i2c', '_addr', '_buf', '_mv', '_last_val')

    def __init__(self, i2c, addr: int = 0x68) -> None:
        self._i2c      = i2c
        self._addr     = addr
        self._buf      = bytearray(6)
        self._mv       = memoryview(self._buf)
        self._last_val = 0
```

**Subclass Inheritance Example:**
```python
class AdvancedSensor(SensorDriver):
    # Subclass must declare __slots__, otherwise the subclass will still have __dict__
    # Empty tuple means the subclass does not add new attributes, only inherits parent attributes
    __slots__ = ()
    
    def advanced_read(self) -> int:
        # Can access parent attributes normally
        return self._last_val * 2
```

**Subclass with New Attributes:**
```python
class ExtendedSensor(SensorDriver):
    # New attributes in the subclass must be declared in __slots__
    __slots__ = ('_calibration', '_offset')
    
    def __init__(self, i2c, addr: int = 0x68) -> None:
        super().__init__(i2c, addr)
        self._calibration = 1.0
        self._offset      = 0
```

**Rule Details:**
- `__slots__` must list all attribute names assigned with `self.xxx` in `__init__`
- If a subclass inherits this class, the subclass must also declare `__slots__ = ()` (otherwise the subclass will still have `__dict__`)
- If the driver class dynamically adds attributes at runtime (e.g., plugin-style extensions), this optimization is not applicable
- Attribute names must be string literals, dynamic attribute names are not supported

**Non-Applicable Scenarios:**

| Scenario | Is `__slots__` Applicable? | Reason |
|---|---|---|
| Driver class with a fixed set of attributes | Yes | Saves 50-200 bytes/instance |
| Dynamically adds attributes at runtime | No | `__slots__` prohibits dynamic attributes |
| Plugin system requiring `__dict__` | No | Plugins need to dynamically inject attributes |
| Singleton driver class (only 1 instance) | No | Savings are not significant (< 200 bytes) |

**Memory Savings Estimate:**
- Default class (with `__dict__`): about 200+ bytes/instance
- `__slots__` class: about 50 bytes/instance (only attribute array)
- Savings: about 150 bytes/instance (10 instances = 1.5KB)

#### P2#9 Generators Instead of Lists (Streaming Data)

**Judgment Criteria**: Methods return large lists (> 50 elements), and the caller processes elements one by one — returning a complete list requires allocating all memory at once; generators yield on demand, peak RAM is only the size of a single element.

**Incorrect (Prohibited, large list allocated at once):**
```python
def read_all_channels(self) -> list:
    # Allocate a list of 16 elements at once, peak RAM = 16 x element size
    # If each element is bytearray(6), peak RAM = 16 x 6 = 96 bytes + list overhead
    results = []
    for ch in range(16):
        results.append(self._read_channel(ch))
    return results

# Caller
driver = SensorDriver(...)
data = driver.read_all_channels()  # Peak RAM = complete list
for val in data:
    process(val)
```

**Correct (Generator, Peak RAM = Single Element):**
```python
def iter_channels(self):
    """
    Yield readings channel by channel (generator)
    Yields:
        bytearray: Raw reading for a single channel (6 bytes)
    Notes:
        - Peak RAM is only the size of a single reading (6 bytes), suitable for scenarios with > 16 channels
        - The caller must process one by one, random access is not supported (e.g., results[5])
    """
    for ch in range(16):
        yield self._read_channel(ch)

# Caller
driver = SensorDriver(...)
for val in driver.iter_channels():  # Peak RAM = single element (6 bytes)
    process(val)
```

**Compatibility Scheme (Keep Original API, Add Generator Version):**
```python
class SensorDriver:
    def read_all_channels(self) -> list:
        """
        Read all channels (returns a list, compatible with old code)
        Returns:
            list: Readings for all channels
        Notes:
            - Peak RAM = size of the complete list
            - It is recommended to use the iter_channels() generator version to reduce memory usage
        """
        return list(self.iter_channels())
    
    def iter_channels(self):
        """
        Yield readings channel by channel (generator, recommended)
        Yields:
            bytearray: Raw reading for a single channel
        Notes:
            - Peak RAM is only the size of a single reading
        """
        for ch in range(16):
            yield self._read_channel(ch)
```

**Rule Details:**
- Only applicable to scenarios where the caller **processes elements one by one**; if the caller needs random access (`results[5]`), it is not applicable
- Generator method naming is recommended with the `iter_` prefix to distinguish it from the method that returns a list
- If the original method name is a public API, keep the original method (returns a list), add the `iter_` version, and prompt in the description table
- Generators do not support `len()`, index access, or slicing operations

**Memory Comparison (16 channels, 6 bytes per channel):**

| Method | Peak RAM | Supports Random Access | Applicable Scenario |
|---|---|---|---|
| Returns a list | ~96 bytes + list overhead | Yes | Needs multiple traversals, random access |
| Generator | ~6 bytes (single element) | No | Only needs a single traversal, memory-constrained |

**Generator Applicability Scenario Judgment:**

| Scenario | Is Generator Applicable? | Reason |
|---|---|---|
| 16-channel ADC sampling, processing one by one | Yes | Peak RAM reduced by 94% |
| Batch reading 100 sensors | Yes | Peak RAM reduced from O(N) to O(1) |
| Needs random access `data[5]` | No | Generators do not support indexing |
| Needs to traverse the same data multiple times | No | Generators can only be traversed once |

---

## Optimization Effect Reference (Determine if Rewriting is Worthwhile)

| Optimization Method | Typical Scenario | Estimated RAM Savings | Rewrite Cost |
|---|---|---|---|
| Private `_CONST` (P0#2) | 10 module constants | **~400 bytes** | Low |
| `bytes` instead of `list` (P0#4) | 100 register addresses | **~700 bytes** | Low |
| Pre-allocated buffers (P0#1) | I2C/SPI read/write | **Eliminates peak heap allocation** | Low |
| `__slots__` (P2#8) | Single driver instance | **50–200 bytes/instance** | Low |
| `struct.pack_into()` (P1#7) | High-frequency command construction | Eliminates temporary bytes objects | Low |
| Avoid string `+` in loops (P0#3) | Log/report generation | Eliminates temporary string objects | Low |
| `gc.collect()` before (P1#5) | Batch read operations | Reduces GC trigger randomness | Low |
| `gc.disable()` protection (P1#6) | Timing-sensitive frame transmission | Prevents GC interruption | Medium |
| Generator instead of list (P2#9) | 16+ channel streaming read | **Peak RAM O(N)→O(1)** | Medium |

## Output Format

1. Output the complete optimized Python file content (code block preview).
2. Attach a brief description table:
   - **P0 Execution Status**: List all 4 items, mark "Executed" or "Not applicable (reason)"
   - **P1 Execution Status**: List the actually executed P1 items and the judgment basis (why applicable)
   - **P2 Execution Status**: List the actually executed P2 items and the judgment basis
   - **Overlap with upy-opt-driver**: If P0#1 has been handled by `upy-opt-driver`, note it here
   - **RAM Savings Estimate**: Cumulatively estimate the RAM bytes saved based on the optimization items
3. Ask the user: "Confirm writing to the original file?", and after user confirmation, overwrite the content to the original file.

**Description Table Example:**

```
Optimization Execution Status:

P0 Mandatory Items:
  ✓ P0#1 Pre-allocated Buffers: Executed (3 buffers, eliminated dynamic heap allocation)
  ✓ P0#2 Private _CONST: Executed (8 constants changed to private, saved about 320 bytes)
  ✓ P0#3 Avoid String + in Loops: Executed (build_report method changed to use .join())
  ✓ P0#4 bytes Instead of list: Executed (_REG_TABLE changed to bytes, saved about 56 bytes)

P1 Try-to-Modify Items:
  ✓ P1#5 gc.collect() Before: Executed (added before batch_read method)
  ✓ P1#7 struct.pack_into(): Executed (_build_cmd method changed to use pre-allocated buffer)
  ✗ P1#6 gc.disable() Protection: Not applicable (no timing-sensitive frame-level operations)

P2 Optional Items:
  ✓ P2#8 __slots__: Executed (SensorDriver class, saved about 150 bytes/instance)
  ✗ P2#9 Generator: Not applicable (no methods returning large lists)

Overlap with upy-opt-driver:
  - P0#1 Pre-allocated Buffers: Executed this time (upy-opt-driver has not processed this file)

RAM Savings Estimate:
  - Private constants: ~320 bytes
  - bytes instead of list: ~56 bytes
  - __slots__: ~150 bytes/instance
  - Eliminated temporary objects: Peak RAM reduced by about 40%
  - Total: about 526 bytes + peak optimization
```

---

## Complete Specification Reference

[Memory Optimization Guide (Local)](G:/GraftSense-Drivers-MicroPython#/MicroPython_Memory_Footprint_Minimization_Guide.md)

[Complete Driver Specification Document](https://github.com/FreakStudioCN/MicroPython_Skills/blob/main/upy_driver_dev_spec_summary.md)

## Introspection and Evolution

After each execution, check if the following situations are encountered:
- Edge cases not covered by the rules
- Output errors or rule defects pointed out by the user
- Newly discovered constraint requirements

If so, immediately execute:
1. Append the new rule to the corresponding section of this file
2. Synchronously write the same modification to `G:/MicroPython_Skills/upy-slim-driver/SKILL.md`
3. Execute in the `G:/MicroPython_Skills/` directory:
   `git add upy-slim-driver/SKILL.md && git commit -m "skill(upy-slim-driver): <rule description>"`
