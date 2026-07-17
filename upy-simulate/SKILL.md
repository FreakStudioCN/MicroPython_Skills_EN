---
name: upy-simulate
description: Full-process business simulation on PC. Reads all code under firmware/ as context. LLM autonomously designs scheduling/device/visualization schemes, generates simulation scripts under test/pc/, validates with flake8 + pylint, then runs. Triggered manually after upy-generate completes, or called after upy-autofix repairs.
---

# PC Full-Process Simulation Skill

## Role Definition

Reads all code under `firmware/` as context. **The LLM autonomously designs** the scheduling scheme, Mock device assembly, visualization format, and data scenarios, generating `test/pc/sim_main.py` (and any required auxiliary files) to simulate the complete business process without relying on hardware.

**This is a code generation skill, not a runtime framework skill.** Do not pre-write a scheduler or hardcode a CLI format — the LLM reads the full codebase and generates a project-specific simulation script.

---

## Core Constraints

- **Do not modify any files under `firmware/`** (unless a definite syntax error or logic bug is found)
- **All simulation code goes into `test/pc/`**, importing firmware/ modules via `sys.path.insert(0, ...)`
- **Classes and functions from firmware/ may be inherited or wrapped**, but the original files must not be modified
- **flake8 + pylint validation must pass**; if errors exist, fix and re-validate in a loop until error-free
- **Must ask the user via AskUserQuestion before running**

---

## Execution Steps

### Step 1: Read Full Context

The LLM must read all of the following files without omission:

```
firmware/
├── main.py              → Hardware init layer + callback wrappers + task registration + data flow (_data dict keys)
├── conf.py              → Sampling rate, refresh interval, alarm thresholds, constants
├── project-manifest.json → mode (timer|async|thread), devices[], mcu{}
├── tasks/
│   ├── *.py             → Each task's function signature, parameters, return values
├── drivers/
│   └── */mock.py        → Each Mock class's __init__ parameters, methods, _raise_on exception injection points
├── lib/
│   ├── scheduler/*.py   → Scheduler API (add_task, start, etc.)
│   ├── logger/*.py      → Logging system (device-side log rotation may not be needed for simulation)
│   └── time_helper.py   → Decorators (CPython-compatible branch already exists)
├── board.py             → Pin mapping (reference only, not needed for simulation)
├── boot.py              → Boot sequence (skip for simulation)
└── assets/              → Device-side resource files (audio, images, etc.); simulation may need to reference them
```

### Step 1B: Project Type Classification + Extract Simulatable Interfaces

After reading all files, the LLM must perform the following two analyses:

#### 1. Automatic Project Type Classification

Scan all code under firmware/ for the following signals and output the classification result as a comment at the top of `sim_main.py`:

| Signal | Classified As |
|------|--------|
| `import network` / `WLAN` / `socket` / `umqtt` / `urequests` / `bluetooth` / `NFC` | **IoT / Network** |
| `PWM` / `Servo` / `Stepper` / `encoder` / `H-bridge` / motor driver chip model | **Motor Control** |
| display device + `Pin.IRQ` / button / touch / rotary encoder / keypad driver | **GUI Interaction** |
| `conf.py` contains `*_THRESHOLD` / `*_ALARM` constants | **Alarm Monitoring** |
| `_thread` / `uasyncio` and ≥ 3 concurrent tasks | **Multi-tasking Concurrency** |
| `uart` / `CAN` / `RS485` / `Modbus` / external communication protocol | **Industrial Communication** |

A project can belong to multiple types. Format:
```python
# @ProjectTypes: sensor_monitoring, alarm_monitoring, gui_display
```

#### 2. List of Simulatable Interfaces

Do not only read `drivers/*/mock.py`; extract all **simulatable external boundaries** from the entire firmware codebase:

- **Network**: `wlan.isconnected()`, `wlan.ifconfig()`, `socket.send()/recv()`, `umqtt.publish()/subscribe()/check_msg()`, `urequests.get()/post()`
- **Motor**: `pwm.duty()`, `servo.angle()`, `stepper.step()`, encoder reading, limit switch state
- **Input**: `pin.value()`, `pin.irq()` trigger timing, debounce period
- **Protocol**: MQTT CONNECT/SUBACK/PUBACK status, HTTP response status code

For each simulatable interface, record: **the task that calls it, normal behavior, and possible failure modes**.

---

### Step 2: LLM Autonomous Design

