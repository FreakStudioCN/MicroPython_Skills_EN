---
name: upy-scaffold
description: Step 3 — Project skeleton generation. Reads the project-manifest.json from the select-hw phase and generates a complete firmware/ directory skeleton based on the scheduling mode (Timer/asyncio/_thread). Trigger: automatically enters after upy-select-hw completes.
---

# Project Skeleton Generation Skill

## Role

Given `project-manifest.json` (phase: select-hw), determine the scheduling mode and generate a complete `firmware/` project skeleton. **Do not write business logic, do not fill in driver code, do not convert asynchronous drivers.**

---

## Pre-flight Check

```bash
python --version
python -c "import flake8; print('flake8 OK')"
```

No external dependencies (Python 3 standard library + flake8).

---

## Execution Steps

### Step 1: Approval Selection — Scheduling Mode + Extra Files

Read `project-manifest.json` and use **AskUserQuestion** to present a multi-select approval interface.

#### 1A: Scheduling Mode (single select, with recommended marking)

```
Recommendation rules (only used to mark Recommended, does not affect available options):
  devices contains display and includes LVGL → async
  requirements.network = wifi → async
  requirements.special_requirements contains "lcd" → async
  default → timer
```

AskUserQuestion:

```
header: "Scheduling Mode"
question: "Select scheduling mode (recommended option is marked):"
options:
  - Timer tick (Recommended) — ISR counting + main loop polling, suitable for pure sensor acquisition
  - asyncio — uasyncio native coroutines, suitable for WiFi / LVGL / LCD
  - _thread — multi-threading, suitable for blocking operations
```

_The mode is only used for selecting the main.py and task stub form during skeleton generation. Driver conversion (synchronous → asynchronous) is handled by `upy-generate`._

#### 1B: Extra Modules (multi-select)

```
header: "Extra Modules"
question: "Do you need to inject the following optional modules? (multi-select)"
multiSelect: true
options:
  - Logging system (lib/logger/*) — logging + rotating_logger, device-side log recording and rotation
  - Performance timer (lib/time_helper.py) — timed_function / timed_coro decorators, function timing
  - Maintenance task (tasks/maintenance.py) — GC check + idle callback
  - Deployment tool (tools/flash_device.py) — mpy compilation + firmware flashing + file upload
  - PC log reader (tools/read_device_log.py + log_report.py) — read device logs from PC and generate JSON report
```

#### 1C: User Custom Files (multi-select + free input)

```
header: "Custom"
question: "Do you need to generate additional custom files?"
options:
  - No extra files needed
  - Custom directories/files (please enter in Other, e.g., firmware/lib/my_utils.py, host/gui.py)
```

---

### Step 2: Process by Mode

#### Mode A: Timer tick (default)

**Do not fetch external documentation.** Inject `lib/scheduler/timer_sched.py`, ISR counting + main loop polling.

#### Mode B: asyncio

**WebFetch MicroPython asyncio official documentation** to confirm API usage:

```
WebFetch: https://docs.micropython.org/en/latest/library/asyncio.html
Extract: create_task, run, sleep_ms, gather, Event, Queue and other APIs
```

**Do not inject scheduler.py.** main.py uses `uasyncio` native API directly.

#### Mode C: _thread

**WebFetch Python _thread official documentation** to confirm API usage:

```
WebFetch: https://docs.python.org/3.5/library/_thread.html#module-_thread
Extract: start_new_thread, allocate_lock, exit and other APIs
```

**Do not inject scheduler.py.** main.py uses `_thread` native API directly.

---

### Step 3: Generate Project Skeleton

Call `init_scaffold.py`:

```bash
python G:/MicroPython_Skills/upy-scaffold/scripts/init_scaffold.py \
  --project-dir {project_dir} \
  --mode {timer|async|thread}
```

**Script automatically completes:**

