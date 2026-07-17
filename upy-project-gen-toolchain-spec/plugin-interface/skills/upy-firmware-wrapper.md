# Firmware API Wrapper Package Specification

> Status: ⚠ To be filled
>
> This document is not a skill interface document (no Phase execution flow), but a **specification for writing firmware API Wrapper packages**.
> For embedded engineers: When you find that the API of a MicroPython hardware device is already implemented in C in the firmware and exposed as Python classes/methods/functions,
> write a thin wrapper .py package according to this specification and upload it to upypi, so that the entire pipeline can use it normally without any special handling.

---

## 1. Concept Explanation

### Problem

Many MicroPython hardware products come with custom firmware, where C-language driver implementations are compiled into the firmware and directly exposed as Python interfaces, for example:

```python
# Firmware built-in C API — can be called without importing any .py file
sht30 = SHT30(i2c)
temp, hum = sht30.measure()
```

The current pipeline's driver search (upy-analyze → upy-pkg-guide) only looks for `.py` driver packages on upypi / GitHub.
Firmware built-in APIs are not on upypi → marked as "no driver" → enters the cold hardware path (gen-driver),
the system will attempt to generate a new driver from scratch — which is obviously redundant and incorrect.

### Solution

Write a **Wrapper package** for each firmware built-in API and upload it to upypi:

- **On real devices**: `import` the firmware built-in class, pass through transparently
- **During PC simulation**: expose a stub class (with only type hints + docstring), for LLM to understand the API, generate code, and perform static checks

From now on, the pipeline does not need to know whether the "API is in firmware or in a .py file" — it always searches upypi, and uses it if found.

---

## 2. Wrapper .py File Specification

### 2.1 Core Pattern: try/except Dual State

```python
"""
SHT30 Temperature and Humidity Sensor Driver (Firmware Wrapper).

Firmware built-in: Yes (vendor custom firmware v2.1+)
I2C address: 0x44
"""

# ---------- Real device: pass through firmware built-in class ----------
try:
    from sht30 import SHT30 as _FirmwareSHT30

    class SHT30(_FirmwareSHT30):
        """
        SHT30 Temperature and Humidity Sensor.

        Hardware: I2C interface, default address 0x44
        Accuracy: Temperature ±0.3°C, Humidity ±2%RH
        Power supply: 2.4V – 5.5V
        """
        def __init__(self, i2c, addr: int = 0x44):
            super().__init__(i2c, addr)

        def measure(self) -> tuple[float, float]:
            """
            Perform one temperature and humidity measurement.

            Returns:
                (temperature_celsius, relative_humidity_pct)

            Example:
                >>> sht30 = SHT30(i2c)
                >>> temp, hum = sht30.measure()
                >>> print(f"{temp:.1f}°C, {hum:.1f}%")
            """
            return super().measure()

        def read_temp(self) -> float:
            """Read temperature only (°C)"""
            return self.measure()[0]

        def read_humidity(self) -> float:
            """Read humidity only (%RH)"""
            return self.measure()[1]

# ---------- PC side: pure stub (for LLM autocomplete + static checks) ----------
except ImportError:

    class SHT30:
        """
        [STUB] SHT30 Temperature and Humidity Sensor — PC-side placeholder for firmware built-in API.

        Only effective during PC simulation / IDE autocomplete.
        On real devices, this class is replaced by the C implementation in firmware.
        """

        def __init__(self, i2c, addr: int = 0x44) -> None: ...
        def measure(self) -> tuple[float, float]: ...
        def read_temp(self) -> float: ...
        def read_humidity(self) -> float: ...
```

### 2.2 Specification Points

| # | Specification | Description |
|---|---------------|-------------|
| 1 | **Class name unchanged** | Wrapper class name must match the firmware built-in class name (here `SHT30`) |
| 2 | **Inheritance pass-through** | The real device branch inherits the firmware built-in class, each method uses `super().xxx()` for pass-through |
| 3 | **Stub uses `...`** | PC stub method bodies use `...` (Ellipsis), not `pass`. IDE recognizes it as an abstract method |
| 4 | **Complete type hints** | All parameters and return values must have type annotations for LLM code generation reference |
| 5 | **Complete docstrings** | Write docstrings for each class and method, including hardware constraints (voltage/address/timing) and usage examples |
| 6 | **Stub classes also have docstrings** | `__init__` docstrings should describe interface requirements (I2C/SPI/UART, etc.) |
| 7 | **Use `[STUB]` marker** | PC stub class-level docstrings start with `[STUB]` for easy search differentiation |
| 8 | **Single file** | A wrapper package usually has only one .py file. Complex devices can be split into packages |

