---
name: upy-norm-main
description: Use this skill when the user wants to normalize or standardize an existing MicroPython main.py test file according to the GraftSense coding spec. Invoke when user says things like "normalize this main.py", "规范化测试文件", "按规范改写main.py", or provides an existing main.py path and asks for standardization.
---

# MicroPython Test File Normalization Skill

## Role Positioning

You are the GraftSense MicroPython test file normalization assistant. Given a functional but non-standard `main.py`, rewrite it according to the GraftSense coding specification and output the complete normalized file content.

## Type Determination (Must be completed before executing any step)

After reading the corresponding driver file, immediately determine the type and follow the corresponding branch for subsequent steps:

| Condition | Type |
|---|---|
| Driver located in `middleware/` subdirectory, or imports `network`/`urequests`/`AsyncWebsocketClient`/`asyncio` without I2C/SPI/UART hardware bus operations | **Middleware Library** |
| Other cases | **Hardware Driver** |

**Middleware libraries skip #11 I2C scan + ID verification, replace with the following three rules:**
- **#11a** Define `connect_wifi()` function in the function area, call it in the initialization configuration area and print the IP address
- **#11b** Declare credential constants such as `APP_ID`/`ACCESS_TOKEN` (`UPPER_CASE`) in the global variable area, do not hardcode them in instantiation statements
- **#11c** Use `tests = [...]` list-driven multi-scenario testing in the main program area, replacing the `while True` polling structure; use `asyncio.run()` as the entry point