| Step | File | Method |
|------|------|--------|
| board.py | pinout → BOARDS dictionary + query function | Generate |
| conf.py | requirements → sampling rate/log/watchdog constants | Generate |
| boot.py | WDT + emergency_exception_buf | Generate |
| main.py | Generate different entry points by mode | Generate |
| lib/logger/* | logging + rotating_logger + __init__ | Copy template |
| lib/time_helper.py | timed_function + timed_coro | Copy template |
| lib/scheduler/* | timer_sched.py + __init__ | **Timer mode only** |
| tasks/maintenance.py | GC check + error callback | Copy template |
| drivers/* | One stub package per device | Generate |
| tools/flash_device.py | .py→.mpy compilation + flashing + upload | Copy template |
| tools/read_device_log.py | PC-side device log reading | Copy template |
| tools/log_report.py | Log→JSON report parsing | Copy template |
| host/ | PC host code (no constraints) | .gitkeep |
| test/device/ | Device-side unittest test framework | .gitkeep |
| test/pc/ | PC-side test scripts | .gitkeep |
| build/firmware/ | .bin/.uf2/.hex firmware | .gitkeep |
| build/mpy/ | Compiled .mpy files | .gitkeep |
| firmware/assets/ | Device-side resource files (audio, etc.) | .gitkeep |
| README.md | Project name + BOM + pin table | Generate |
| LICENSE | MIT | Generate |
| .flake8 | F821/F401 exemptions + max-line=120 | Generate |
| — | flake8 verification | Automatically executed at end of script |

---

### main.py Forms for the Three Modes

main.py is generated by scaffold **only with hardware instantiation + scheduler framework**. Task registration is left for `upy-generate`.

**Timer:**
```python
from machine import Pin, I2C
from lib.scheduler.timer_sched import Scheduler
from tasks.maintenance import maintenance_tick

# Pin numbers from manifest.hardware.pinout
i2c = I2C(<bus_id>, scl=Pin(<scl>), sda=Pin(<sda>), freq=400000)
# ...

sc = Scheduler(timer_id=<port_timer_id>, tick_ms=100, idle_cb=maintenance_tick)
# TODO: upy-generate registers tasks here
sc.start()
```

`<port_timer_id>` must be selected according to the MicroPython port: only RP2/Pico/RP2040/RP2350 and Zephyr use `-1` virtual Timer; other MCU/ports default to `0` or another verified non-negative hardware Timer ID.

**asyncio:**
```python
import uasyncio as asyncio
from machine import Pin, I2C
from tasks.maintenance import maintenance_tick

i2c = I2C(<bus_id>, scl=Pin(<scl>), sda=Pin(<sda>), freq=400000)
# ...

async def main():
    # TODO: upy-generate creates async tasks here
    while True:
        maintenance_tick()
        await asyncio.sleep_ms(100)
asyncio.run(main())
```

**_thread:**
```python
import _thread
import time
from machine import Pin, I2C
from tasks.maintenance import maintenance_tick

i2c = I2C(<bus_id>, scl=Pin(<scl>), sda=Pin(<sda>), freq=400000)
# ...

# TODO: upy-generate starts threads here
while True:
    maintenance_tick()
    time.sleep_ms(100)
```

---

## Relationship with Other Skills

- ← `upy-select-hw`: Input manifest (mcu + pinout + bom + devices)
- → `upy-generate`: Pass complete skeleton + manifest, business code generation
- → `upy-wiring`: Pin assignment table → wiring diagram
- → `upy-diagram`: Code structure → architecture diagram
- → `upy-simulate`: PC-side full-process business simulation

---

## Hard Constraints

- **Do not generate business task files**: Only place general utilities (`maintenance.py` + `__init__.py`) under `tasks/`. Business tasks (sensor/display/alarm/network) are created by `upy-generate`
- **Do not convert drivers**: Driver synchronous/asynchronous conversion is the responsibility of `upy-generate`
- **asyncio / _thread modes do not inject scheduler.py**: Use native API directly, no extra wrapper
- **Timer mode uses existing Scheduler.py reference implementation**: ISR only counts, main loop executes
- **board.py does not perform hardware initialization**: Only stores constant mappings; instance creation is in main.py
- **conf.py does not contain sensitive data**: No Wi-Fi passwords, API Keys
- **Automatic flake8 at end of generation**: Script automatically verifies at the end; prints warning if it fails
