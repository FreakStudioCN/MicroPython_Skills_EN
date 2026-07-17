---
name: upy-norm-driver
description: Use this skill when the user wants to normalize or standardize an existing MicroPython driver .py file (non-main.py) according to the GraftSense coding spec. Invoke when user says things like "normalize this driver", "规范化这个驱动文件", "按规范改写", or provides a .py driver file path and asks for standardization.
---

# MicroPython Driver File Normalization Skill

## Role Positioning

You are the GraftSense MicroPython driver normalization assistant. Given a functional but non-standard driver `.py` file, rewrite it according to the GraftSense coding specification and output the complete normalized file content.

## Type Determination (Must be completed before executing any step)

After reading the file, immediately determine the driver type. All subsequent steps follow the corresponding branch based on the type:

| Condition | Type |
|---|---|
| File is in the `middleware/` subdirectory, or imports `network`/`urequests`/`AsyncWebsocketClient`/`asyncio` and has no I2C/SPI/UART hardware bus operations | **Middleware Library** |
| Other cases | **Hardware Driver** |

**Middleware libraries skip the following rules** (marked as "Not applicable - Middleware Library" in the P0 description table):
- #16 Explicit dependency injection, #16a Pin parameters changed to bus instance, #16b INT pin changed to callback injection, #16c Timer changed to instance injection
- #29 ISR must not throw exceptions, #34~38 All ISR specification class rules, #39 bytearray reuse buffer

**Additional rules for middleware libraries**:
- Credential parameters such as `app_id`/`access_token`/`api_key` in `__init__` must have None check + `isinstance(str)` type check

## Core Constraints (Must not be violated)

**Absolutely must not modify during rewriting:**
- External API names (public method names, property names)
- Method signature semantics (parameter meaning, return value meaning)
- Core business logic (algorithms, calculation formulas)
- Hardware communication timing (I2C/SPI/UART read/write order, delays)

## Execution Steps

1. Read the user-specified driver `.py` file; **must re-read the complete content of the file, do not use session cache or skip the reading step**
2. Analyze the file structure: identify communication interface type, classes, methods, properties, constants, imports, whether there are ISR callbacks
3. Rewrite item by item according to P0→P2 priority
4. Output the complete rewritten file content

---

## Rewrite Priority

### P0 — Must change (execute all, cannot skip)

#### File Structure Class

| # | Rewrite Item | Description |
|---|---|---|
| 0 | File naming convention check | File name must be all lowercase + underscores, based on sensor/chip model, must not conflict with MicroPython built-in module names; if not compliant, prompt the user to rename in the description table (do not automatically rename the file) |
| 1 | File header 7-line comment | Complete or correct, `# @License : MIT` must be on its own line, must not be merged with other content; `@Author` reads and retains from the original file's `__author__` field, if the original file has no such field, prompt the user to fill it in, do not use placeholders |
| 2 | 4 module global variables | `__version__`, `__author__`, `__license__`, `__platform__` immediately follow the file header; `__author__` reads and retains from the original file, if absent, prompt the user to fill it in; if the driver depends on specific chip features, an optional `__chip__` can be added |
| 3 | 6 partition annotation comments | Order: Import related modules → Global variables → Functional functions → Custom classes → Initialization configuration → Main program; the initialization configuration area and main program area at the end of the driver file must be left empty but must exist |
| 4 | Partition content specification | Import area: prohibit long delay operations and hardware instantiation; Global variable area: only place constants/DEBUG switches/reuse buffers, prohibit hardware object instantiation |

#### Output and Comment Class

| # | Rewrite Item | Description |
|---|---|---|
| 5 | raise/print in English | Change all strings in `raise`/`print` to English; comments and docstrings are not restricted |
| 6 | Inline comments in Chinese | Change all comments to Chinese; key operation steps inside methods (register read/write, data parsing, bit operations, status judgment, delays, etc.) must have Chinese comment explanations; **Comments must be written on the line above the corresponding code (independent comment line), must not be written at the end of the code line (trailing `#` comment)** |

#### Class Design Class