Based on the full context from Step 1, the LLM autonomously decides the following 5 things. **Do not pre-set a framework or hardcode a solution.**

#### 2A. Scheduling Scheme

| manifest mode | CPython Alternative | Description |
|--------------|-----------------|------|
| timer | `while` + `time.sleep(tick_ms/1000)` + manual counter | LLM must generate a `SimScheduler` class with the same API as `Scheduler` in `firmware/lib/scheduler/timer_sched.py` (`add_task`, `start`), removing `machine.Timer` ISR |
| async | `asyncio.run()` | CPython 3.7+ native asyncio, highly compatible with MicroPython `uasyncio` API; LLM can use it directly |
| thread | `threading.Thread(target=loop, daemon=True)` | CPython native threading; main thread keeps alive with `time.sleep` |

**Special note for timer mode**: The `SimScheduler` generated by the LLM must be interface-compatible with the original `Scheduler`. CPython uses a `time.sleep` loop instead of an ISR, incrementing all task counters on each tick and executing callbacks that are due.

#### 2B. Mock Device Assembly + Data Generator (Core)

**This is the most critical design decision for upy-simulate.** Mocks must not be static values — `MockSHT30(measure=(25.0, 60.0))` returns the same data every time, with no variation between ticks, and business logic branches will never be tested.

**Correct approach: Data generator = function of tick.**

The LLM reads each `drivers/*/mock.py` to determine:
- What parameters `__init__` accepts (e.g., `measure=(25.0, 60.0)`, `temp=25.5`)
- What `_raise_on` values are supported (e.g., `'measure'`, `'read_compensated_data'`)
- What methods are provided

**Data injection mechanism** (without modifying firmware/ code):

```python
# Define data generators in sim_main.py (tick → return value)
def gen_sht30(tick):
    """Temperature 22-28°C sinusoidal fluctuation, period 60s"""
    import math
    temp = 25.0 + 3.0 * math.sin(2 * math.pi * tick / 600)
    hum = 60.0 + 10.0 * math.cos(2 * math.pi * tick / 600)
    return (temp, hum)

def gen_bmp280(tick):
    """Pressure 1000-1020 hPa, with random noise"""
    import random
    press = 101000 + 1000 * math.sin(2 * math.pi * tick / 300) + random.uniform(-200, 200)
    return (25.0, press, 55.0)  # temp, pressure, humidity

# Before each sensor callback executes, update the mock's internal state
def _sensor_cb():
    sht30._measure = gen_sht30(tick_count)   # ← Dynamic injection
    bmp280._read_data = gen_bmp280(tick_count)
    sensor_read(sht30, bmp280, _data)
```

**Mock state update must happen before the callback**, ensuring that when the task function calls `mock.measure()`, it receives the current tick's data.

#### 2B+. Mock Design Patterns (by Project Type)

The following are Mock design references for **non-sensor projects**. The LLM must select the corresponding pattern based on the classification result from Step 1B.

**Network Mock (state-machine-driven cross-tick behavior):**

```python
class MockWLAN:
    """Key: state machine — connecting → connected → disconnected → reconnecting"""
    def __init__(self, **kwargs):
        self._connected = kwargs.get('connected', True)
        self._rssi = kwargs.get('rssi', -40)
        self._raise_on = kwargs.get('_raise_on', None)  # 'connect', 'disconnect'
        self._connect_delay_ticks = kwargs.get('connect_delay', 0)

    def isconnected(self):
        if self._raise_on == 'isconnected':
            raise OSError('WLAN error')
        return self._connected

    def ifconfig(self):
        return ('192.168.1.100', '255.255.255.0', '192.168.1.1', '8.8.8.8')

# Data generator: network_state(tick) → (connected, rssi, raise_on)
def gen_network_disconnect(tick):
    if 10 <= tick <= 14:
        return (False, -80, None)     # Disconnected for 5 ticks
    elif tick == 15:
        return (True, -40, None)      # Reconnected
    return (True, -40, None)
```

**Protocol Mock (above socket layer, simulating application protocol behavior):**

```python
class MockMQTT:
    """Simulate MQTT protocol behavior, not TCP socket"""
    def __init__(self, **kwargs):
        self._connected = False
        self._subscriptions = {}
        self._pending_messages = []      # broker → device message queue
        self._connect_fail_until = kwargs.get('_connect_fail_until', -1)
        self._raise_on = kwargs.get('_raise_on', None)

    def connect(self):
        if self._raise_on == 'connect':
            raise OSError('Connection refused')
        self._connected = True

    def check_msg(self):
        if self._raise_on == 'check_msg':
            raise OSError('Socket error')
        return self._pending_messages.pop(0) if self._pending_messages else None

# Scenario data: inject a specific sequence of MQTT messages into _pending_messages
# Scenario: server sends erroneous command → verify parsing rejection
```