**Middleware library sensitive data replacement rules (#11d):**
Scan all real credential values in the file and uniformly replace them with placeholders, including:
| Data Type | Replace With |
|---|---|
| WiFi SSID | `"your_wifi_ssid"` |
| WiFi Password | `"your_wifi_password"` |
| App ID / appid | `"your_app_id"` |
| Access Token / token | `"your_access_token"` |
| API Key | `"your_api_key"` |
| Other credential fields | `"your_<field_name>"` |

After replacement, add a comment above the corresponding constant in the global variable area: `# Replace with your actual XXX`

## Core Constraints (Cannot be violated)

- Do not modify the test's business logic and API call order
- Do not delete any existing functional functions or test cases
- Do not modify hardware pin configurations (unless obviously incorrect)

## Execution Steps

1. Read the user-specified `main.py` file; **must re-read the complete file content, do not use session cache or skip the reading step**
2. Analyze the existing structure: identify imports, global variables, functions, initialization, main loop
3. Rewrite step by step according to P0→P1→P2 priority
4. Output the complete rewritten file content

## Rewrite Priority

### P0 — Mandatory (All must be executed, cannot be skipped)

| # | Rewrite Item | Description |
|---|---|---|
| 1 | 7-line file header comment | Complete or correct (no need for `__version__` and other global variables); `@Author` reads from the original file and retains it; if absent, prompt the user to fill it in, do not use placeholders |
| 2 | 6 partition annotation comments | Order: Import Related Modules → Global Variables → Functional Functions → Custom Classes → Initialization Configuration → Main Program |
| 3 | `time.sleep(3)` | Must be at the beginning of the initialization configuration area, cannot be deleted |
| 4 | FreakStudio print | Must have a `print("FreakStudio: ...")` format print in the initialization configuration area |
| 5 | Instantiation location | Instantiation (`I2C()`, `Pin()`, etc.) is prohibited in the global variable area; move to the initialization configuration area |
| 6 | While loop location | `while` loops are only allowed in the main program area; no other area may have them |
| 7 | raise/print in English | Change all strings in `raise`/`print` to English |
| 8 | try/except/finally | Wrap the while loop in the main program area with `try/except KeyboardInterrupt/OSError/Exception/finally` |
| 9 | finally resource cleanup | In `finally`, call `device.close()`/`deinit()`, `del` hardware objects, and print exit prompt |
| 10 | Inline comments in Chinese | Change all inline comments to Chinese; **Comments must be written on a separate line above the corresponding code line (independent comment line), and are prohibited from being written at the end of the code line (trailing `#` comment)** |
| 11 | I2C device scan + ID verification | If the driver uses I2C, the initialization configuration area must include complete scan logic: ① `i2c.scan()` scans the bus; if the list is empty, `raise RuntimeError("No I2C device found")`; ② Traverse the device list; if the target address is found, record it; if not found, `raise RuntimeError("Device not found at expected address")`; ③ Read the chip ID register and compare with the expected value, print "Device found" or "Device not found"; Device ID register address and expected value are declared as constants in the global variable area (`UPPER_CASE`); Additional `import` required for I2C scan must be placed in the import area, do not `import` inside the initialization configuration area |
| 11a | Middleware library: WiFi connection function | If the driver is a middleware library (see upy-norm-driver type determination rules), skip #11 I2C scan and replace with: define `connect_wifi()` function in the function area, call it in the initialization configuration area and print the IP address |
| 11b | Middleware library: Credential configuration area | Declare credential constants such as `APP_ID`/`ACCESS_TOKEN` (`UPPER_CASE`) in the global variable area, do not hardcode them in instantiation statements |
| 11c | Middleware library: Scenario list structure | Use `tests = [...]` list-driven multi-scenario testing in the main program area, replacing the `while True` polling structure; use `asyncio.run()` as the entry point |

### P1 — Try to apply

| # | Rewrite Item | Description |
|---|---|---|
| 11 | High-frequency function handling | Keep the definition of high-frequency update/mode switching functions, comment out the automatic call in the main program, add a comment explaining that it can be called manually via REPL |
| 12 | Three types of test scenario coverage check | Check whether the existing test code covers normal parameter scenarios, boundary parameter scenarios (hardware limit values), and abnormal parameter scenarios (invalid values to verify if exceptions are thrown); add call code for missing scenarios |
| 13 | Functional function docstring | Add a brief Chinese docstring for each functional function |
| 14 | Global variable naming | Change to `snake_case`, e.g., `print_interval`, `last_print_time` |

### P2 — Optional

| # | Rewrite Item | Applicable Condition |
|---|---|---|
| 14 | Batch operation encapsulation | When there are multiple similar API calls, encapsulate them into a batch test function for one-click REPL invocation |

## Key Specification Summary

### File Header Format (main.py version)
```python
# Python env   : MicroPython v1.23.0
# -*- coding: utf-8 -*-
# @Time    : YYYY/MM/DD HH:MM
# @Author  : Author Name
# @File    : main.py
# @Description : Code for testing XXX driver class
# @License : MIT
```

### Standard Structure of Initialization Configuration Area
```python
# ======================================== Initialization Configuration ==========================================
time.sleep(3)
print("FreakStudio: Using XXX ...")
# Hardware object instantiation
uart = UART(0, baudrate=115200, tx=Pin(16), rx=Pin(17), timeout=0)
device = DriverClass(uart)
```

### Standard Structure of Main Program Area
```python
# ========================================  Main Program  ===========================================
try:
    while True:
        current_time = time.ticks_ms()
        if time.ticks_diff(current_time, last_print_time) >= print_interval:
            # Low-frequency query retains automatic execution
            ...
            last_print_time = current_time
        # print_high_freq_data()  # High-frequency function, commented out by default, can be called manually via REPL
        time.sleep_ms(10)

except KeyboardInterrupt:
    print("Program interrupted by user")
except OSError as e:
    print("Hardware communication error: %s" % str(e))
except Exception as e:
    print("Unknown error: %s" % str(e))
finally:
    print("Cleaning up resources...")
    device.close()
    del device
    print("Program exited")
```

## Output Format

1. Output the complete rewritten Python file content (code block preview).
2. Attach a brief description listing which rewrite items were actually executed.
3. Ask the user: "Confirm writing to the original file?", and overwrite the original file after user confirmation.

## Complete Specification Reference

The rewrite rules of this Skill are based on the GraftSense driver writing specification document. For the complete specification (22 chapters, 2200+ lines), please refer to:

[Complete Specification Document](https://github.com/FreakStudioCN/MicroPython_Skills/blob/main/upy_driver_dev_spec_summary.md)

## Introspection and Evolution

After each execution, check whether the following situations are encountered:
- Boundary cases not covered by the rules
- Output errors or rule defects pointed out by the user
- Newly discovered constraint requirements

If so, immediately execute:
1. Append the new rule to the corresponding section of this file
2. Synchronize the same modification to `G:/MicroPython_Skills/upy-norm-main/SKILL.md`
3. Execute in the `G:/MicroPython_Skills/` directory:
   `git add upy-norm-main/SKILL.md && git commit -m "skill(upy-norm-main): <rule description>"`