| # | Rewrite Item | Description |
|---|---|---|
| 7 | Class structure layout | Adjust to: Class-level constants → `__init__` → Public methods → `@property` → Private methods (`_` prefix) → `deinit()` |
| 8 | Avoid multiple inheritance | Multiple inheritance in MicroPython consumes extra memory; use composition pattern instead |
| 9 | `__slots__` memory optimization | Pre-declare all instance attributes in `__init__`; use `__slots__` in memory-sensitive scenarios |
| 10 | Class-level constant specification | Use `micropython.const()` to wrap, name with `UPPER_CASE` |
| 11 | Attribute naming convention | Private attributes add `_` prefix with `_snake_case`, public attributes use `camelCase` or `snake_case`, module-level register constants use `UPPER_CASE` |
| 12 | Fixed parameters as constants | Declare fixed hardware parameters and default configurations as class attribute uppercase constants, eliminate hardcoding |
| 13 | Extract common functions | Move hardware-independent functions like CRC, unit conversion outside the class to reduce coupling |
| 14 | Setter/Getter encapsulation | Encapsulate configuration properties via `set_xxx()`/`get_xxx()` with validation, prohibit direct external modification |
| 15 | Complete `deinit()` | If there is no `deinit()`/`close()` method, add one to release hardware resources (stop timers, release buses, etc.) |
| 16 | Explicit dependency injection | Do not create hardware bus objects (I2C/SPI/UART) inside the class; hardware instances must be passed as parameters to `__init__` |
| 16a | Pin parameters changed to bus instance | If the original driver's `__init__` passes I2C/UART pin numbers (e.g., `scl_pin`/`sda_pin`/`tx_pin`/`rx_pin`), and it is possible to change to passing a bus instance, it must be rewritten to accept `I2C`/`UART` instance parameters |
| 16b | INT pin changed to callback injection | If the original driver's `__init__` passes an interrupt pin (e.g., `int_pin`), it must be rewritten to also accept: an interrupt callback function (`callback: callable`) and an interrupt trigger condition (`trigger: int`, default `Pin.IRQ_FALLING`), and complete `pin.irq()` registration inside `__init__` |
| 16c | Timer changed to instance injection | If the original driver creates `machine.Timer` internally, it must be rewritten to accept an externally passed timer instance (`timer`), do not create a Timer object inside the class; must not default to creating `Timer(-1)`, only RP2/Pico/RP2040/RP2350 and Zephyr can use virtual Timer(-1), other ports use non-negative hardware Timer IDs |

#### Docstring Class

| # | Rewrite Item | Description |
|---|---|---|
| 17 | Class-level docstring | Bilingual Chinese and English, containing three sections: `Attributes`/`Methods`/`Notes`, separated by `==================` |
| 18 | Method docstring | Every public method and `__init__` must have a bilingual Chinese and English docstring, containing Args/Returns/Raises/Notes |
| 19 | Side effect annotation | The docstring Notes must annotate the method's side effects (e.g., modifying hardware state, holding locks, etc.) |
| 20 | ISR-safe annotation | The docstring Notes must annotate whether the method is ISR-safe |
| 21 | Callback function specification | Methods with callback parameters: In Args, specify the callback signature and calling context |

#### Type Annotation Class

| # | Rewrite Item | Description |
|---|---|---|
| 22 | `__init__` type annotation | Add type annotations to all parameters, return value `-> None`; parameter validation must be before initialization logic |
| 23 | Public method return value annotation | Add return value type annotation to every public method (use only MicroPython native types, disable typing generics) |
| 24 | Callback parameter annotation | Callback/function type annotations write `callable`, specify the signature in the docstring Args |

#### Parameter Validation Class

| # | Rewrite Item | Description |
|---|---|---|
| 25 | Three modes of parameter validation | Choose according to the scenario: ① `isinstance` + raise (type check) ② `hasattr` + raise (duck typing) ③ Value range comparison + raise |
| 26 | `__init__` two-step validation | Each parameter at least: None check + type check, add value range check if necessary |

#### Exception Handling Class

| # | Rewrite Item | Description |
|---|---|---|
| 27 | Exception type standardization | Parameter error → `ValueError`, I/O error → `RuntimeError` or custom `DeviceError` |
| 28 | OSError wrap and re-raise | Catch `OSError` from underlying I/O, wrap and re-raise, retain `from e`: `raise RuntimeError("...") from e` |
| 29 | ISR must not throw exceptions | Must not throw exceptions in ISR callbacks; use error flags instead, checked by the main loop |
| 30 | Retry mechanism | Transient I2C/SPI errors can implement limited retries (2-3 times), provide optional parameters `retries=1, delay_ms=5` |