**Motor Mock (cross-tick cumulative physical state):**

```python
class MockStepper:
    """Key: cumulative physical state across ticks, not reassigned every tick"""
    def __init__(self, **kwargs):
        self._position = kwargs.get('position', 0)
        self._target = kwargs.get('target', 0)
        self._speed = kwargs.get('speed', 100)         # steps/s
        self._stall_at = kwargs.get('_stall_at', None)  # Stall at a certain position
        self._limit_switch = kwargs.get('_limit_switch', None)
        self._raise_on = kwargs.get('_raise_on', None)

    def step(self, direction, steps=1):
        if self._raise_on == 'step':
            raise RuntimeError('Motor driver error')
        if self._stall_at is not None and self._position >= self._stall_at:
            raise RuntimeError('Motor stalled at position {}'.format(self._position))
        self._position += steps * direction

    def position(self):
        return self._position

# Motor scenarios don't use "data generators"; use "event injection":
# def _scenario_motor_stall(sht30, bmp280, tick):
#     if tick == 10:
#         stepper._stall_at = 300  # ← Inject stall event
```

**Input Device Mock (event queue + time-series replay):**

```python
class MockButton:
    """Key: pre-load event queue, replay by tick, simulate real physical timing"""
    def __init__(self, **kwargs):
        self._event_queue = kwargs.get('events', [])  # [(tick, value), ...]
        self._current_tick = 0

    def set_tick(self, tick):
        self._current_tick = tick

    def value(self):
        for tick, val in reversed(self._event_queue):
            if self._current_tick >= tick:
                return val
        return 0

# Scenario: 10 rapid presses (100ms interval) to verify debounce:
# events = [(1,1), (2,0), (3,1), (4,0), (5,1), (6,0), ...]
```

**Actuator Mock (record state flip history, not just current value):**

```python
# Standard MockBuzzer/MockLED are sufficient (_state + on/off + value)
# But the LLM should verify in the scenario: after on() is called, value() is True at the expected time
```

**Industrial Communication Mock (Modbus / RS485 / CAN):**

```python
class MockModbus:
    """Simulate Modbus RTU slave device"""
    def __init__(self, **kwargs):
        self._registers = kwargs.get('registers', {})  # {addr: value}
        self._raise_on = kwargs.get('_raise_on', None)  # 'read', 'write', 'timeout'
        self._response_delay = kwargs.get('response_delay', 0)

    def read_holding_registers(self, addr, count):
        if self._raise_on == 'read':
            raise OSError('Modbus timeout')
        return [self._registers.get(addr + i, 0) for i in range(count)]

# Scenario: slave device offline → read_holding_registers times out repeatedly → master enters safe mode
```

#### 2C. Visualization Format

**CLI + rich is preferred. tkinter GUI is an optional fallback.**

tkinter has known issues on Windows: `StringVar.set()` / Canvas operations may not refresh within the `root.after()` callback chain, and `sys.stdout` redirection can interfere with the event loop. CLI + rich has no such issues and allows faster development iteration.

| Project Feature | Preferred Solution | Description |
|---------|---------|------|
| All projects | **`rich` library (Live/Table/Panel/Layout)** | Terminal dynamic dashboard, cross-platform, no rendering issues |
| Has display device (OLED/LCD/TFT) | rich `Panel` + `Text` to simulate screen content | Panel title indicates device model, content area updates virtual screen text in real-time |
| Has actuator (Buzzer/LED/Relay) | rich `Table` inline status markers | `ON`/`OFF` with color highlighting (green=active, gray=standby) |
| Log output | rich `Live` fixed bottom area scrolling | Does not hijack `sys.stdout`, writes directly to Panel |

**The LLM generates CLI mode (rich) by default.** `--mode gui` (tkinter) is only generated when the user explicitly requests it, and the following precautions must be followed:
- Do not use `sys.stdout` redirection; instead, call `log_widget.insert()` directly
- Call `root.update_idletasks()` at the end of the `root.after()` callback to force a refresh
- Wrap each `scheduler_tick()` in `try/except` to prevent silent failures

