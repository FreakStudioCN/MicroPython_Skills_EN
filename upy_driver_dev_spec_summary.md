# GraftSense-Drivers-MicroPython Driver Development Specification

> This document is compiled based on the actual code in the repository and the automated checking rules of `code_checker.py`. All specifications marked with "automated check" are enforced by pre-commit hooks.

---

## Table of Contents

1. [General Requirements](#1-general-requirements)
2. [File Header Format](#2-file-header-format)
3. [Module-Level Global Variables](#3-module-level-global-variables)
4. [File Section Structure](#4-file-section-structure)
5. [Detailed Explanation of 8 code_checker Rules](#5-detailed-explanation-of-8-code_checker-rules)
6. [Class Design Specification](#6-class-design-specification)
7. [Communication Protocol Implementation Patterns](#7-communication-protocol-implementation-patterns)
8. [Exception Handling Specification](#8-exception-handling-specification)
9. [Core Design and Implementation Details](#9-core-design-and-implementation-details)
10. [Import Specification](#10-import-specification)
11. [pre-commit Toolchain Configuration](#11-pre-commit-toolchain-configuration)
12. [package.json Specification](#12-packagejson-specification)
13. [Driver Directory Structure and Naming Convention](#13-driver-directory-structure-and-naming-convention)
14. [Comment Style](#14-comment-style)
15. [Instance Attribute Naming Convention](#15-instance-attribute-naming-convention)
16. [Function Design Specification](#16-function-design-specification)
17. [Type Annotation Specification](#17-type-annotation-specification)
18. [Manual Use of code_checker](#18-manual-use-of-code_checker)
19. [Specification Overview Quick Reference Table](#19-specification-overview-quick-reference-table)

---

## 1. General Requirements

- **Goal**: Driver code that is readable, reusable, resource-friendly (memory/flash), with complete documentation and test routines, suitable for MicroPython v1.x (Raspberry Pi Pico / ESP series, etc.).
- **Style**: Follows PEP8 (4-space indentation, lowercase with underscores for functions/variables), but with a greater focus on readability and memory allocation control in constrained environments.
- **File Header Comment**: Code must include a file header comment (environment, author, license, brief description, reference links, time). See [Section 2](#2-file-header-format).
- **Single Responsibility**: Each driver module corresponds to one chip/one type of peripheral. Avoid cramming multiple peripheral functions into the same file.
- **Parameter Validation**: Validate input parameters and raise meaningful exceptions (e.g., `ValueError`, `TypeError`). See [Section 6](#6-class-design-specification).
- **English Output + Type Annotations**: Except for comments, all `raise`/`print` statements must be in English. Use built-in type annotations supported by MicroPython (`: int`, `-> None`, etc.).

---

## 2. File Header Format

Every **non-`main.py`** driver file must start with the following fixed format:

```python
# Python env   : MicroPython v1.23.0
# -*- coding: utf-8 -*-
# @Time    : 2026/3/14 下午3:00
# @Author  : Author Name
# @File    : filename.py
# @Description : Brief description of module functionality
# @License : MIT
```

**Notes:**

- `# @License : MIT` must exist as an **independent comment line** and cannot be merged with other content (this is the exact match target for code_checker rule 2; missing it results in `[FAIL]`)
- `@Author` can be a Chinese or English name
- `@Description` content may contain Chinese

---

## 3. Module-Level Global Variables

Immediately after the file header, four required module-level global variables must exist (**code_checker rule 1, automated check**):

```python
__version__ = "1.0.0"
__author__ = "Author Name"
__license__ = "MIT"
__platform__ = "MicroPython v1.23"
```

> **Missing any one** will result in `[FAIL]`. `main.py` skips this check.

**Optional: `__chip__` variable**

If the driver depends on specific chip features (e.g., RP2040's PIO), declare it additionally:

```python
__chip__ = "RP2040"
```

Hardware adaptation can be done at runtime based on this:

```python
if "rp2040" in __chip__.lower():
    self._i2c = I2C(0, sda=Pin(0), scl=Pin(1))
elif "esp32" in __chip__.lower():
    self._i2c = I2C(scl=Pin(22), sda=Pin(21))
```

---

## 4. File Section Structure

All files must use fixed section marker comments. code_checker uses **fuzzy matching** for sections (ignoring the number of `=` and spaces), matching only the core title text.

### 4.1 Driver File (non-main.py) Structure

```python
# Python env   : MicroPython v1.23.0
# -*- coding: utf-8 -*-
# @Time    : YYYY/MM/DD HH:MM
# @Author  : Author Name
# @File    : filename.py
# @Description : Function description
# @License : MIT

__version__ = "1.0.0"
__author__ = "Author Name"
__license__ = "MIT"
__platform__ = "MicroPython v1.23"

# ======================================== Import Related Modules =========================================

# ... import statements ...

# ======================================== Global Variables ============================================

# ... constants / register address definitions ...

# ======================================== Utility Functions ============================================

# ... module-level utility functions (can be empty) ...

# ======================================== Custom Classes ============================================

# ... class definitions ...

# ======================================== Initialization Configuration ===========================================

# ========================================  Main Program  ===========================================
```

> The "Initialization Configuration" and "Main Program" sections at the end of the driver file can be left empty, but the **section markers must exist**.

### 4.2 Content Specification for Each Section

**Import Related Modules Section**

- Place: Standard library → MicroPython hardware modules → Third-party/local modules, grouped alphabetically or by usage frequency
- Do not place: Long delay operations (e.g., `time.sleep`), hardware instantiation

**Global Variables Section**

- Place: Module-level constants (I2C default address, register addresses), `DEBUG` switch, reusable buffers
- Do not place: Hardware object instantiation (`I2C()`, `Pin()`, etc.)

Reusable buffers can reduce runtime memory allocation:

```python
# Correct: Global reusable buffer
_BUF2 = bytearray(2)

def read_data():
    i2c.readfrom_mem_into(addr, reg, _BUF2)  # Zero memory allocation

# Incorrect: New buffer created on each call
def read_data():
    buf = bytearray(2)  # Allocates new memory each time
    i2c.readfrom_mem_into(addr, reg, buf)
```

**Utility Functions Section**

- Place: Module-level pure utility functions (no hardware I/O, good for unit testing), e.g., `clamp()`, address formatting
- Do not place: Functions that frequently create large objects, I/O operations that consume significant memory

**Custom Classes Section**

- Internal class order: Class-level constants → `__init__` → Public methods → Private methods (`_` prefix) → `deinit()`
- Avoid long delays in `__init__`; if detection/reset is needed, parameterize retries and delays

**Initialization Configuration Section (main.py)**

- Place: Hardware object instantiation, device scanning and address selection, initial parameter configuration, brief power-on self-test
- Leave this section empty in library modules; do not create hardware instances at import time (avoid import side effects)

**Main Program Section (main.py)**

- Place: Main flow logic, use `try/except/finally` to ensure safe peripheral exit
- Do not place: Infinite loops without exit conditions (example loops must provide an interruption method)

```python
# ========================================  Main Program  ===========================================
try:
    while True:
        # Main logic
        pass
except Exception as e:
    print("ERROR:", e)
finally:
    try:
        sensor.deinit()
    except Exception:
        pass
```

### 4.3 main.py Specific Structure

```python
# Python env   : MicroPython v1.23.0
# -*- coding: utf-8 -*-
# @Time    : YYYY/MM/DD HH:MM
# @Author  : Author Name
# @File    : main.py
# @Description : XXX sensor test file

# ======================================== Import Related Modules =========================================

from machine import I2C, Pin
import time
from bh_1750 import BH1750

# ======================================== Global Variables ============================================

# Only simple variable assignments allowed, no instantiation
bh_addr = None
SAMPLE_INTERVAL = 1000

# ======================================== Utility Functions ============================================

# ======================================== Custom Classes ============================================

# ======================================== Initialization Configuration ===========================================

# Must include time.sleep(3)
time.sleep(3)
# Must include a print starting with "FreakStudio:"
print("FreakStudio: test Light Intensity Sensor now")

# Object instantiation must be placed here
i2c = I2C(0, scl=Pin(5), sda=Pin(4), freq=100000)
sensor = BH1750(bh_addr, i2c)

# ========================================  Main Program  ===========================================

# while loops are only allowed in this section
while True:
    lux = sensor.measurement
    print(lux)
    time.sleep(1)
```

---

## 5. Detailed Explanation of 8 code_checker Rules

This is the core of the entire specification. All rules are automatically executed in `code_checker.py` via AST or regular expressions.

### Rule 1: Must contain 4 module-level global variables (non-main.py)

- **Check method:** AST parsing of `ast.Assign` nodes
- **Check content:** `__version__`, `__author__`, `__license__`, `__platform__`
- **main.py skips**

### Rule 2: Must have an independent `# @License : MIT` comment line (non-main.py)

- **Check method:** Line-by-line exact match (after `strip()`)
- Must be an independent line, cannot be written as `# @License : MIT, Apache`
- **main.py skips**

### Rule 3: No Chinese characters in raise/print (all files)

- **Check scope:** All lines containing `raise` or `print(`
- **Check content:** After extracting string literals, match with regex `[\u4e00-\u9fff]`
- **Comments and docstrings are not restricted**, only runtime strings are limited

```python
# Violation - Check fails
raise ValueError("参数不合法")
print("传感器初始化完成")

# Compliant - Passes
raise ValueError("parameter is invalid")
print("Sensor initialized successfully")
```

### Rule 4: main.py instantiation location check

- **Global Variables Section:** No instantiation allowed (matches patterns like `x = Class()`, `x = module.Class()`, `module.Class()`)
- **Initialization Configuration Section:** At least one instantiation must exist
- **Non-main.py skips**

### Rule 5: while loop location check (main.py)

- `while` loops are **only allowed** after the "Main Program" section
- Appearing in any other location is a violation
- Check uses regex `^\s*while\s+` for multi-line matching, stripping comments before checking
- **Non-main.py skips**

### Rule 6: Required content in Initialization Configuration Section (main.py)

- Must include `time.sleep(3)` (supports whitespace variations: `time.sleep ( 3 )`)
- Must include a print statement in the format `print("FreakStudio: ...")`
- **Non-main.py skips**

### Rule 7: `__init__` method parameter type annotations (all files)

- Use AST traversal to find the `__init__` method
- Check if parameters have `annotation` (i.e., `: Type` annotation)
- **Files without an `__init__` method skip this check**

### Rule 8: Parameter validation for class methods (non-main.py)

The checker traverses all methods of all classes (excluding `self`/`cls`). For methods with parameters, the method body must satisfy both conditions:

**Condition 1: At least one of the following:**
- `isinstance(param, Type)` call
- `hasattr(obj, "attr")` call
- Comparison expression (`==`, `!=`, `>`, `<`, `<=`, `>=`) or boolean combination (`and`/`or`)

**Condition 2:**
- The `if` block containing the above condition must have a `raise` statement

- **main.py skips**

---

## 6. Class Design Specification

### 6.1 Design Principles

- **Single Responsibility**: Each class encapsulates only one peripheral or a set of closely related functions
- **Minimal Side Effects**: Constructor does only necessary validation and lightweight initialization; provide separate methods for reset/calibration
- **Explicit Dependency Injection**: Do not create hardware bus objects (I2C/SPI/Pin) inside the class; pass them as parameters
- **Testability**: Extract pure logic (calibration/conversion) into pure functions; encapsulate I/O into small, easily mockable methods (`_read_reg`, `_write_reg`)
- **Exception Strategy**: Raise `ValueError` for parameter errors; raise `RuntimeError` or custom `DeviceError` for I/O/hardware errors

```python
# Incorrect: Hardware object created internally (bus cannot be reused, cannot be tested)
class MPU6050:
    def __init__(self):
        self.i2c = I2C(1)  # Hardcoded

# Correct: Dependency injection
class MPU6050:
    def __init__(self, i2c_bus):
        self.i2c = i2c_bus

shared_i2c = I2C(1)
sensor1 = MPU6050(shared_i2c)
sensor2 = BMP280(shared_i2c)  # Bus reuse
```

### 6.2 MicroPython Platform Features

- **Type Annotations**: Use only native types (`int`, `float`, `bytes`, `bytearray`, etc.) and `I2C`/`Pin`; write callback annotations as `callable`
- **Avoid Multiple Inheritance**: Prefer composition; multiple inheritance may cause memory issues on MicroPython
- **Pre-declare all attributes**: Declare all fields in `__init__`; use `__slots__` to fix attribute slots and reduce memory

```python
# Recommended: __slots__ for memory optimization
class Sensor:
    __slots__ = ('temp', 'humidity', '_buf')
    def __init__(self):
        self.temp = 0
        self.humidity = 0
        self._buf = bytearray(4)
```

### 6.3 Class Structure Layout Order

1. Class-level constants (`UPPER_CASE`)
2. `__init__` (parameter validation → attribute assignment → lightweight initialization)
3. Public API methods (ordered by common usage)
4. `@property` accessors
5. Private methods (`_` prefix)
6. `deinit()` / `close()` (resource cleanup, placed last)

### 6.4 Class-Level Docstring (Chinese and English)

```python
class BH1750:
    """
    This class is used to control the BH1750 digital ambient light sensor, supporting configuration of measurement mode, resolution, measurement time,
    and obtaining light intensity (lux) data.

    Attributes:
        _address (int): BH1750 I2C address.
        _i2c (I2C): machine.I2C instance for I2C communication.
        _measurement_mode (int): Measurement mode (continuous measurement or one-time measurement).
        _resolution (int): Resolution mode (high resolution, high resolution 2, low resolution).
        _measurement_time (int): Measurement time (range 31-254, default 69).

    Methods:
        configure(measurement_mode, resolution, measurement_time): Configure measurement parameters.
        reset(): Reset the sensor, clear registers.
        power_on(): Power on the sensor.
        power_off(): Power off the sensor.
        measurement -> float: Get current light intensity (lux).
        measurements(): Generator for continuous light intensity data.

    Notes:
        - I2C operations are not ISR-safe, please avoid calling in interrupts.
        - Sensor measurement time and resolution directly affect the lux calculation result.

    ==========================================

    BH1750 driver for digital ambient light sensor.

    Attributes:
        _address (int): I2C address of the sensor.
        _i2c (I2C): machine.I2C instance for bus communication.
        _measurement_mode (int): Measurement mode (continuous or one-time).
        _resolution (int): Resolution mode (high, high2, low).
        _measurement_time (int): Measurement time (31-254, default 69).

    Methods:
        configure(...): Configure measurement mode and resolution.
        reset(): Reset sensor, clear illuminance data register.
        power_on(): Power on the sensor.
        power_off(): Power off the sensor.
        measurement -> float: Get current lux reading.
        measurements(): Generator for continuous lux readings.

    Notes:
        - Methods performing I2C are not ISR-safe.
        - Lux calculation depends on configured resolution and measurement time.
    """
```

**Pattern:**
- Chinese first, English second
- Separated by `==========================================`
- Contains three standard sections: `Attributes`, `Methods`, `Notes`

### 6.5 Class-Level Constants (using `micropython.const()`)

```python
from micropython import const

class BH1750:
    MEASUREMENT_MODE_CONTINUOUSLY = const(1)
    MEASUREMENT_MODE_ONE_TIME     = const(2)

    RESOLUTION_HIGH   = const(0)
    RESOLUTION_HIGH_2 = const(1)
    RESOLUTION_LOW    = const(2)

    MEASUREMENT_TIME_DEFAULT = const(69)
    MEASUREMENT_TIME_MIN     = const(31)
    MEASUREMENT_TIME_MAX     = const(254)
```

Can also be defined as regular constants at the module top level (register addresses, etc.):

```python
DATA_FORMAT = 0x31
BW_RATE     = 0x2C
POWER_CTL   = 0x2D
INT_ENABLE  = 0x2E
OFSX        = 0x1E
OFSY        = 0x1F
OFSZ        = 0x20
```

### 6.6 `__init__` Method Specification

**Complete Example (ADXL345 Driver):**

```python
def __init__(self, bus: int, scl: int, sda: int, cs: Pin) -> None:
    """
    Initialize sensor and configure working parameters

    Args:
        bus (int): I2C bus number (usually 0 or 1)
        scl (int): I2C SCL pin number (valid GPIO pin number)
        sda (int): I2C SDA pin number (valid GPIO pin number)
        cs (Pin): Chip select pin object (configured Pin instance)

    Raises:
        ValueError: Any parameter is None or value out of range
        TypeError: Parameter type does not meet requirements
        OSError: I2C bus initialization failed or sensor not found

    Notes:
        Initialization flow: Configure chip select pin → Initialize I2C bus → Scan and match sensor address → Configure registers

    ==========================================
    Initialize sensor and configure working parameters

    Args:
        bus (int): I2C bus number (usually 0 or 1)
        scl (int): I2C SCL pin number (valid GPIO pin number)
        sda (int): I2C SDA pin number (valid GPIO pin number)
        cs (Pin): Chip select pin object (configured Pin instance)

    Raises:
        ValueError: Any parameter is None or value out of range
        TypeError: Parameter type does not meet requirements
        OSError: I2C bus initialization failed or sensor not found

    Notes:
        Initialization: Configure chip select → Initialize I2C bus → Scan and match sensor address → Configure registers
    """
    # Parameter validation (must be before logic code)
    if bus is None:
        raise ValueError("bus cannot be None")
    if not isinstance(bus, int):
        raise TypeError(f"bus must be int, got {type(bus).__name__}")
    if bus not in (0, 1):
        raise ValueError(f"bus must be 0 or 1, got {bus}")

    if scl is None:
        raise ValueError("scl cannot be None")
    if not isinstance(scl, int):
        raise TypeError(f"scl must be int, got {type(scl).__name__}")
    if scl < 0:
        raise ValueError(f"scl must be a valid GPIO pin number, got {scl}")

    if sda is None:
        raise ValueError("sda cannot be None")
    if not isinstance(sda, int):
        raise TypeError(f"sda must be int, got {type(sda).__name__}")
    if sda < 0:
        raise ValueError(f"sda must be a valid GPIO pin number, got {sda}")

    if cs is None:
        raise ValueError("cs cannot be None")
    if not isinstance(cs, Pin):
        raise TypeError(f"cs must be Pin object, got {type(cs).__name__}")

    # Initialization logic
    self.scl = scl
    self.sda = sda
    self.cs  = cs
    cs.value(1)
    time.sleep(1)
    self.i2c = I2C(bus, scl=self.scl, sda=self.sda, freq=10000)

    # Scan and match device
    slv = self.i2c.scan()
    for s in slv:
        buf = self.i2c.readfrom_mem(s, 0, 1)
        if buf[0] == 0xE5:          # ADXL345 device ID
            self.slvAddr = s
            print("adxl345 found")
            break

    # Configure registers
    self.writeByte(DATA_FORMAT, 0x2B)   # 16g range, full resolution
    self.writeByte(BW_RATE,     0x0A)   # 100Hz output rate
    self.writeByte(INT_ENABLE,  0x00)   # No interrupt used
    self.writeByte(OFSX,        0x00)
    self.writeByte(OFSY,        0x00)
    self.writeByte(OFSZ,        0x00)
    self.writeByte(POWER_CTL,   0x28)   # Measurement mode
    time.sleep(1)
```

**Requirements:**
1. All parameters must have type annotations (`: int`, `: Pin`, etc.)
2. Return type annotation `-> None`
3. Parameter validation must be written before initialization logic
4. Each parameter requires at least two validation steps: None check + type check, with value range check added when necessary

### 6.7 Regular Method Specification

**Complete Example:**

```python
def writeByte(self, addr: int, data: int) -> None:
    """
    Write a single byte of data to the specified register of the sensor

    Args:
        addr (int): Register address (8-bit, 0x00-0x3F)
        data (int): Byte data to be written (0-255)

    Raises:
        ValueError: Parameter is None or value out of range
        TypeError: Parameter type error
        OSError: I2C write failure

    Notes:
        Uses I2C memory write operation to write data to the specified register address

    ==========================================
    Write a single byte of data to the specified register of the sensor

    Args:
        addr (int): Register address (8-bit, 0x00-0x3F)
        data (int): Byte data to be written (0-255)

    Raises:
        ValueError: Parameter is None or value out of range
        TypeError: Parameter type error
        OSError: I2C write failure
    """
    # Parameter validation (required by code_checker rule 8)
    if addr is None:
        raise ValueError("addr cannot be None")
    if not isinstance(addr, int):
        raise TypeError(f"addr must be int, got {type(addr).__name__}")
    if addr < 0 or addr > 0x3F:
        raise ValueError(f"addr must be 0x00-0x3F, got {hex(addr)}")

    if data is None:
        raise ValueError("data cannot be None")
    if not isinstance(data, int):
        raise TypeError(f"data must be int, got {type(data).__name__}")
    if data < 0 or data > 255:
        raise ValueError(f"data must be 0-255, got {data}")

    d = bytearray([data])
    self.i2c.writeto_mem(self.slvAddr, addr, d)
```

### 6.8 Three Modes of Parameter Validation

**Mode 1: `isinstance` + `raise` (type check, most common)**

```python
if not isinstance(address, int):
    raise TypeError("address must be an integer")
```

**Mode 2: `hasattr` + `raise` (duck typing check, for verifying object interface)**

```python
# Avoid compatibility issues from directly importing machine.I2C, use duck-typing instead
if not hasattr(i2c, "writeto") or not hasattr(i2c, "readfrom_into"):
    raise TypeError("i2c must be a machine.I2C instance")
```

**Mode 3: Value range comparison + `raise`**

```python
if not (address == 0x23 or address == 0x5C):
    raise ValueError("address must be 0x23 or 0x5C")

if not (BH1750.MEASUREMENT_TIME_MIN <= measurement_time <= BH1750.MEASUREMENT_TIME_MAX):
    raise ValueError(
        "measurement_time must be between {0} and {1}".format(
            BH1750.MEASUREMENT_TIME_MIN, BH1750.MEASUREMENT_TIME_MAX
        )
    )
```

### 6.9 `@property` Decorator

Suitable for read-only data access methods:

```python
@property
def measurement(self) -> float:
    """
    Get current light intensity (lux).

    Returns:
        float: Light intensity in lux.

    Notes:
        If in one-time measurement mode, triggers a measurement when called.

    ==========================================

    Get current light intensity (lux).

    Returns:
        float: Light intensity in lux.

    Notes:
        If in one-time mode, triggers a measurement when called.
    """
    if self._measurement_mode == BH1750.MEASUREMENT_MODE_ONE_TIME:
        self._write_measurement_mode()

    buffer = bytearray(2)
    self._i2c.readfrom_into(self._address, buffer)
    lux = (buffer[0] << 8 | buffer[1]) / (1.2 * (BH1750.MEASUREMENT_TIME_DEFAULT / self._measurement_time))

    if self._resolution == BH1750.RESOLUTION_HIGH_2:
        return lux / 2
    return lux
```

### 6.10 Private Method Naming

Internal helper methods use a single underscore prefix:

```python
def _write_measurement_time(self):
    """
    Write measurement time to sensor registers.

    Notes:
        Internal method, not intended for direct use.

    ==========================================

    Write measurement time to sensor registers.

    Notes:
        Internal method, not intended for direct use.
    """
    buffer = bytearray(1)
    high_bit = 1 << 6 | self._measurement_time >> 5
    low_bit  = 3 << 5 | (self._measurement_time << 3) >> 3
    buffer[0] = high_bit
    self._i2c.writeto(self._address, buffer)
    buffer[0] = low_bit
    self._i2c.writeto(self._address, buffer)
```

### 6.11 Generator Method

```python
def measurements(self):
    """
    Generator for continuous light intensity measurements.

    Returns:
        generator: Returns float lux value on each iteration.

    Notes:
        Sleep time is calculated based on resolution and measurement time.

    ==========================================

    Generator for continuous light intensity measurements.

    Returns:
        generator: Returns float lux value on each iteration.

    Notes:
        Sleep time is calculated based on resolution and measurement time.
    """
    while True:
        yield self.measurement
        if self._measurement_mode == BH1750.MEASUREMENT_MODE_CONTINUOUSLY:
            base = 16 if self._resolution == BH1750.RESOLUTION_LOW else 120
            sleep_ms(math.ceil(base * self._measurement_time / BH1750.MEASUREMENT_TIME_DEFAULT))
```

### 6.12 Supplementary Method Comment Specification

**Side Effect Description**

If a method changes external state in addition to its main function, it must be clearly noted in Notes:

```python
def read_acceleration(self) -> tuple:
    """
    Read three-axis acceleration values.

    Notes:
        - Performs one I2C read transaction (6 bytes).
        - Updates internal `_last_read` timestamp during the call.
        - Implicitly wakes up the sensor if in sleep mode.

    ==========================================

    Notes:
        - Performs one I2C read transaction (6 bytes).
        - Updates internal `_last_read` timestamp.
        - Implicitly wakes up the sensor if in sleep mode.
    """
```

**ISR-Safe Annotation**

Clearly indicate in Notes whether the method can be safely called in an interrupt:

```python
def _handle_interrupt(self, pin) -> None:
    """
    Notes:
        - ISR-safe, can be safely invoked inside ISR.
        - Execution time < 50 μs. No memory allocation. No I2C/SPI.
    ==========================================
    Notes:
        - ISR-safe, can be safely invoked inside ISR.
        - Execution time < 50 μs. No memory allocation. No I2C/SPI.
    """

def calibrate_zero(self) -> None:
    """
    Notes:
        - Main-only, not ISR-safe. Contains blocking delay (10 ms). Initiates I2C write transaction.
    ==========================================
    Notes:
        - Main-only, not ISR-safe. Contains blocking delay (10 ms).
    """
```

**Callback Function Specification**

Write the callback signature and invocation context in Args:

```python
def start_listening(self, callback: callable) -> None:
    """
    Args:
        callback (callable): Invoked when a card is detected.
            Signature: ``def cb(card_id: int, uid: bytes) -> None``.
            Context: ISR. Execution < 100 μs. No memory allocation.
    ==========================================
    Args:
        callback (callable): Invoked when a card is detected.
            Signature: ``def cb(card_id: int, uid: bytes) -> None``.
            Context: ISR. Execution < 100 μs. No memory allocation.
    """
```

---

## 7. Communication Protocol Implementation Patterns

### 7.1 I2C Protocol (Most Common for Sensor Classes)

```python
from machine import I2C, Pin
import ustruct

# Initialization
self.i2c = I2C(bus, scl=Pin(scl), sda=Pin(sda), freq=10000)

# Device scan + ID verification
slv = self.i2c.scan()
for s in slv:
    buf = self.i2c.readfrom_mem(s, DEVICE_ID_REG, 1)
    if buf[0] == 0xE5:          # Compare device ID
        self.slvAddr = s
        print("Device found")
        break

# Write register (single byte)
d = bytearray([data])
self.i2c.writeto_mem(self.slvAddr, register_addr, d)

# Read register (single byte)
result = self.i2c.readfrom_mem(self.slvAddr, register_addr, 1)

# Direct byte stream read/write (no register address)
self.i2c.writeto(self._address, bytearray(b"\x01"))
buffer = bytearray(2)
self.i2c.readfrom_into(self._address, buffer)

# Data unpacking (little-endian signed 16-bit)
(value,) = ustruct.unpack("<h", buf)
```

### 7.2 UART Protocol (Common for Communication Modules)

```python
from machine import UART

# UART object passed as parameter (initialized externally, passed in for reusability)
self._uart = uart

# Send hexadecimal frame
self._uart.write(bytes.fromhex(cmd_hex_string))
time.sleep(0.05)

# Receive response
if self._uart.any():
    resp    = self._uart.read()
    tag     = resp[4]       # Protocol tag byte
    payload = resp[5:]      # Payload
    payload_hex = payload.hex()
```

### 7.3 Timer Callback Pattern (Watchdog / Periodic Task)

```python
from machine import Pin, Timer

# Initialize timer
self.wdi   = Pin(wdi_pin, Pin.OUT)
self.state = 0
self.timer = Timer(0)  # Use Timer(-1) only on RP2/Pico/RP2040/RP2350 or Zephyr.
self.timer.init(
    period=feed_interval,
    mode=Timer.PERIODIC,
    callback=self._feed
)

def _feed(self, t: Timer) -> None:
    """ISR-safe timer callback, feeds watchdog by toggling state"""
    self.state ^= 1
    self.wdi.value(self.state)
```

---

## 8. Exception Handling Specification

### 8.1 General Principles

- **Clear and Predictable**: Exceptions thrown externally should have stable semantics (parameter error, communication failure, device error, etc.)
- **Minimize Exposed Low-Level Exceptions**: Catch underlying `OSError`/`ValueError` and re-raise with clear custom or standard exceptions
- **Parameter Validation First**: Validate parameters at function entry and immediately raise `ValueError` to avoid entering an indeterminate state
- **Do Not Raise Exceptions in ISR**: Record error flags in ISR, handle exceptions in the main loop
- **Idempotent Cleanup**: Ensure resource safety in exception paths, call `deinit()` or restore default state in `finally`
- **Document**: Each method's docstring must list `Raises:` entries

**Example of Wrapping Low-Level Exceptions:**

```python
class DeviceError(RuntimeError):
    pass

class I2CDevice:
    def _read_reg(self, reg: int) -> int:
        try:
            return self.i2c.readfrom_mem(self.addr, reg, 1)[0]
        except OSError as e:
            raise DeviceError(f"Failed to read register 0x{reg:02X}") from e
```

**ISR Error Handling Example:**

```python
class Encoder:
    def __init__(self):
        self._error_flags = 0

    def _isr_handler(self, pin):
        try:
            # Core logic
            pass
        except Exception:
            self._error_flags |= 0x01  # Record error flag, do not raise exception

    def check_errors(self):
        if self._error_flags:
            raise DeviceError(f"Encoder error: flags={bin(self._error_flags)}")
```

### 8.2 Exception Classification

- **`ValueError`**: Parameter validation errors (index out of bounds, frequency not in allowed range, type error)
- **`RuntimeError` or custom `DeviceError`**: Hardware communication failures (I2C/SPI read/write failure, timeout, CRC check failure)
- **Custom exceptions** (e.g., `PCA9685Error`, `SensorError`): Unrecoverable or serious faults, allowing upper layers to distinguish by type
- **Return value or debug output**: Warnings/non-fatal situations
- **Error flags**: Errors in ISR, do not raise exceptions in ISR

### 8.3 Catching and Wrapping

Catch `OSError` in all underlying I/O methods and raise `DeviceError` or `RuntimeError`, preserving original information:

```python
def _write_reg(self, reg: int, value: int) -> None:
    try:
        self.i2c.writeto_mem(self.addr, reg, bytes([value]))
    except OSError as e:
        raise DeviceError(f"Failed to write register 0x{reg:02X}") from e
```

### 8.4 Retry Mechanism

Implement limited retries (2-3 times) for transient I2C/SPI errors, providing optional parameters `retries=1, delay_ms=5`:

```python
def read_with_retry(self, reg: int, retries: int = 2) -> int:
    for attempt in range(retries + 1):
        try:
            return self._read_reg(reg)
        except DeviceError:
            if attempt == retries:
                raise
            time.sleep_ms(5)
```

### 8.5 Resource Cleanup

Use `try/except/finally` to ensure the peripheral is placed in a safe state:

```python
def move_to_position(self, pos: int):
    self.enable()
    try:
        self._set_target(pos)
        self._wait_until_done(timeout=10.0)
    except TimeoutError:
        self.emergency_stop()
        raise
    finally:
        self._release_brake()
        self._set_power(0)
```

### 8.6 Debug Output

Avoid using large logging libraries; use a lightweight debug switch to control `print()`:

```python
def _log_error(self, msg: str):
    if self.debug:
        print(f"[ERROR] {msg}")
```

Exception messages should be concise and include key information (function/register/channel/address):

```
"I2C write failed reg=0x06 ch=3: [Errno 5]"
```

---

## 9. Core Design and Implementation Details

Sensor driver development revolves around the core principles of "cohesive hardware logic, clean external interface, comprehensive exception handling, and closed-loop resource management," covering all dimensions including function separation, parameter management, and hardware adaptation.

### 9.1 General Function Separation

Extract hardware-independent, reusable general functions (data format conversion, CRC check, unit conversion, etc.) and declare them outside the class to reduce class coupling:

```python
# General utility functions outside the class
def sht30_crc8(data: bytes) -> int:
    POLYNOMIAL = 0x31
    crc = 0xFF
    for byte in data:
        crc ^= byte
        for _ in range(8):
            crc = (crc << 1) ^ POLYNOMIAL if crc & 0x80 else crc << 1
        crc &= 0xFF
    return crc

def raw_temp_to_celsius(raw_temp: int) -> float:
    return -45.0 + 175.0 * raw_temp / 65535.0

def raw_hum_to_percent(raw_hum: int) -> float:
    return 100.0 * raw_hum / 65535.0

# Sensor class only handles hardware interaction
class SHT30:
    def __init__(self, i2c, addr: int = 0x44):
        self.i2c = i2c
        self.addr = addr
```

### 9.2 Custom Exception Classes

Define specific exception classes for typical exception scenarios to improve error localization efficiency:

```python
class SensorError(Exception):
    pass

class SensorCommunicationError(SensorError):
    def __init__(self, msg: str = "Sensor communication failed"):
        super().__init__(msg)

class SensorInvalidParamError(SensorError):
    def __init__(self, param: str, value, valid_range: tuple):
        super().__init__(f"Invalid param {param}={value}, valid: {valid_range}")

class SensorDataError(SensorError):
    def __init__(self, msg: str = "Sensor data validation failed"):
        super().__init__(msg)
```

### 9.3 Parameter Management: Class Attributes for Registers/Commands/Default Parameters

Declare fixed hardware parameters and default configurations as class attributes (uppercase constants) to avoid hardcoding:

```python
class SHT30:
    CMD_MEASURE_HIGH_REP = b'\x2C\x06'
    CMD_SOFT_RESET       = b'\x30\xA2'
    DEFAULT_SAMPLING_RATE = 1000   # ms
    TEMP_RANGE = (-45.0, 125.0)
    HUM_RANGE  = (0.0, 100.0)
```

### 9.4 Private/Public Method Division

- Public methods: API exposed externally (`init`, `read_temp_hum`, `soft_reset`)
- Private methods (`_` prefix): Low-level hardware interaction (`_read_raw_data`, `_parse_raw_data`, `_validate_data`)

```python
class SHT30:
    def read_temp_hum(self) -> tuple:
        raw = self._read_raw_data()
        temp, hum = self._parse_raw_data(raw)
        self._validate_data(temp, hum)
        return temp, hum

    def _read_raw_data(self) -> bytes:
        try:
            return self.i2c.readfrom(self.addr, 6)
        except OSError as e:
            raise SensorCommunicationError(f"I2C read failed: {e}") from e

    def _validate_data(self, temp: float, hum: float) -> None:
        if not (self.TEMP_RANGE[0] <= temp <= self.TEMP_RANGE[1]):
            raise SensorDataError(f"Temperature {temp} out of range {self.TEMP_RANGE}")
```

### 9.5 Interrupt Adaptation: micropython.schedule

Interrupt callback functions should only perform lightweight operations. Use `micropython.schedule` to schedule core processing logic to the main loop:

```python
import micropython
from machine import Pin

class SHT30:
    def __init__(self, i2c, addr: int = 0x44, int_pin: int = None):
        self._int_flag = False
        if int_pin is not None:
            self._int_pin = Pin(int_pin, Pin.IN, Pin.PULL_UP)
            self._int_pin.irq(trigger=Pin.IRQ_FALLING, handler=self._int_callback)

    def _int_callback(self, pin):
        micropython.schedule(self._handle_interrupt, None)

    def _handle_interrupt(self, _):
        if not self._int_flag:
            self._int_flag = True
            try:
                temp, hum = self.read_temp_hum()
                print(f"Interrupt: {temp:.2f}C, {hum:.2f}%")
            except SensorError as e:
                print(f"Interrupt failed: {e}")
            self._int_flag = False
```

### 9.6 Non-blocking Execution: Timer for Periodic Sampling

Avoid `time.sleep()` blocking the main loop; use `machine.Timer` for non-blocking periodic sampling:

```python
from machine import Timer

class SHT30:
    def start_periodic_sampling(self, interval: int = None):
        interval = interval or self.DEFAULT_SAMPLING_RATE
        self._timer = Timer(0)  # Use Timer(-1) only on RP2/Pico/RP2040/RP2350 or Zephyr.
        self._timer.init(period=interval, mode=Timer.PERIODIC, callback=self._sampling_callback)

    def stop_periodic_sampling(self):
        if self._timer:
            self._timer.deinit()
            self._timer = None

    def _sampling_callback(self, timer):
        try:
            self._latest_temp, self._latest_hum = self.read_temp_hum()
        except SensorError as e:
            print(f"Sampling failed: {e}")
```

### 9.7 Resource Management: Context Manager

Implement `__enter__`/`__exit__` to support automatic resource management with the `with` statement:

```python
class SHT30:
    def deinit(self):
        if self._timer:
            self._timer.deinit()
            self._timer = None
        if self._int_pin:
            self._int_pin.irq(handler=None)
            self._int_pin = None
        self._is_init = False

    def __enter__(self):
        self.init()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.deinit()
        return False

# Usage example
with SHT30(i2c, addr=0x44) as sht30:
    sht30.start_periodic_sampling(1000)
    # Resources are automatically released on exit
```

### 9.8 Parameter Configuration: Setter/Getter Encapsulation

Prohibit direct external modification of configuration attributes; encapsulate and validate through `set_xxx()`/`get_xxx()` methods:

```python
class SHT30:
    def set_sampling_rate(self, rate: int) -> None:
        if not (100 <= rate <= 10000):
            raise SensorInvalidParamError("sampling_rate", rate, (100, 10000))
        self._sampling_rate = rate
        if self._timer:
            self.stop_periodic_sampling()
            self.start_periodic_sampling(rate)

    def get_sampling_rate(self) -> int:
        return self._sampling_rate
```

### 9.9 Platform Compatibility

Automatically adapt default pins for different hardware platforms using `sys.platform`:

```python
import sys

class SHT30:
    def __init__(self, i2c=None, addr: int = 0x44):
        if i2c is None:
            platform = sys.platform
            if platform == "esp32":
                i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=400000)
            elif platform == "rp2":
                i2c = I2C(0, scl=Pin(1), sda=Pin(0), freq=400000)
            elif platform == "esp8266":
                i2c = I2C(scl=Pin(5), sda=Pin(4), freq=100000)
            else:
                raise SensorError(f"Unsupported platform: {platform}")
        self.i2c = i2c
```

### 9.10 Data Debouncing and Caching

Cache the latest valid data to avoid frequent hardware reads; support averaging over multiple consecutive samples:

```python
class SHT30:
    DEBOUNCE_COUNT = 3
    CACHE_EXPIRE   = 500  # ms

    def read_temp_hum(self, debounce: bool = True) -> tuple:
        now = time.ticks_ms()
        if (self._cache_temp is not None and
                time.ticks_diff(now, self._cache_time) < self.CACHE_EXPIRE):
            return self._cache_temp, self._cache_hum

        if debounce:
            temps, hums = [], []
            for _ in range(self.DEBOUNCE_COUNT):
                t, h = self._read_once()
                temps.append(t); hums.append(h)
                time.sleep_ms(10)
            temp = sum(temps) / len(temps)
            hum  = sum(hums)  / len(hums)
        else:
            temp, hum = self._read_once()

        self._cache_temp, self._cache_hum, self._cache_time = temp, hum, now
        return temp, hum
```

---

### 9.11 Type Annotations and Docstrings

Function signatures use MicroPython native type annotations; docstrings explain parameters, return values, and exceptions:

```python
def read_temp_hum(self, debounce: bool = True) -> tuple:
    """
    Read temperature and humidity values (supports debouncing and caching)
    Args:
        debounce: Whether to enable debouncing (default True)
    Returns:
        tuple: (temperature ℃, humidity %RH)
    Raises:
        SensorError: Sensor not initialized
        SensorCommunicationError: I2C communication failed
    """
```

---

### 9.12 Singleton Pattern

Avoid re-initializing hardware resources; use class variable `_instance` to implement singleton:

```python
class SHT30:
    _instance = None

    @staticmethod
    def get_instance(i2c=None, addr=0x44):
        if SHT30._instance is None:
            SHT30._instance = SHT30(i2c, addr)
        return SHT30._instance
```

---

## 10. Test File main.py Requirements

### 10.1 Core Design Principles

Test files follow the principle of "full coverage, flexible debugging, safe and controllable":

- **Full Coverage**: Completely cover all APIs of the driver library, breaking down interfaces by core functional dimensions of the chip type:

| Chip Type | Core API Functional Dimensions |
|---|---|
| Sensor Class | Basic status query, core data acquisition, parameter configuration, mode switching, calibration compensation |
| Motor Driver Class | Hardware initialization, motion control, status reading, reset/sleep |
| Communication Module Class | Network/protocol configuration, data transmission/reception, status query, power control |
| Storage Chip Class | Data read/write, address configuration, erase/reset |
| GPIO/Bus Expander Class | Pin configuration, level read/write, interrupt configuration |

Cover three types of test scenarios: normal parameters, boundary parameters, and abnormal parameters:

| Test Scenario Type | Core Validation Goal |
|---|---|
| Normal Parameter Scenario | Validate basic usability of API under default/common parameters |
| Boundary Parameter Scenario | Validate API's adaptability to hardware limit parameters |
| Abnormal Parameter/Environment Scenario | Validate API's fault tolerance and error feedback capability |

Code handling methods for different API characteristics:

| API Characteristic Type | Code Handling Method |
|---|---|
| Low-frequency Core API | Keep auto-execution logic, call periodically in main loop |
| High-frequency Update API | Keep function definition, comment out auto-execution, for REPL manual call |
| Mode Switching API | Keep call code, comment out auto-execution, for REPL manual trigger |
| Batch Operation API | Encapsulate as batch test function, for REPL one-click call |

- **Flexible Debugging**: High-frequency update/mode switching functions are commented out by default for auto-execution; only function definitions are kept for REPL manual calls.
- **Safe and Controllable**: Standardized initialization, exception catching, resource cleanup, ensuring no resource leaks, hardware hangs, etc.

---

### 10.2 Initialization Configuration Section

Must include a 3-second power-on delay and a fixed-format debug print before creating hardware objects:

```python
# ======================================== Initialization Configuration ==========================================

# Power-on delay 3s (mandatory, do not delete)
time.sleep(3)
# Print debug message (unified format: FreakStudio: Using + hardware/driver name + function description)
print("FreakStudio: Using R60ABD1 millimeter wave information collection")

uart = UART(0, baudrate=115200, tx=Pin(16), rx=Pin(17), timeout=0)
processor = DataFlowProcessor(uart)
device = R60ABD1(processor, parse_interval=200)
```

---

### 10.3 File Top Metadata Comment Specification

```python
# Python env   : MicroPython v1.23.0
# -*- coding: utf-8 -*-
# @Time    : 2025/11/4 下午5:33
# @Author  : Li Qingshui
# @File    : main.py
# @Description : Test code for R60ABD1 radar device driver class
# @License : CC BY-NC 4.0
```

---

### 10.4 Fixed Comment Block Order

File content strictly follows the following block order, each block using a uniform separator `# ======================================== Block Name =========================================`:

```python
# ======================================== Import Related Modules =========================================
from machine import UART, Pin, Timer
import time
from data_flow_processor import DataFlowProcessor
from r60abd1 import R60ABD1, format_time

# ======================================== Global Variables ============================================
last_print_time = time.ticks_ms()
print_interval = 2000

# ======================================== Utility Functions ============================================
def print_report_sensor_data():
    """Print high-frequency reported sensor data (changes quickly, call commented out by default)"""
    ...

# ======================================== Custom Classes ============================================
# (Can be left empty or commented if no custom classes)

# ======================================== Initialization Configuration ==========================================
time.sleep(3)
print("FreakStudio: Using R60ABD1 millimeter wave information collection")
uart = UART(0, baudrate=115200, tx=Pin(16), rx=Pin(17), timeout=0)
...

# ========================================  Main Program  ===========================================
try:
    while True:
        ...
        # print_report_sensor_data()  # High-frequency print function, commented out by default, for REPL manual call
except KeyboardInterrupt:
    ...
finally:
    ...
```

---

### 10.5 Exception Handling and Safe Shutdown Specification

```python
# ========================================  Main Program  ===========================================
try:
    while True:
        current_time = time.ticks_ms()
        if time.ticks_diff(current_time, last_print_time) >= print_interval:
            success, presence_status = device.query_presence_status()
            if success:
                print("Presence Status: %s" % ("Someone" if presence_status == 1 else "No one"))
            else:
                print("Query Presence Status failed")
            last_print_time = current_time
            time.sleep(0.2)
        time.sleep_ms(10)

except KeyboardInterrupt:
    print("%s Program interrupted by user" % format_time())
except OSError as e:
    print("%s Hardware communication error: %s" % (format_time(), str(e)))
except Exception as e:
    print("%s Unknown error: %s" % (format_time(), str(e)))

finally:
    print("%s Cleaning up resources..." % format_time())
    device.close()
    del device
    del uart
    print("%s Program exited" % format_time())
```

---

## 11. README.md Writing Specification

### 11.1 Basic Structure Requirements

README.md must include the following sections (in order):

| Section | Description |
|---|---|
| Project Title | `# [Sensor/Peripheral Name] MicroPython Driver` |
| Description | Brief introduction to the driver's purpose, main features, and applicable scenarios |
| Key Features | List of feature highlights |
| Hardware Requirements | Test hardware table + wiring example |
| Software Environment | Firmware version, driver version, dependencies |
| File Structure | File tree structure |
| File Description | Explanation of each file's purpose |
| Design Approach | Driver implementation method or structural approach (optional) |
| Quick Start | Step-by-step usage instructions + code example |
| Notes | Chip operating conditions, usage limitations, compatibility, etc. |
| Version History | Table format: version number, date, author, change description |
| Contact Developer | Email or other contact information |
| License | CC BY-NC 4.0, distinguish between official modules (MIT) and self-written drivers |

---

### 11.2 Project Title and Description

```markdown
# RCWL9623 Transceiver Integrated Ultrasonic Module Driver - MicroPython Version

## Introduction
Ultrasonic distance measurement module based on the RCWL9623 chip, supporting multiple communication modes (GPIO, 1-Wire, UART, I2C).
Widely used in robot obstacle avoidance, smart home distance measurement, safety monitoring, and other scenarios.

> **Note**: Not suitable for high-precision applications such as safety rescue or other special occasions.
```

---

### 11.3 Key Features

List driver feature highlights using bullet points:

```markdown
## Key Features
- **Multiple Operating Mode Support**: GPIO trigger/Echo return, single bus (1-Wire), UART, I2C
- **Measurement Range**: Approximately 20cm to 7m, returns invalid value if out of range
- **Unified Unit**: Unit is uniformly centimeters (cm)
- **Simple and Easy to Use**: Provides a unified interface `read_distance()`
- **Cross-platform Support**: Compatible with multiple MicroPython-compatible development boards
```

---

### 11.4 Hardware Requirements

Include recommended test hardware and pin description table:

```markdown
## Hardware Requirements
### Recommended Test Hardware
- Raspberry Pi Pico/Pico W
- RCWL9623 Ultrasonic Module

### Module Pin Description
| Pin  | Function Description |
|-------|--------------------|
| VCC   | Power supply positive (3.3V-5V)|
| GND   | Power supply negative           |
| TRIG  | Trigger pin (GPIO mode)|
| ECHO  | Echo pin (GPIO mode)|
| SCL/SDA | I2C communication pins     |
```

---

### 11.5 File Structure and File Description

```markdown
## File Structure
├── rcwl9623.py   # Core driver
├── main.py       # Test example
└── README.md     # Documentation

## File Description
- `rcwl9623.py`: Core driver, implements four operating modes
- `main.py`: Example main program, reads distance in a loop
- `README.md`: Documentation
```

---

### 11.6 Quick Start

Step-by-step usage instructions with code example:

```markdown
## Quick Start
1. Upload `rcwl9623.py` to the development board
2. Wire according to hardware requirements
3. Run the example program:

\`\`\`python
from machine import Pin
from rcwl9623 import RCWL9623

sensor = RCWL9623(mode=RCWL9623.GPIO_MODE, gpio_pins=(5, 4))
print(sensor.read_distance())
\`\`\`
```

---

### 11.7 License

Clearly distinguish between official modules and self-written driver licenses:

```markdown
## License
In this project, except for MicroPython official modules such as `machine` (MIT license),
all driver and extension code written by the author is released under the
**Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)** license.

**Copyright belongs to FreakStudio.**
```

---

## 12. Import Specification

```python
# 1. MicroPython machine-related modules
from machine import Pin, I2C, UART, Timer, SPI

# 2. MicroPython standard modules
import time
from time import sleep_ms
import ustruct
import math

# 3. micropython specific modules
from micropython import const

# 4. Third-party / project internal dependencies
from pca9685 import PCA9685
```

> **About `const()`:** `const()` is an undefined name in standard Python and will trigger flake8's `F821` error. It is widely used in the project, and the flake8 configuration does not suppress it. If this error occurs during commit, manually add `# noqa: F821` or add an ignore rule in the flake8 configuration.

---

## 13. pre-commit Toolchain Configuration

**`.pre-commit-config.yaml`:**

```yaml
repos:
  - repo: git@github.com:psf/black.git
    rev: 24.3.0
    hooks:
      - id: black
        args: [--line-length=150]         # Maximum line length 150 characters

  - repo: git@github.com:pycqa/flake8.git
    rev: 7.0.0
    hooks:
      - id: flake8
        args: ["--ignore=E203,E501,W503", "--max-line-length=150"]

  - repo: local
    hooks:
      - id: run-code-checker
        name: Run Code Checker
        entry: python code_checker.py
        language: system
        pass_filenames: true
        types: [python]
```

**Ignored Flake8 Error Codes:**

| Error Code | Meaning |
|--------|------|
| E203   | Whitespace before colon (Black formatting can produce this) |
| E501   | Line too long (already handled by Black) |
| W503   | Line break before binary operator |

**Workflow After Black Reformats:**

```bash
# Black modifies the file and returns a non-zero exit code, causing the commit to fail
# Solution: Re-stage the modified files and commit again
git add <files formatted by Black>
git commit -m "Same commit message as before"
```

**Emergency Hook Bypass (for emergencies only, must restore afterwards):**

```bash
# Bypass
git config --local core.hooksPath NUL
# Restore
git config --local --unset core.hooksPath
```

---

## 14. package.json Specification

Each driver directory must have a `package.json` at its root:

```json
{
  "name": "adxl345_driver",
  "version": "1.0.1",
  "description": "A MicroPython library to control adxl345_driver",
  "author": "leeqingshui",
  "license": "MIT",
  "chips": "all",
  "fw": "all",
  "_comments": {
    "chips": "Chip models supported by this package, all means no chip restrictions",
    "fw": "Specific firmware dependencies like ulab, lvgl, all means no firmware dependencies"
  },
  "urls": [
    ["adxl345.py", "code/adxl345.py"]
  ]
}
```

**Field Description:**

| Field | Type | Required | Description |
|------|------|------|------|
| `name` | String | ✅ | Package name, lowercase letters + underscores, consistent with directory name |
| `urls` | 2D Array | ✅ | File mapping: `[["target filename", "source path"], ...]` |
| `version` | String | ✅ | Semantic versioning `major.minor.patch` |
| `_comments` | Object | ❌ | Comment field, explains chips/fw meaning, does not affect parsing |
| `description` | String | ❌ | Function description (English) |
| `author` | String | ❌ | Author information, must be consistent with original author when referencing others' code |
| `license` | String | ❌ | License, follow original repository when referencing others' code, default MIT for original work |
| `chips` | String/Array | ❌ | Supported chips: `"all"` or `["ESP32", "STM32F407"]` |
| `fw` | String/Array | ❌ | Firmware dependencies: `"all"` or `["ulab", "lvgl"]` |
| `deps` | Array | ❌ | Dependency package list, see 14.4 |

**Multi-file Driver Example:**

```json
"urls": [
  ["bus_dc_motor.py", "code/bus_dc_motor.py"],
  ["pca9685.py",      "code/pca9685.py"]
]
```

**Three Installation Methods:**

```python
# Method 1: mip (run on device)
import mip
mip.install("github:FreakStudioCN/GraftSense-Drivers-MicroPython/sensors/adxl345_driver")

# Method 2: mpremote (command line)
# mpremote mip install github:FreakStudioCN/GraftSense-Drivers-MicroPython/sensors/adxl345_driver

# Method 3: upypi (recommended, visit https://upypi.net/ to search for package name and get command)
```

### 14.3 Example File (bmp280_driver)

```json
{
  "name": "bmp280_driver",
  "urls": [
    ["bmp280_float.py", "code/bmp280_float.py"]
  ],
  "version": "1.0.0",
  "_comments": {
    "chips": "Chip models supported by this package, all means no chip restrictions",
    "fw": "Specific firmware dependencies like ulab, lvgl, all means no firmware dependencies"
  },
  "description": "A MicroPython library to control BMP280 pressure sensor",
  "author": "robert-hh",
  "license": "MIT",
  "chips": "all",
  "fw": "all"
}
```

> This driver references robert-hh's open-source code; the `author` and `license` fields are consistent with the original repository.

### 14.4 deps Dependency Field

Use the `deps` field when there are external dependencies:

```json
{
  "urls": [
    ["mlx90640/__init__.py", "mlx90640/__init__.py"],
    ["mlx90640/utils.py", "mlx90640/utils.py"]
  ],
  "deps": [
    ["collections-defaultdict", "latest"],
    ["os-path", "latest"],
    ["github:org/micropython-additions", "main"],
    ["gitlab:org/micropython-otheradditions", "main"]
  ],
  "version": "0.2"
}
```

Third-party platform dependency example:

```json
{
  "name": "xfyun_asr",
  "version": "1.0.1",
  "description": "iFlytek online ASR WebSocket driver for MicroPython",
  "author": "leeqingsui",
  "license": "MIT",
  "chips": "all",
  "fw": "all",
  "deps": [
    ["https://upypi.net/pkgs/async_websocket_client/1.0.0", "latest"]
  ],
  "urls": [
    ["xfyun_asr.py", "code/xfyun_asr.py"]
  ]
}
```

### 14.5 License and Copyright Notice Notes

- **Referencing others' code**: The `author` and `license` fields must be consistent with the original repository; if the original repository has no license, use MIT and note the reference source in README.md.
- **Original code**: Use FreakStudio copyright notice, follow MIT protocol.

---

## 15. Driver Directory Structure and Naming Convention

Standard directory structure for each driver module:

```
sensors/adxl345_driver/
├── code/
│   ├── adxl345.py      # Driver implementation (main file)
│   └── main.py         # Usage example / test code
├── package.json         # mip package configuration
├── README.md            # Usage documentation (Chinese)
└── LICENSE              # MIT license file
```

**Category Directories (9 categories):**

| Directory | Content |
|------|------|
| `sensors/` | Sensors (temperature/humidity, IMU, gas, light, distance, ECG, etc.) |
| `communication/` | Communication modules (NFC, 4G, Bluetooth, RF, Ethernet) |
| `motor_drivers/` | Motor drivers (DC, stepper, servo) |
| `lighting/` | Display and lighting (LED, OLED, LCD, 7-segment display) |
| `input/` | Input devices (buttons, touch screen, encoder, joystick) |
| `signal_acquisition/` | Signal acquisition (ADC modules) |
| `signal_generation/` | Signal generation (waveform generator, DAC) |
| `storage/` | Storage (EEPROM, SD card) |
| `misc/` | Miscellaneous (RTC, audio, relay, I2C multiplexer) |

### 15.1 Folder Naming Convention

- Use lowercase letters + underscores, format: `[sensor_name]_driver` or `[function]_driver`
- Example: `pca9685_driver`, `bus_dc_motor_driver`

**Purpose of Each File/Folder:**

| File / Folder | Purpose Description |
|--------------|---------|
| `sensor_driver/` | Project root directory, naming follows `[sensor_name]_driver` convention, serves as container for the entire driver package |
| `code/` | Source code root directory, acts as the code execution root when flashed to the device |
| `code/sensor.py` | Driver implementation file, named after the sensor, encapsulates core logic like initialization, data reading, configuration |
| `code/main.py` | Test/example entry file (can also be named `demo.py`), provides usage examples and functional verification code |
| `package.json` | Package configuration file, defines name, version, file mapping, dependencies, compatible chips/firmware and other metadata, used for `mip` installation |
| `README.md` | Project documentation, includes driver functionality, installation method, API reference, notes, etc. |
| `LICENSE` | Open-source license file, clarifies usage, modification, and distribution rules (MIT) |

### 15.2 Driver File Naming Convention

**Four Rules:**

1. **Lowercase letters + underscores**: File names are all lowercase, words separated by underscores
2. **Based on sensor/chip model**: Prioritize using sensor model or chip model for naming
3. **Concise and clear**: File name accurately reflects driver functionality, avoid vague naming
4. **Avoid conflicts**: Do not use names that conflict with MicroPython built-in modules

**Naming Examples:**

| Sensor/Module | Standard Name | Description |
|------------|---------|------|
| PCA9685 (PWM Controller) | `pca9685.py` | Directly use chip model |
| BMP280 (Pressure Sensor) | `bmp280.py` | Directly use sensor model |
| DHT11 (Temperature/Humidity Sensor) | `dht11.py` | Specific to model, avoid confusion with `dht22` |
| L298N (Motor Driver) | `l298n.py` | Prioritize using chip model |
| HC-SR04 (Ultrasonic) | `hc_sr04.py` | Convert hyphens to underscores |
| MPU6050 (IMU) | `mpu6050.py` | Directly use chip model |
| WS2812B (RGB LED Strip) | `ws2812b.py` | Avoid using built-in module name `neopixel` |
| DS18B20 (Temperature Sensor) | `ds18b20.py` | Avoid using built-in module name `ds18x20` |
| Thermistor (No model) | `analog_temp.py` | Name by function, ensure no conflict with built-in modules |

**MicroPython Built-in Module Names to Avoid:**

```
machine, time, os, sys, math, network,
uos, usys, utime, ubinascii, ustruct, ucollections,
uhashlib, uheapq, uio, ujson, ure, uzlib,
uarray, uasyncio, ucryptolib, uctypes,
neopixel, dht, ds18x20, onewire
```

---

## 16. Comment Style

**Inline comments** in driver files should all be in Chinese, with a high comment density to improve readability:

```python
# Assign SCL pin number
self.scl = scl
# Assign SDA pin number
self.sda = sda
# Set chip select pin to high level
cs.value(1)
# Delay 1 second for hardware stabilization
time.sleep(1)
# Initialize I2C bus communication object
self.i2c = I2C(bus, scl=self.scl, sda=self.sda, freq=10000)
# Scan for slave addresses on the I2C bus
slv = self.i2c.scan()
```

Docstrings use **Chinese and English bilingual**, inline comments use **Chinese**.

---

## 17. Instance Attribute Naming Convention

| Scenario | Naming Style | Example |
|------|---------|------|
| Public attributes | `camelCase` or `snake_case` | `self.slvAddr`, `self.scl`, `self.sda` |
| Private attributes | `_snake_case` (single underscore prefix) | `self._address`, `self._i2c`, `self._measurement_mode` |
| Class-level constants | `UPPER_CASE` | `MEASUREMENT_MODE_CONTINUOUSLY`, `RESOLUTION_HIGH` |
| Module-level register constants | `UPPER_CASE` | `DATA_FORMAT`, `POWER_CTL`, `BW_RATE` |

---

## 18. Function Design Specification

### 18.1 Design Principles

- **Single Responsibility**: Each function does only one thing (read register / calculate value / format output), making it easy to test and reuse
- **Small and Clear**: Break complex flows into multiple small functions (IO / parsing / validation separated), each function should ideally not exceed one screen (20 lines)
- **Side-Effect-Free First**: Separate code with side effects (I2C/SPI write/read/GPIO control) from pure computation
- **Testability**: Put pure algorithms into independent functions, easy to test on the host side with unittest/pytest (mock hardware layer)
- **Resource Awareness**: Avoid large allocations in hot loops or ISRs; reuse bytearray/buffers as much as possible

**Single Responsibility Example:**

```python
# Incorrect: Mixed responsibilities
def read_temperature():
    i2c.writeto(0x40, b'\xF3')
    time.sleep_ms(50)
    raw = i2c.readfrom(0x40, 2)
    return round((raw[0] << 8 | raw[1]) / 256, 1)

# Correct: Separation of responsibilities
def _trigger_measurement():
    i2c.writeto(0x40, b'\xF3')

def _read_raw_data():