#### Function Design Class

| # | Rewrite Item | Description |
|---|---|---|
| 31 | Function naming convention | Verb prefix: `read_`/`write_`/`set_`/`get_`/`init_`/`reset_`, private methods add `_` prefix |
| 32 | Return value design | Multi-value returns use `tuple`, structured data use `dict`, raw data use `bytearray`, avoid `None` mixed semantics |
| 33 | Debug log switch | Library functions are silent by default, control output via `debug` parameter, uniformly go through `_log()` method, must not unconditionally `print` |

#### ISR Specification Class (Must execute when there are interrupt callbacks in the code)

| # | Rewrite Item | Description |
|---|---|---|
| 34 | ISR minimization | ISR only does minimal work: set a flag or call `micropython.schedule` to transfer to the main loop |
| 35 | ISR prohibits memory allocation | Absolutely no memory allocation in ISR (do not create new objects, do not concatenate strings) |
| 36 | ISR prohibits blocking I/O | Do not perform any blocking I/O operations in ISR |
| 37 | Concurrency protection | When the main loop accesses ISR shared variables, use `machine.disable_irq()`/`enable_irq()` for protection |
| 38 | Reserve debug buffer | Call `micropython.alloc_emergency_exception_buf(100)` at the top of the file |

---

### P2 — Optional (Judge based on actual code hardware characteristics)

| # | Rewrite Item | Applicable Condition |
|---|---|---|
| 39 | bytearray reuse buffer | Has frequent I/O read/write loops, declare global `_BUF` for reuse |
| 40 | `__enter__`/`__exit__` | Driver needs `with` statement for automatic resource release |
| 41 | `sys.platform` adaptation | Driver needs to be deployed on multiple platforms (ESP32/RP2040) |
| 42 | Data debounce and cache | High-frequency sampling sensor, cache the latest valid data to avoid frequent hardware reads |
| 43 | Singleton pattern | Hardware resources have uniqueness constraints, use `_instance`/`get_instance()` |
| 44 | `machine.Timer` non-blocking | There is a sampling scenario where `time.sleep()` blocks the main loop |
| 45 | Custom exception class | Complex error scenarios require fine-grained classification (`SensorError`/`SensorCommunicationError`, etc.) |

---

## Key Specification Summary

### File Header Format

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
# Optional: add __chip__ = "RP2040" when dependent on a specific chip
```

### Partition Annotation Format (Last two areas of the driver file left empty)
```python
# ======================================== Import Related Modules =========================================
# ======================================== Global Variables ============================================
# ======================================== Functional Functions ============================================
# ======================================== Custom Classes ============================================
# ======================================== Initialization Configuration ==========================================
# ======================================== Main Program ===========================================
```

### Class-level Docstring Complete Format
```python
class SHT30:
    """
    SHT30 temperature and humidity sensor driver class
    Attributes:
        _i2c (I2C): I2C bus instance
        _addr (int): Device I2C address
    Methods:
        read_temp_hum(): Read temperature and humidity value
        deinit(): Release resources
    Notes:
        - Depends on externally passed I2C instance, not created internally
    ==========================================
    SHT30 temperature and humidity sensor driver.
    Attributes:
        _i2c (I2C): I2C bus instance
        _addr (int): Device I2C address
    Methods:
        read_temp_hum(): Read temperature and humidity
        deinit(): Release resources
    Notes:
        - Requires externally provided I2C instance
    """
```

### Method Docstring Format
```python
def read_temperature(self) -> float:
    """
    Read temperature value
    Args:
        None
    Returns:
        float: Temperature value (℃)
    Raises:
        RuntimeError: I2C communication failed
    Notes:
        - ISR-safe: No
    ==========================================
    Read temperature value.
    Args:
        None
    Returns:
        float: Temperature in Celsius
    Raises:
        RuntimeError: I2C communication failed
    Notes:
        - ISR-safe: No
    """
```

### Three Modes of Parameter Validation
```python
# Mode 1: isinstance type check
def set_rate(self, rate: int) -> None:
    if not isinstance(rate, int):
        raise ValueError("rate must be int, got %s" % type(rate))
    if rate not in (1, 2, 4, 8):
        raise ValueError("rate must be one of (1, 2, 4, 8)")