GUI mode is marked at the top of `sim_main.py`: `# @GUI: experimental — prefer --mode cli for reliable output`

#### 2D. Data Scenarios (Temporal Evolution + Coverage Framework)

The LLM must design **at least one scenario for each coverage dimension** based on the project classification result from Step 1B, the constants in `conf.py`, and the logic branches in `tasks/*.py`.

##### 2D-1. Coverage Dimension Table (by Project Type)

The LLM must design corresponding scenarios for each project classification across the following dimensions:

| Project Type | Coverage Dimension | Scenario Design Points |
|----------|---------|------------------|
| **Sensor Monitoring** | Normal data flow | Data fluctuates within thresholds; verify scheduling + data flow integrity |
| | Threshold crossing (high) | Data crosses from below to above threshold; verify conditional trigger |
| | Threshold crossing (low) | Same, opposite direction |
| | Sensor failure | Inject exception via `_raise_on`; verify independent fault tolerance |
| | Sensor recovery | Recovery after failure; verify automatic recovery |
| | Multi-sensor partial failure | A fails, B is normal; verify no mutual interference |
| **Alarm/Actuator** | Alarm trigger → Actuator ON | Verify buzzer/LED/relay activation timing |
| | Alarm cooldown | Verify no repeated trigger during cooldown period |
| | Alarm recovery → Actuator OFF | Actuator turns off after condition clears |
| | Alarm repeated trigger | Trigger again after cooldown period (requires ticks > cooldown_ticks × 2) |
| **IoT/Network** | Connection loss | `isconnected() → False`; verify offline caching/queuing |
| | Connection recovery | Reconnect + queue retransmission |
| | Send timeout | `send() → timeout`; verify retry logic |
| | Receive anomalous data | Format error / checksum failure → verify rejection |
| | Server disconnect | Broker actively disconnects; verify reconnection |
| | Weak signal | RSSI continuously drops; verify degradation strategy |
| **Motor Control** | Normal motion | Target position reached; verify motion completion |
| | Stall/Stuck | Position no longer changes; verify stall detection |
| | Limit trigger | Reaches limit switch; verify stop logic |
| | Emergency stop | Stop command mid-motion; verify interruption |
| **GUI/Human Interaction** | Normal operation path | Complete menu browsing / workflow |
| | Rapid consecutive input | Multiple events within N ms; verify debounce |
| | Long press vs short press | Different durations; verify press duration parsing |
| | Invalid input | Out-of-bounds data; verify rejection |
| **Industrial Communication** | Slave device timeout | Modbus/CAN timeout → safe mode |
| | Slave device returns error | Error code → error handling |
| | Bus disconnection | Physical layer disconnection → alarm |
| **Multi-tasking Concurrency** | All tasks due simultaneously | Verify no starvation |
| | One task takes too long | Verify impact on other tasks |

##### 2D-2. Minimum Number of Scenarios

Each project must generate at least **N scenarios**, where `N = number of project classifications × 2 + 1`. Correspondence:

| Project Classification Count | Minimum Scenarios |
|-----------|-----------------|
| Sensor only | 3 (normal + failure + threshold trigger) |
| Sensor + Alarm | 5 |
| Sensor + Alarm + Network | 7 |
| Sensor + Motor + Alarm | 8 |

##### 2D-3. Scenario Self-Check + @Coverage Comment (Mandatory)

After generating each scenario, the LLM **must** declare in the code with a `@Coverage` comment which classification's which dimension it covers, and at which tick it expects which event to trigger. The comment is written into the scenario function's docstring or file header:

```python
def _scenario_temp_rising(sht30, bmp280, tick):
    """@Coverage: [alarm] TEMP_HIGH_THRESHOLD(35.0) crossed at tick ~21
       @Coverage: [alarm] buzzer.on() + led.on() called at tick ~21
       @Coverage: [alarm] cooldown active ticks 21-50 — no repeat trigger within 30 ticks
    """
```

The LLM **must check each one**: whether each scenario can actually trigger the declared events within its default `--ticks` value. The conclusion is written at the top of `sim_main.py`:

```python
# @CoverageReport:
#   normal:              [sensor] data flow only — no threshold crossed
#   temp_rising:         [alarm] temp high at tick 21, buzzer+LED ON
#   temp_dropping:       [alarm] temp low at tick 23
#   intermittent:        [sensor] OSError at ticks 3,6,9,...
#   sensor_death:        [sensor] permanent failure from tick 5
#   ⚠ GAP: humidity threshold (80% HIGH / 20% LOW) not covered by any scenario
#   ⚠ GAP: alarm recovery (all clear) needs ticks > 51 with temp_rising
```

