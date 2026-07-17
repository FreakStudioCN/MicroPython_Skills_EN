---
name: upy-project
description: Use this skill when the user describes a MicroPython project — what it should do, what microcontroller and sensors to use. Invoke when user says things like "I want to create a MicroPython project", "Help me write an ESP32 program", "Build a weather station with BMP280 and OLED", "Help me implement XX functionality", or describes any embedded project with hardware components.
---

# MicroPython End-to-End Project Generation Skill

## Role

Given a user's project description, complete the full workflow from requirement clarification, component selection, code generation to device debugging, and finally run successfully on the user's device.

## Pre-checks (Step 1 — mandatory, all must pass to continue)

### 1. Check Python

```bash
python --version
```

- Successfully outputs version → continue
- Fails → stop and prompt:
  ```
  Python is missing. Please install it first: https://python.org
  Restart after installation.
  ```

### 2. Check requests library

```bash
python -c "import requests; print('requests OK')"
```

- Outputs `requests OK` → continue
- Fails → stop and prompt:
  ```
  The requests library is missing. Please run: pip install requests
  Restart after installation.
  ```

### 3. Check mpremote

```bash
mpremote --version
```

- Successfully outputs version → continue
- Fails → stop and prompt:
  ```
  mpremote is missing. Please run: pip install mpremote
  Restart after installation.
  ```

---

## Phase 0: Parse Links in User Input

If the user input contains links:

- **Must be a GitHub link**, otherwise prompt:
  ```
  Please upload the relevant files to GitHub first, then provide the link.
  ```
- Call the **fetch-doc skill** to retrieve the link content
- Extract from the content: hardware model, pin information, functional description
- Merge the extracted information into the requirement understanding to reduce follow-up questions

---

## Phase 1: Requirement Clarification

**List all missing information at once — do not ask multiple rounds of questions.**

Information that must be confirmed:

| Information | If not specified |
|---|---|
| Microcontroller model | Ask: ESP32 / RP2040 / Pico W? |
| Sensor/module list | Ask: Which hardware is needed? |
| Pin assignment for each module | See pin handling rules below |
| Functional description | Ask: What effect should be achieved? |
| Device serial port | Ask: COM port or /dev/ttyXXX? (Use `mpremote devs` to list) |

### Pin Handling Rules

**Step 1: Identify the board type, look up the built-in pin table**

| Board Type | Available GPIO (recommended) | Notes |
|---|---|---|
| ESP32 | 4,5,12,13,14,15,16,17,18,19,21,22,23,25,26,27,32,33,34(read-only),35(read-only),36(read-only),39(read-only) | GPIO0/2/15 are boot-sensitive, avoid using; GPIO6-11 connect to internal Flash, do not use |
| ESP32-S3 | 1-21,35-45 | GPIO19/20 are USB, avoid using |
| Pico / Pico W | GP0-GP28 (physical pins 1-40) | GP23/24/25/29 are used internally, avoid using |
| ESP8266 | D1(GPIO5),D2(GPIO4),D5(GPIO14),D6(GPIO12),D7(GPIO13) | GPIO0/2/15 are boot-sensitive |

**Step 2: If the board is not in the table above, or the user uses a custom board**

Ask additionally during requirement clarification:
```
You are using a non-standard development board. Please provide a pinout diagram link (GitHub link) or directly tell me which GPIO each module is connected to.
```
If the user provides a GitHub link → call the **fetch-doc skill** to extract pin information.

**Step 3: Pin assignment principles**

- I2C: Prefer the board's default I2C pins (ESP32: SCL=22/SDA=21, Pico: SCL=GP5/SDA=GP4)
- SPI: Prefer the default SPI pins
- Avoid using boot-sensitive pins
- If the user has already specified pins, use them directly without overriding

---

## Phase 2: Component Selection

**All components must be selected from drivers available on upypi. Do not use drivers not available on upypi.**

For each component, **search with multiple keywords in parallel**, then merge and deduplicate the results before making a decision:

```bash
# Example: Microphone module — search simultaneously
curl -s "https://upypi.net/api/search?q=mic"
curl -s "https://upypi.net/api/search?q=microphone"
curl -s "https://upypi.net/api/search?q=audio"
curl -s "https://upypi.net/api/search?q=i2s"
```