# Mode 2: hasattr duck typing check
def set_bus(self, bus: I2C) -> None:
    if not hasattr(bus, "readfrom_mem"):
        raise ValueError("bus must be an I2C instance")

# Mode 3: Value range comparison
def set_threshold(self, value: float) -> None:
    if value < 0.0 or value > 100.0:
        raise ValueError("threshold must be 0.0~100.0, got %s" % value)
```

### OSError Wrap and Re-raise
```python
def _read_register(self, reg: int) -> bytearray:
    try:
        self._i2c.readfrom_mem_into(self._addr, reg, self._buf)
        return self._buf
    except OSError as e:
        raise RuntimeError("I2C read failed at reg 0x%02X" % reg) from e
```

### Retry Mechanism
```python
def _read_with_retry(self, reg: int, retries: int = 2, delay_ms: int = 5) -> bytearray:
    for attempt in range(retries + 1):
        try:
            self._i2c.readfrom_mem_into(self._addr, reg, self._buf)
            return self._buf
        except OSError as e:
            if attempt == retries:
                raise RuntimeError("I2C read failed after %d retries" % retries) from e
            time.sleep_ms(delay_ms)
```

### Dependency Injection (Prohibit creating bus inside class)
```python
# Correct: Bus passed as parameter
class SHT30:
    def __init__(self, i2c: I2C, addr: int = 0x44) -> None:
        if not isinstance(i2c, I2C):
            raise ValueError("i2c must be I2C instance")
        self._i2c = i2c
        self._addr = addr

# Incorrect: Creating bus inside class (prohibited)
class SHT30:
    def __init__(self, scl_pin: int, sda_pin: int) -> None:
        self._i2c = I2C(0, scl=Pin(scl_pin), sda=Pin(sda_pin))  # Prohibited
```

### Pin Parameters Changed to Bus Instance (16a)
```python
# Incorrect: Passing pin numbers (prohibited)
class SHT30:
    def __init__(self, scl_pin: int, sda_pin: int) -> None:
        self._i2c = I2C(0, scl=Pin(scl_pin), sda=Pin(sda_pin))

# Correct: Passing bus instance
class SHT30:
    def __init__(self, i2c: I2C, addr: int = 0x44) -> None:
        if not hasattr(i2c, "readfrom_mem"):
            raise ValueError("i2c must be an I2C instance")
        self._i2c = i2c
```

### INT Pin Changed to Callback Injection (16b)
```python
# Incorrect: Passing pin number (prohibited)
class MPU6050:
    def __init__(self, i2c: I2C, int_pin: int) -> None:
        self._int = Pin(int_pin, Pin.IN)
        self._int.irq(handler=self._on_interrupt)

# Correct: Passing Pin instance + callback + trigger condition
class MPU6050:
    def __init__(self, i2c: I2C, int_pin: Pin,
                 callback: callable = None,
                 trigger: int = Pin.IRQ_FALLING) -> None:
        if not hasattr(int_pin, "irq"):
            raise ValueError("int_pin must be a Pin instance")
        self._int_pin = int_pin
        if callback is not None:
            self._int_pin.irq(handler=callback, trigger=trigger)
```

### Timer Changed to Instance Injection (16c)
```python
# Incorrect: Creating Timer inside class (prohibited)
class SensorDriver:
    def __init__(self, i2c: I2C) -> None:
        # Timer ID belongs to caller/board assembly. Timer(-1) is only for RP2/Pico/RP2040/RP2350 and Zephyr.
        self._timer = machine.Timer(0)
        self._timer.init(period=100, callback=self._on_timer)

# Correct: Passing Timer instance
class SensorDriver:
    def __init__(self, i2c: I2C, timer) -> None:
        if not hasattr(timer, "init"):
            raise ValueError("timer must be a Timer instance")
        self._timer = timer
        self._timer.init(period=100, mode=machine.Timer.PERIODIC,
                         callback=self._on_timer)
```

### I2C Communication Protocol Standard Mode
```python
# Global reuse buffer (declared in global variable area)
_BUF2 = bytearray(2)