**If the LLM finds that a certain coverage dimension has no corresponding scenario in the current 5-scenario template, it must add a new scenario.** If the project does not involve a certain dimension (e.g., no network module), mark it as "N/A" in the report.

##### 2D-4. Scenario Data Design Reference

Each scenario is a **Python function `(tick: int) → side effect`** (updating mock internal state). The LLM autonomously chooses from the following design methods:

- **Mathematical expressions**: sine wave, linear rise/fall, exponential decay
- **Event injection**: modify `mock._stall_at`, `mock._connected`, `mock._event_queue`, etc. at specific ticks
- **Exception injection**: set `mock._raise_on = 'xxx'` at specific ticks
- **Lookup table sequence**: predefined `[(tick1, val1), (tick2, val2), ...]` time series

#### 2E. User-Defined Scenarios (Natural Language → Mock API)

**Trigger condition**: The user provides a natural language description when invoking the skill, or the user selects a custom scenario in the AskUserQuestion from Step 5.

##### 2E-1. User Input Examples

- "I want to test the scenario where WiFi disconnects for 5 seconds and then automatically reconnects"
- "If the motor gets stuck at position=300, will the system stop?"
- "If the user rapidly presses the button 10 times, will the menu jump around?"
- "What happens if BMP280 and SHT30 fail simultaneously?"
- "MQTT broker suddenly disconnects and then recovers after 3 seconds"

##### 2E-2. LLM Mapping Rules

The LLM must map the user's natural language to the corresponding mock's API:

```
User Description              →  LLM Parsing                   →  Mock API Mapping
──────────────────────────────────────────────────────────────────────────
"WiFi disconnect 5 seconds"   →  Network connection lost, duration 5000ms →  wlan._connected = False
                                                                 for tick in range(t, t+5)

"Auto reconnect"              →  Restore connection             →  wlan._connected = True
                                                                 wlan._raise_on = None

"Motor stuck at position=300" →  Stall event                    →  stepper._stall_at = 300

"Rapid button press 10 times" →  100ms interval pulse sequence  →  button._event_queue =
                                                                  [(t,1), (t+1,0), (t+2,1), ...]

"Two sensors fail simultaneously" →  Dual raise_on injection    →  sht30._raise_on = 'measure'
                                                                 bmp280._raise_on = 'read_compensated_data'

"MQTT broker disconnect then recover" →  Disconnect + delay + reconnect →  mqtt._raise_on = 'connect' ticks 5-7
                                                                 mqtt._raise_on = None from tick 8

"Modbus slave timeout 3 times" →  read fails consecutively       →  modbus_reg._raise_on = 'read' for ticks 1-3
```

##### 2E-3. Mapping Failure Handling

If the LLM cannot map the user's description to an available API of an existing mock, **it must output feedback instead of silently skipping**:

```
⚠ Cannot map: "Motor torque exceeded" — MockStepper does not provide torque simulation.
   Available injection points: _stall_at (position stall), step() raises RuntimeError
   Suggestion: Change to "Motor stalls at position=200", or add a _torque_limit attribute to MockStepper.

⚠ Cannot map: "WPA3 handshake failure" — MockWLAN does not simulate the authentication layer.
   Available injection points: _raise_on='connect' (connection refused), _connect_delay_ticks (connection time)
   Suggestion: Change to "WiFi connect fails 3 times then gives up"
```

##### 2E-4. Generation Rules

- The scenario function after successful mapping is named `_scenario_custom_N`, with key `custom_N`
- Automatically added to the `SCENARIOS` dict
- Add a comment to the top of `sim_main.py`:
  ```python
  # @CustomScenario: custom_1 = "WiFi disconnect 5s then reconnect" → wlan disconnect ticks 10-14, reconnect tick 15
  # @CustomScenario: custom_2 = "Double button press" → button events at ticks 3,4,8,9
  ```

##### 2E-5. Preliminary Inquiry (Optional)

If the user does not provide a custom description when invoking the skill, the LLM may insert a question before Step 5:
```
header: "Custom Scenario"
question: "Do you need a custom test scenario? If so, please describe (e.g., WiFi disconnect, motor stall, rapid button press, etc.)"
options:
  - Use preset scenarios — Automatically generate coverage list based on project type
  - Custom scenario — Enter natural language description in Other
```

