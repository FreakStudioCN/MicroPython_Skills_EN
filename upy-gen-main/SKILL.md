---
name: upy-gen-main
description: Use this skill when the user wants to generate a new main.py test file from scratch for a MicroPython driver. Invoke when user says things like "generate main.py", "生成测试文件", "从零生成main.py", "帮我写测试文件", or provides a driver .py file and asks to create a test program.
---

# MicroPython Test File Generation Skill

## Role Definition

You are the GraftSense MicroPython test file generation assistant. Given a driver `.py` file, analyze all its public APIs and generate a complete `main.py` test file from scratch that conforms to GraftSense specifications.

## Type Determination (Must be completed before executing any step)

After reading the driver file, immediately determine its type. Subsequent steps will follow the corresponding branch based on the type:

| Condition | Type |
|---|---|
| The driver is located in the `middleware/` subdirectory, or imports `network`/`urequests`/`AsyncWebsocketClient`/`asyncio` and has no I2C/SPI/UART hardware bus operations | **Middleware Library** |
| All other cases | **Hardware Driver** |

**Middleware libraries do not apply "boundary parameter scenarios" and "abnormal parameter scenarios"; replace them with "multi-parameter combination scenarios"**: Cover the various parameter combinations supported by the driver (e.g., different timbres, languages, speech rates, emotional styles). Also skip I2C scanning, replacing it with WiFi connection + credential configuration structure (see upy-norm-main #11a/#11b/#11c).

**Middleware library sensitive data replacement rules**: All credential fields in the generated main.py must use placeholders; real values must not be written:
| Data Type | Placeholder |
|---|---|
| WiFi SSID | `"your_wifi_ssid"` |
| WiFi Password | `"your_wifi_password"` |
| App ID / appid | `"your_app_id"` |
| Access Token / token | `"your_access_token"` |
| API Key | `"your_api_key"` |
| Other credential fields | `"your_<field_name>"` |

Add a comment above each placeholder constant: `# Please replace with your actual XXX`

## Execution Steps

1. Read the driver `.py` file specified by the user; **must re-read the complete content of the file, do not use session cache or skip the reading step**
2. Analyze the driver: extract all public methods, properties, constants, constructor parameters, and communication interface types
3. Classify APIs by chip function dimension (see dimension table below)
4. Generate test code based on the principle of full coverage
5. Output the complete `main.py` file

## Full API Coverage Principle

### Functional Dimension Classification

| Chip Type | Functional Dimensions to Cover |
|---|---|
| Sensor Class | Basic status query, core data acquisition, parameter configuration, mode switching, calibration/compensation |
| Motor Driver Class | Hardware initialization, motion control, status reading, reset/sleep |
| Communication Module Class | Network/protocol configuration, data transmission/reception, status query, power control |
| Memory Chip Class | Data read/write, address configuration, erase/reset |
| GPIO/Bus Expander Class | Pin configuration, level read/write, interrupt configuration |
| Middleware Library Class | Multi-parameter combination scenarios (e.g., timbre/language/speech rate/emotion), streaming vs. non-streaming comparison, credential configuration, WiFi connection, resource release |

### Three Scenarios Must Be Covered

| Scenario Type | Requirement |
|---|---|
| Normal Parameter Scenario | Basic calls with default/common parameters |
| Boundary Parameter Scenario | Hardware limit parameters (maximum, minimum) |
| Abnormal Parameter Scenario | Illegal parameters (out of range, wrong type), verify that exceptions are correctly raised |

> **Note: Middleware libraries do not apply "boundary parameter scenarios" and "abnormal parameter scenarios" (no hardware limits); replace them with "multi-parameter combination scenarios": Cover the various parameter combinations supported by the driver (e.g., different timbres, languages, speech rates, emotional styles).**

### API Feature Handling Methods

| API Feature | Code Handling Method |
|---|---|
| Low-frequency core API | Keep auto-execution, call periodically in main loop |
| High-frequency update API | Keep function definition, comment out auto-call, add comment indicating REPL manual call is possible |
| Mode switching API | Keep call code, comment out auto-execution, add comment indicating REPL manual trigger is possible |
| Batch operation API | Encapsulate as independent function for one-click REPL call |

## Generated Content Specifications

### Must Include (All)

| # | Content | Description |
|---|---|---|
| 1 | 7-line file header comment | Contains `@File : main.py`, `@Description : Test XXX driver class`; `@Author` read from the driver file's `__author__` field and reused; if absent, prompt the user to fill it in, do not use placeholders |
| 2 | 6 partition annotation comments | Correct order |
| 3 | `time.sleep(3)` | At the beginning of the initialization configuration section |
| 4 | `print("FreakStudio: ...")` | Indicates the driver module currently being tested |
| 5 | Hardware object instantiation | In the initialization configuration section, generated based on driver constructor parameters |
| 5a | I2C device scan + ID verification | If the driver uses I2C, the initialization configuration section must include complete scan logic: ① `i2c.scan()` scans the bus; if the list is empty, `raise RuntimeError("No I2C device found")`; ② Iterate through the device list; if the target address is found, record it; if not found, `raise RuntimeError("Device not found at expected address")`; ③ Read the chip ID register and compare with the expected value, print "Device found" or "Device not found"; device ID register address and expected value declared as global variable area constants (`UPPER_CASE`); additional `import` statements required for I2C scanning (e.g., `import micropython`) must be placed in the import section, not within the initialization configuration section |
| 6 | Call code for all public APIs | Low-frequency auto-execution, high-frequency/mode switching commented out |
| 7 | `try/except/finally` | Wraps the main program area, includes three catch types: KeyboardInterrupt/OSError/Exception |
| 8 | finally resource cleanup | `close()`/`deinit()`, `del`, exit prompt |
| 9 | raise/print all in English | All runtime strings in English |
| 10 | Inline comments in Chinese | All comments use Chinese; key operation steps inside functions (hardware initialization, data reading, conditional judgment, resource cleanup, etc.) must have Chinese comment explanations; **Comments must be written on the line above the corresponding code (independent comment line); comments at the end of code lines (trailing `#` comments) are prohibited** |

### Key Specification Summary

**File Header Format**
```python
# Python env   : MicroPython v1.23.0
# -*- coding: utf-8 -*-
# @Time    : YYYY/MM/DD
# @Author  : FreakStudio
# @File    : main.py
# @Description : Code to test the XXX driver class
# @License : MIT
```

**Global Variable Area** (only simple assignments allowed, instantiation prohibited)
```python
# ======================================== Global Variables ============================================
last_print_time = time.ticks_ms()
print_interval = 2000   # Print interval (ms)
```

**Function Area Example** (high-frequency/mode switching functions)
```python
# ======================================== Functions ============================================
def print_realtime_data():
    """Print real-time high-frequency data (high-frequency, default commented out, can be called manually via REPL)"""
    data = device.read_raw()
    print("Raw data: %s" % str(data))

def switch_to_sleep_mode():
    """Switch to sleep mode (mode switching, default commented out, can be triggered manually via REPL)"""
    device.sleep()
    print("Device entered sleep mode")
```

**Main Program Area Example**
```python
# ========================================  Main Program  ===========================================
try:
    while True:
        current_time = time.ticks_ms()
        if time.ticks_diff(current_time, last_print_time) >= print_interval:
            # Low-frequency query: keep auto-execution
            success, value = device.read_value()
            if success:
                print("Value: %s" % str(value))
            else:
                print("Read failed")
            last_print_time = current_time
        # print_realtime_data()    # High-frequency function, commented out by default, can be called manually via REPL
        # switch_to_sleep_mode()   # Mode switching, commented out by default, can be triggered manually via REPL
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

1. Output the complete `main.py` file content (code block preview).
2. Attach a brief explanation: list which APIs were covered, which were set to auto-execute, which were commented out for manual calls, and the reason.
3. Ask the user: "Confirm writing to `main.py` in the same directory?", and write the content to the file after user confirmation.

## Complete Specification Reference

This skill's rewriting rules are based on the GraftSense driver development specification document. For the complete specification (22 chapters, 2200+ lines), please refer to:

[Complete Specification Document](https://github.com/FreakStudioCN/MicroPython_Skills/blob/main/upy_driver_dev_spec_summary.md)

## Introspection and Evolution

After each execution, check if the following situations are encountered:
- Boundary cases not covered by the rules
- Output errors or rule defects pointed out by the user
- Newly discovered constraint requirements

If so, immediately execute:
1. Append the new rule to the corresponding section of this file
2. Synchronize the same modification to `G:/MicroPython_Skills/upy-gen-main/SKILL.md`
3. Execute in the `G:/MicroPython_Skills/` directory:
   `git add upy-gen-main/SKILL.md && git commit -m "skill(upy-gen-main): <rule description>"`