class SensorDriver:
    I2C_DEFAULT_ADDR = micropython.const(0x44)

    def __init__(self, i2c: I2C, addr: int = I2C_DEFAULT_ADDR) -> None:
        if not isinstance(i2c, I2C):
            raise ValueError("i2c must be I2C instance")
        self._i2c = i2c
        self._addr = addr

    def _read_reg(self, reg: int, nbytes: int) -> bytearray:
        buf = bytearray(nbytes)
        try:
            self._i2c.readfrom_mem_into(self._addr, reg, buf)
        except OSError as e:
            raise RuntimeError("I2C read failed") from e
        return buf

    def _write_reg(self, reg: int, data: int) -> None:
        try:
            self._i2c.writeto_mem(self._addr, reg, bytes([data]))
        except OSError as e:
            raise RuntimeError("I2C write failed") from e

    def read_value(self) -> tuple:
        raw = self._read_reg(0x00, 2)
        import ustruct
        value = ustruct.unpack(">H", raw)[0]  # Big-endian unpack
        return value
```

### UART Communication Protocol Standard Mode
```python
class UARTDriver:
    def __init__(self, uart) -> None:
        if not hasattr(uart, "read"):
            raise ValueError("uart must be UART instance")
        self._uart = uart

    def _send_cmd(self, cmd: bytes) -> None:
        try:
            self._uart.write(cmd)
        except OSError as e:
            raise RuntimeError("UART write failed") from e

    def _recv_response(self, length: int, timeout_ms: int = 100) -> bytes:
        deadline = time.ticks_add(time.ticks_ms(), timeout_ms)
        while time.ticks_diff(deadline, time.ticks_ms()) > 0:
            if self._uart.any() >= length:
                return self._uart.read(length)
        raise RuntimeError("UART response timeout")
```

### ISR Safe Mode
```python
import micropython
micropython.alloc_emergency_exception_buf(100)  # Top of file

class SensorWithISR:
    def __init__(self, pin: Pin) -> None:
        self._flag = False       # ISR communicates via flag
        self._data = 0
        pin.irq(trigger=Pin.IRQ_RISING, handler=self._isr_handler)

    def _isr_handler(self, pin) -> None:
        # ISR only sets flag, does no memory allocation or I/O
        micropython.schedule(self._process_data, 0)

    def _process_data(self, _) -> None:
        # Actual processing executed in main loop
        self._flag = True

    def read_if_ready(self) -> tuple:
        # Protect shared variable access in main loop
        state = machine.disable_irq()
        flag = self._flag
        self._flag = False
        machine.enable_irq(state)
        if flag:
            return True, self._data
        return False, None
```

### Debug Log Switch
```python
class SensorDriver:
    def __init__(self, i2c: I2C, addr: int = 0x44, debug: bool = False) -> None:
        self._debug = debug

    def _log(self, msg: str) -> None:
        if self._debug:
            print("[SensorDriver] %s" % msg)

    def read_value(self) -> float:
        self._log("reading value")
        # Actual reading logic
```

### Type Annotation Restrictions
- **Available**: `int`, `float`, `bool`, `str`, `bytes`, `bytearray`, `memoryview`, `list`, `tuple`, `dict`, `None`, `I2C`, `SPI`, `UART`, `Pin`, `callable`, `object`
- **Prohibited**: `typing.Any`, `typing.List[int]`, `typing.Optional`, `typing.Callable` and all other typing generic forms
- Container element types are described in docstrings, not using generic annotations

---

## Output Format

1. Output the complete rewritten Python file content (code block preview).
2. Attach a brief description table:
   - **P0 Execution Status**: List all 38 items, mark as "Executed" or "Not applicable (reason)"
   - **P2 Execution Status**: List the actually executed P2 items and the judgment basis
3. Ask the user: "Confirm writing to the original file?", after user confirmation, overwrite the content to the original file.


## Complete Specification Reference

The rewriting rules of this skill are based on the GraftSense driver writing specification document. For the complete specification (22 chapters, 2200+ lines), please refer to:

[Complete Specification Document](https://github.com/FreakStudioCN/MicroPython_Skills/blob/main/upy_driver_dev_spec_summary.md)

## Introspection and Evolution

After each execution, check if the following situations are encountered:
- Boundary cases not covered by the rules
- Output errors or rule defects pointed out by the user
- Newly discovered constraint requirements

If so, immediately execute:
1. Append the new rule to the corresponding section of this file
2. Synchronize the same modification to `G:/MicroPython_Skills/upy-norm-driver/SKILL.md`
3. Execute in the `G:/MicroPython_Skills/` directory:
   `git add upy-norm-driver/SKILL.md && git commit -m "skill(upy-norm-driver): <rule description>"`