---

### Step 3: Generate Simulation Code

Generate one or more `.py` files into `test/pc/`:

```
test/pc/
├── sim_main.py          # Main entry (Mock assembly + scheduling + visualization + scenario control)
└── sim_scheduler.py     # Required for timer mode (SimScheduler class)
```

**`sim_main.py` must include the following in its header comment:**
```python
# @Generated by upy-simulate
# @Date: <generation time>
# @ProjectTypes: sensor_monitoring, alarm_monitoring, gui_display
# @Description: PC full-process simulation entry point
#   Does not depend on MicroPython runtime or hardware devices
#   Imports firmware/ task functions and mock drivers via sys.path
# @CoverageReport:
#   normal:              [sensor] data flow only — no threshold crossed
#   temp_rising:         [alarm] temp high at tick ~21, buzzer+LED ON
#   temp_dropping:       [alarm] temp low at tick ~23
#   intermittent:        [sensor] OSError at ticks 3,6,9,...
#   sensor_death:        [sensor] permanent failure from tick 5
```

**Code constraints:**
- Import firmware/ via `sys.path.insert(0, os.path.join(...))`
- Callback wrapping method must be consistent with `firmware/main.py`
- Task registration method must be consistent with `firmware/main.py`
- `_data` dict keys must be consistent with `firmware/main.py`
- Accept command-line arguments `--ticks` (number of run cycles), `--scenario` (scenario), `--mode` (CLI/GUI, default CLI)
- **Each scenario must define a data generator function** `gen_xxx(tick) → value`; the return value changes as tick changes
- **Before the sensor callback executes, the mock's internal state must be updated**: `mock._measure = gen_sht30(tick_count)`, ensuring the task function receives the current tick's data
- **Do not create a Mock once and never update its internal state**

---

### Step 4: flake8 + pylint Validation

```bash
# flake8
python -m flake8 test/pc/sim_main.py --max-line-length=120

# pylint (relaxed for embedded projects)
python -m pylint test/pc/sim_main.py --max-line-length=120 --disable=missing-docstring,too-few-public-methods
```

**Validation loop:**
```
Generate code → flake8 → errors → fix → re-validate
       → no errors → pylint → warnings/errors → fix → re-validate
                        → no errors → Step 5
```

After each fix, both tools must be re-run to confirm. Maximum 5 fix rounds; if exceeded, report to the user.

---

### Step 5: Ask the User

After flake8 + pylint both pass, use **AskUserQuestion** to inquire.

#### 5a. Pre-Run Analysis (LLM must complete before asking)

1. Detect project type (reuse classification result from Step 1B)
2. If alarm/threshold logic exists, calculate the **minimum recommended ticks**:
   - `min_recommended_ticks = max(ticks needed for each threshold crossing) + ALARM_COOLDOWN_MS / SAMPLE_INTERVAL_MS × 2`
   - Ensure coverage of at least: trigger → cooldown → second trigger (complete cycle)
3. From the generated scenario list, distinguish between **"data flow only"** and **"covers business branches"** categories

#### 5b. AskUserQuestion Content

**The default mode is always CLI (rich).**

```
header: "Simulation Run"
question: "PC simulation script has passed syntax validation. Start running?

  Project types: [sensor ×2, alarm, OLED display, no network]
  Generated scenarios: normal, temp_rising, temp_dropping, intermittent_failure, sensor_death
  Recommended: temp_rising --ticks 60 (covers complete alarm cycle: trigger→cooldown→recovery)
  The normal scenario only verifies data flow and does not trigger any business branches.
"
options:
  - Run recommended scenario (Recommended) — temp_rising --mode cli --ticks 60
  - Run normal scenario — Only verify data flow + scheduling
  - Switch scenario to run — Enter in Other (e.g., --mode cli --scenario intermittent_failure)
  - Run GUI mode (experimental) — Enter in Other (e.g., --mode gui --scenario temp_rising)
  - Custom scenario — Enter natural language description in Other (e.g., "WiFi disconnect 5 seconds")
  - Do not run for now — Keep test/pc/sim_main.py, run manually later
```

---

### Step 6: Run + Evaluation

```bash
python test/pc/sim_main.py --ticks {N} --mode {mode} --scenario {scenario}
```

**Observe during run:**
- Any Python Traceback → FAIL (LLM fixes sim_main.py, returns to Step 3)
- Are task functions executing normally in the output → Check per-tick logs
- Is business logic triggered → Check against the scenario's `@Coverage` comments
- Is data flow complete → `_data` dict keys should be correctly read/written by each task