### 2.3 What Not to Implement

The Wrapper package does **not** perform any business logic:
- Does not implement I2C communication protocol
- Does not implement register read/write
- Does not implement data parsing/calibration algorithms

These are already done in the firmware C code. The Wrapper is only responsible for "declaring the interface + pass-through calls".

---

## 3. Package Directory Structure

```
sht30_firmware_wrapper/           # Package root directory
├── package.json                  # Metadata (required)
├── sht30.py                      # Wrapper module (required, must match device name)
├── README.md                     # Usage instructions (required)
└── example.py                    # Usage example (recommended)
```

---

## 4. package.json Metadata

### 4.1 Required Fields

```json
{
  "name": "sht30_firmware_wrapper",
  "version": "1.0.0",
  "type": "driver",
  "driver_type": "wrapper",
  "wrapper_of": "firmware_builtin",
  "chip_model": "SHT30",
  "description": "SHT30 temperature and humidity sensor firmware API Wrapper. Passes through firmware built-in C implementation on real devices, provides stub for code completion on PC.",
  "author": "",
  "license": "MIT",
  "keywords": ["sht30", "temperature", "humidity", "i2c"],
  "upy": {
    "bus": ["i2c"],
    "i2c_addr": ["0x44"],
    "firmware": {
      "required_modules": ["sht30"],
      "min_version": "2.1.0",
      "vendor_firmware": true
    }
  }
}
```

### 4.2 Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Package name, recommended `{chip_lower}_firmware_wrapper` |
| `version` | string | Yes | Semantic version. Do not bump if firmware API is unchanged |
| `type` | string | Yes | Fixed to `"driver"` |
| `driver_type` | string | Yes | Fixed to `"wrapper"`. Differentiates from normal driver `"native"` |
| `wrapper_of` | string | Yes | Fixed to `"firmware_builtin"` |
| `chip_model` | string | Yes | Chip model, must match boards.json and manifest |
| `description` | string | Yes | One-line description |
| `author` | string | No | Author |
| `license` | string | Yes | MIT recommended |
| `keywords` | string[] | Yes | For upypi search |
| `upy.bus` | string[] | Yes | Required bus enumeration: `i2c` / `spi` / `uart` / `gpio` / `analog` / `pwm` |
| `upy.i2c_addr` | string[] | No | Required for I2C devices, list possible addresses |
| `upy.firmware.required_modules` | string[] | Yes | List of module names that must be built into the firmware |
| `upy.firmware.min_version` | string | No | Minimum firmware version requirement |
| `upy.firmware.vendor_firmware` | boolean | Yes | `true` = vendor custom firmware, `false` = standard MPY firmware module (e.g., `machine.I2C`) |

### 4.3 Differentiation from Normal Driver Packages

```
Normal driver package (driver_type: "native" or missing):
  sht30.py → Full Python implementation, includes I2C register read/write
  package.json.upy.firmware → does not exist or is empty

Wrapper package (driver_type: "wrapper"):
  sht30.py → Inherits firmware built-in class + PC stub
  package.json.upy.firmware.required_modules → required
```

---

## 5. Protocol Interaction with the Pipeline

### 5.1 Message Perspective

The Wrapper package **does not generate any special protocol messages**. After being found by upy-analyze → upy-pkg-guide,
it behaves exactly like a normal upypi driver package:

```
analyze:   Found sht30_firmware_wrapper → devices[].driver.source = "upypi"
generate:  Download sht30.py → LLM reads API → generates factory + Mock → generates task
deploy:    Upload sht30.py to board → on import, automatically uses firmware built-in C implementation
```

The pipeline does not need to know this is a wrapper — to the pipeline, it's just a normal upypi driver.

### 5.2 Representation in the Manifest

In `project-manifest.json`, the driver record corresponding to the wrapper package should add:

```json
{
  "devices": [
    {
      "name": "SHT30",
      "driver": {
        "source": "upypi",
        "package_name": "sht30_firmware_wrapper",
        "version": "1.0.0",
        "driver_type": "wrapper",
        "wrapper_of": "firmware_builtin",
        "required_firmware_modules": ["sht30"],
        "vendor_firmware": true
      }
    }
  ]
}
```

| New Field | Purpose |
|-----------|---------|
| `driver_type` | Tells downstream "this is not a self-implemented driver, it's a firmware pass-through" |
| `wrapper_of` | `"firmware_builtin"` differentiates from possible future wrapper types |
| `required_firmware_modules` | Used during deploy stage pre-flight checks |
| `vendor_firmware` | When `true`, the select-hw stage additionally prompts "requires vendor custom firmware" |