**Keyword expansion rules**:
- Use chip model (e.g., `inmp441`) + functional category (e.g., `mic`, `microphone`, `audio`) + interface type (e.g., `i2s`, `uart`, `i2c`) — at least 3 keywords
- For voice/AI projects, additionally search: `asr`, `tts`, `speech`, `llm`, `ai`
- Merge all search results, deduplicate, and display uniformly

- **Results found** → List package names, recommend the best match, explain the reason, wait for user confirmation
- **No results at all** → Inform the user, search for alternatives (also with multiple keywords), recommend a replacement component
- **Multiple results** → Recommend the best match, list other options

After component selection is confirmed, call the **upy-pkg-guide skill** to obtain the API usage for each component as a reference for code generation.

---

## Phase 3: Code Generation

### Project File Structure

```
(Device /lib/ directory)
/lib/
└── {driver}.py          ← Downloaded from upypi, transferred via mpremote

(Project directory, uploaded to the device root)
{functionA}_task.py      ← Single function module
{functionB}_task.py
main.py                  ← Unified scheduler
```

### Task File Specification

Each `xx_task.py` is responsible for one functional module and must include:

```python
# Initialization function, returns hardware object or None (on failure)
def init():
    try:
        print("[INIT] {module name} initializing...")
        obj = DriverClass(...)
        print("[OK] {module name} ready")
        return obj
    except Exception as e:
        print("[FAIL] {module name} init:", e)
        return None

# Core function function, independent try/except
def run(obj):
    try:
        result = obj.read()
        print("[DATA] {module name}:", result)
        return result
    except Exception as e:
        print("[ERROR] {module name}:", e)
        return None
```

### main.py Specification

```python
# File header 7 lines of comments
import time
from {functionA}_task import *
from {functionB}_task import *

# I2C/SPI scan diagnostics (required when using I2C)
from machine import I2C, Pin
i2c = I2C(0, scl=Pin(X), sda=Pin(Y))
devices = i2c.scan()
print("[I2C] Found:", [hex(d) for d in devices])
if not devices:
    raise RuntimeError("No I2C device found")

time.sleep(3)
print("FreakStudio: {project name} starting...")

# Initialize all modules
obj_a = init_a()
obj_b = init_b()

try:
    while True:
        if obj_a:
            run_a(obj_a)
        if obj_b:
            run_b(obj_b)
        time.sleep_ms(100)
except KeyboardInterrupt:
    print("Stopped by user")
except Exception as e:
    print("[FATAL]", e)
finally:
    print("Cleanup done")
```

---

## Phase 4: mpremote Automatic Debugging (Maximum 3 Times)

### Each Debugging Flow

```bash
# 1. Reset the device
mpremote connect {port} reset

# 2. Create /lib directory
mpremote connect {port} fs mkdir /lib

# 3. Upload driver files (upload dependency packages together)
mpremote connect {port} fs cp {driver}.py :/lib/{driver}.py

# 4. Upload task files and main.py
mpremote connect {port} fs cp {function}_task.py :{function}_task.py
mpremote connect {port} fs cp main.py :main.py

# 5. Run and capture output
mpremote connect {port} run main.py
```

### Output Parsing Logic

| Output Characteristic | Judgment | Fix Direction |
|---|---|---|
| No Traceback, no `[FAIL]` | **Success** | Notify user of completion |
| `ImportError` | Driver not uploaded or path error | Re-upload the corresponding file |
| `OSError` | Hardware not connected or pin error | Prompt user to check wiring, do not retry |
| `ValueError` | Parameter error | Modify code parameters and retry |
| `[FAIL]` | Locate the specific task | Modify the corresponding task file and retry |
| `AttributeError` | API call error | Re-read the driver source code, correct the call method |

### After 3 Failures

Output:
```
3 attempts made. Current blocker: {error description}
Possible cause: {analysis}
Suggested manual troubleshooting: {specific steps}
```

---

## Situations That Cannot Be Automatically Resolved (Directly Inform the User)

- Physical hardware wiring errors (software cannot detect; prompt to check wiring when OSError occurs)
- Driver incompatible with the current firmware version
- Special firmware required (ulab, lvgl, etc.)
- Device cannot be recognized by mpremote (driver/permission issue)