**Evaluation rules (three levels):**
```
Traceback present                              → FAIL  → Fix sim_main.py, return to Step 3
No Traceback, all @Coverage declared events
  occur within expected tick range              → PASS  → End of process
No Traceback, some @Coverage declared events
  did not occur (scenario conservative or ticks insufficient) → WEAK_PASS → Prompt user to switch scenario
```

**After the run completes, the LLM must output a coverage summary report by project type:**
```
=== Simulation Coverage Report ===
☑ [sensor] Sensor reading:             30/30 ticks normal
☑ [sensor] Sensor fault tolerance:          Triggered → SHT30 read failed at ticks 3,6,9,12,...
☑ [alarm]  Alarm check:                30/30 ticks normal
☑ [alarm]  High temperature alarm trigger:            Triggered → [alarm] temp high at tick 21
☑ [alarm]  Actuator activation:              Triggered → Buzzer ON, LED ON at tick 21
☐ [alarm]  Alarm recovery (all clear):   Not triggered → ticks insufficient, need > 51
☐ [alarm]  Low temperature alarm trigger:            Not triggered → current scenario does not cover
☐ [alarm]  Humidity alarm trigger:            Not triggered → no scenario covers 80%/20% humidity threshold
☐ [network] Network fault tolerance:               N/A → Project has no network module
☐ [motor]  Motor control:               N/A → Project has no motor module
==================================
Result: WEAK_PASS — 3/5 coverage dimensions tested, 2 dimensions not covered.
  Suggestion: python test/pc/sim_main.py --scenario temp_dropping --ticks 30
  Suggestion: Add humidity_high scenario to cover 80% humidity threshold
```

On WEAK_PASS, the LLM must **proactively suggest specific re-run recommendations** (down to command-line arguments or new scenario names).

---

## Scheduling Mode Detailed Reference

### Timer Mode: SimScheduler Design

After reading the `Scheduler` class from `firmware/lib/scheduler/timer_sched.py`, the LLM generates `SimScheduler`:

```
Original Scheduler                    SimScheduler
───────────                           ─────────────
machine.Timer ISR driven              time.sleep loop driven
ISR only increments tick_cnt          Loop body manually increments tick_cnt
Main loop checks tick_cnt → executes cb   Main loop checks tick_cnt → executes cb
start() never returns                 start() supports max_ticks limit
No visualization hooks                on_tick callback pushes state
```

Key interfaces must be consistent: `add_task(callback, interval_ms, name=None) → tid`, `start()`

### Async Mode: Use CPython asyncio Directly

```python
import asyncio
# Task function signature is async def xxx_coro():
# Register directly with asyncio.create_task()
# Start with asyncio.run(main_coro())
```

MicroPython `uasyncio` and CPython `asyncio` APIs are highly compatible; adaptation is usually unnecessary.

### Thread Mode: Use CPython threading Directly

```python
import threading
# Task function is a while True loop
# threading.Thread(target=sensor_loop, daemon=True).start()
# Main thread keeps alive with time.sleep
```

---

## Visualization Examples (For LLM Reference, Not Mandatory)

### Preferred: rich CLI Terminal Dashboard

```python
from rich.live import Live
from rich.table import Table
from rich.panel import Panel
from rich.layout import Layout
from rich.text import Text

# Main layout: sensor data table + virtual screen + status panel
layout = Layout()
layout.split_column(
    Layout(Table(...), name="data"),
    Layout(Panel("", title="SSD1306 OLED 128x64"), name="display"),
)

# Update every tick
with Live(layout, refresh_per_second=10, transient=False) as live:
    while running:
        # Update sensor Table
        table.add_row(str(tick), f"{temp:.1f}°C", f"{hum:.1f}%", ...)

        # Update virtual OLED Panel (simulate display device)
        oled_content = f"T: {temp:.1f}°C\nH: {hum:.1f}%\n{'ALARM!' if alarm else 'OK'}"
        layout["display"].renderable = Panel(oled_content, title="SSD1306")

        # Actuator status: use rich Text with color markers
        buzzer_text = Text("Buzzer: ", style="bold")
        buzzer_text.append("ON", style="bold red") if buzzer_on else buzzer_text.append("OFF", style="dim")
```