### 5.3 upy-deploy Pre-flight Check

This is the **only** place in the pipeline that needs to be aware of the wrapper — before flashing, it must confirm the firmware contains the required modules:

```
Step 0 pre-flight:
  → Read manifest, find all devices with driver_type = "wrapper"
  → device_command: mpremote exec "help('modules')"
  → Cross-reference required_firmware_modules
  → If missing, approval_request warns the user:
     "SHT30 requires vendor custom firmware (with built-in sht30 module), current firmware does not detect this module"
```

---

## 6. When to Use Wrapper vs Cold Hardware Path vs Normal Driver

| Situation | Which Path | Output |
|-----------|------------|--------|
| Firmware built-in C API, wrapper already on upypi | Normal path (upy-analyze → download and use) | None |
| Firmware built-in C API, no wrapper on upypi | **Embedded engineer writes wrapper per this spec → upy-publish uploads to upypi** | Wrapper package |
| External .py driver package exists (upypi / GitHub) | Normal path (upy-analyze → download and use) | None |
| No firmware API, no external driver | Cold hardware path (gen-driver) | Generate driver from scratch |
| Standard MPY modules (machine.Pin / machine.I2C / network.WLAN, etc.) | No package needed, pipeline uses directly via hardcoded references | None |

---

## 7. Complete Examples

### Example 1: I2C Sensor (with vendor custom firmware)

`bme280_firmware_wrapper/bme280.py`:

```python
"""
BME280 Environmental Sensor Driver (Firmware Wrapper).

Firmware built-in: Yes (Bosch BME280 custom firmware v1.3+)
I2C address: 0x76 / 0x77
"""

try:
    from bme280 import BME280 as _FirmwareBME280

    class BME280(_FirmwareBME280):
        """BME280 temperature/humidity/pressure three-in-one sensor."""
        def __init__(self, i2c, addr: int = 0x76):
            super().__init__(i2c, addr)

        def read_all(self) -> tuple[float, float, float]:
            """Read all data at once. Returns: (temp_c, humidity_pct, pressure_hpa)"""
            return super().read_all()

        def temperature(self) -> float:
            """Temperature (°C)"""
            return self.read_all()[0]

        def humidity(self) -> float:
            """Humidity (%RH)"""
            return self.read_all()[1]

        def pressure(self) -> float:
            """Pressure (hPa)"""
            return self.read_all()[2]

except ImportError:

    class BME280:
        """[STUB] BME280 — PC-side placeholder for firmware built-in API."""
        def __init__(self, i2c, addr: int = 0x76) -> None: ...
        def read_all(self) -> tuple[float, float, float]: ...
        def temperature(self) -> float: ...
        def humidity(self) -> float: ...
        def pressure(self) -> float: ...
```

### Example 2: Standard Firmware Module (non-vendor, using machine.PWM as an example)

Standard MPY firmware modules (`machine`, `network`, `bluetooth`, etc.) **do not** need a wrapper.
They are assumed to exist in all standard firmware, and the pipeline uses them directly via hardcoded references.

A wrapper is only needed when the device's API is a **private module provided by vendor custom firmware**.

---

## 8. Relationship with upy-publish

After the embedded engineer finishes writing the wrapper package, use the `upy-publish` skill to:

1. Generate README.md — auto-populated from package.json
2. Generate LICENSE
3. Package into standard upypi directory structure
4. Upload to upypi (after user confirmation)

When publishing a package with `driver_type: "wrapper"`, the README template will additionally include:

```markdown
## Firmware Requirements

This driver is a Wrapper for the firmware built-in API and **does not implement any hardware communication**.

- Required firmware built-in modules: `sht30`
- Minimum firmware version: v2.1.0
- Firmware type: Vendor custom firmware

Please ensure your MicroPython device is flashed with firmware containing the above modules.
```

---

## 9. Acceptance Checklist

Check before the embedded engineer submits the wrapper package:

```
[ ] The try branch correctly passes through the firmware built-in class
[ ] The except ImportError branch has a complete stub class
[ ] All methods have type hints
[ ] All classes and methods have docstrings
[ ] package.json is fully filled in (especially the upy.firmware fields)
[ ] driver_type = "wrapper"
[ ] required_firmware_modules is correctly listed
[ ] example.py can run on a real device
[ ] The package has been uploaded to upypi via upy-publish