Use `rich.Panel` to simulate virtual screen content for display devices. Use `rich.Text` with color markers for actuator status (green=normal, red=active, gray=standby). Do not hijack `sys.stdout`; append logs directly to a separate `log_lines` list and write them to a bottom Panel during `Live` refresh.

### Optional: tkinter GUI (Experimental)

Only generated when the user explicitly requests it. When generated, the code header must be marked with `# @GUI: experimental — prefer --mode cli for reliable output`.

```python
import tkinter as tk
root = tk.Tk()
root.title("PC Simulation [EXPERIMENTAL GUI]")

# Key rules:
# 1. Do not redirect sys.stdout → call log_widget.insert() directly
# 2. Call root.update_idletasks() after each update
# 3. Wrap scheduler_tick() in try/except to prevent silent failures

def scheduler_tick():
    try:
        sc.execute_one_tick()
        render_oled()
        update_status()
        root.update_idletasks()  # ← Force refresh
    except Exception as e:
        print(f"[GUI ERROR] {e}", file=sys.stderr)
    if sc._running:
        root.after(sc._tick_ms, scheduler_tick)

root.after(100, scheduler_tick)
root.mainloop()
```

---

## Relationship with Other Skills

```
upy-generate
    │
    ├─→ upy-simulate (manual trigger)
    │       test/pc/sim_main.py → PASS → End of process / proceed to deploy
    │
    └─→ upy-deploy
            │
            └─→ FAIL → upy-autofix
                          │
                          └─→ After fix, optional call: python test/pc/sim_main.py --ticks 30
                                 Quickly verify fix effect → then deploy
```

- ← `upy-generate`: Provides complete firmware/ + manifest
- → Can be called by `upy-autofix`: Verify on PC after fix, then flash
- Parallel optional with `upy-deploy`: User can simulate first, then deploy

---

## Strong Constraints

- **Do not modify any files under `firmware/`** (unless a definite bug is found)
- **All new code goes into `test/pc/`**
- **flake8 + pylint validation must pass**, otherwise loop fix
- **Must ask the user via AskUserQuestion before running**
- **Scheduling scheme is autonomously decided by the LLM based on manifest.mode**, no pre-set framework
- **Visualization format is autonomously decided by the LLM based on the project's device combination**, no hardcoding
- **CLI + rich is the preferred mode**: All projects generate CLI mode (rich Live/Table/Panel) by default, data dynamically refreshes in the terminal over time
- **For projects with display devices, use rich Panel to simulate a virtual screen**: Panel title indicates device model, content area updates the current display text every tick
- **Do not hijack sys.stdout**: Logs are written directly to a rich Panel or via independent `loguru`/`print`, avoiding redirection interference with output timing
- **tkinter GUI is an optional experimental mode**: Only generated when the user explicitly requests it; when generated, the code header must be marked with `# @GUI: experimental`, and GUI safety rules must be followed (no stdout redirection, update_idletasks, try/except wrapping)
- **Before the sensor callback executes, the mock's internal state must be updated** (e.g., `mock._measure = gen_sht30(tick_count)`), allowing the task function to receive dynamic data without modifying firmware/ code
- **The simulation entry point `sim_main.py` must support the three command-line arguments `--ticks`, `--scenario`, and `--mode`**, where `--mode` can be `cli|gui`
- **Do not simulate driver internal details** (sensor protocols, I2C timing, etc.); only verify business logic through Mock object return values and exception injection
- **Step 1B is mandatory**: The LLM must perform project type classification and output `@ProjectTypes` at the top of `sim_main.py`
- **Step 2D is mandatory**: Each scenario must include a `@Coverage` comment declaring the covered dimension; the LLM must check each scenario to ensure it can actually trigger the declared events within the default ticks, and write the conclusion in `@CoverageReport`
- **Step 2D is mandatory**: Number of scenarios ≥ number of project classifications × 2 + 1; must cover every coverage dimension for each project classification (see coverage dimension table); if the existing template is insufficient, a new scenario must be added
- **Step 2E is mandatory**: When the user provides a natural language description, the LLM must map it to the mock API; if mapping fails, feedback must be output explaining the reason and suggesting alternatives
- **Step 5 is mandatory**: AskUserQuestion must specify the project type + recommend a scenario that covers business logic branches + mark which scenarios only verify data flow
- **Step 6 is mandatory**: Evaluation is divided into three levels: PASS / WEAK_PASS / FAIL; after running, a coverage summary report by project type must be output; on WEAK_PASS, alternative scenarios must be proactively suggested
