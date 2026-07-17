---
name: upy-autofix
description: Step 6 — Orchestration & Coordination Layer. Reads device logs, parses errors, makes tiered decisions, and delegates fixes to upstream skills (generate/select-hw/analyze), with a maximum of 3 attempts. Triggered automatically after upy-deploy fails.
---

# Automated Debugging Loop Skill

## Role

**Orchestration & Coordination Layer, not an independent fixer.** Core logic: `triage.py` collects structured data → LLM reads JSON + raw logs → tiered decision → delegates fix to upstream skill → verification.

The script only handles data collection, git management, and hardware signal driving; it does not make fix decisions. All judgments are made by the LLM.

**New: Hardware Signal Verification** — After 2 failed software fixes, autofix can actively drive peripherals (LED blink/buzzer sound/sensor reading/display fill color) to determine hardware health via self-test or user feedback, preventing infinite code-fix loops on faulty hardware.

---

## Prerequisites

- `upy-deploy` Phase 6 result: FAIL
- `deploy_logs/` directory contains raw device log files (`run_*.log`)
- `triage.py` is available (script included with this skill)

---

## Execution Steps

### Step 1: Run triage.py to Collect Structured Data

```bash
python G:/MicroPython_Skills/upy-autofix/scripts/triage.py \
  --log-dir {deploy_logs_path} \
  --port {COM} \
  --attempt 1
```

Outputs JSON to stdout, which the LLM captures and parses. The JSON structure is described in the header comments of `triage.py`.

**Every field has a default value** — the script uses try/except and will not crash on malformed log formats. The `warnings` field lists all degradation cases.

### Step 2: LLM Comprehensive Analysis

The LLM reads two information sources simultaneously:

| Source | Purpose | When to Read |
|--------|---------|--------------|
| triage.py JSON | Quick localization: error type, P-level, I2C status, attempt count | Always |
| deploy_logs/*.log raw logs | Deep understanding: full traceback, print timing, context | When JSON is insufficient |

**Analysis Order:**

1. Check JSON's `i2c_ok` field first
   - `false` → Hardware issue, jump to Step 5 (output troubleshooting guide), **do not fix code**
   - `true` or `null` (no I2C device) → Software issue, continue

2. Check JSON's `p_level` + `error_type`
   - P0 spelling/import → LLM directly Edit file (one-line fix, not worth starting upstream skill)
   - P0 driver API error → Step 3 delegate to upy-generate
   - P1 pin/address conflict → Step 3 delegate to upy-select-hw
   - P1 watchdog/memory → Step 3 delegate to upy-generate
   - P2 sensor anomaly → Step 3 delegate to upy-generate
   - P3 infinite loop/no output → Step 3 delegate to upy-generate
   - `unknown` → Read raw logs, LLM independently determines error type, then follows the corresponding path

3. Determine if hardware signal verification is needed (Step 2.5) — **triggered if any condition is met:**
   - `attempt >= 2` and the same `error_type` appears consecutively → Software fix ineffective, suspect hardware
   - `error_type` = `"NoOutput"` or `"unknown"` → Code ran but produced no output, suspect silent peripheral
   - `error_type` = `"OSError_19"` or `"OSError_110"` → Peripheral communication failure, possibly wiring/power

### Step 2.5: Hardware Signal Verification

When the trigger conditions in Step 2 are met, do not proceed directly to software fixes. Perform hardware diagnostics first.

#### Step 2.5A: LLM Reads Source Code & Generates Diagnostic Configuration

The LLM reads all `.py` source files in `firmware/` plus `project-manifest.json`:

```
Identify objects:
  ├── machine.Pin(x)     → GPIO pin, peripheral role (inferred from variable name/context)
  ├── machine.PWM(Pin(x)) → PWM output (LED/buzzer/servo)
  ├── machine.I2C(...)   → I2C bus + slave device address
  ├── machine.SPI(...)   → SPI bus + CS pin
  ├── machine.UART(...)  → UART bus + baud rate
  ├── machine.ADC(Pin(x)) → Analog input
  └── from xxx import Xxx → Driver class instantiation, __init__ parameter pattern
```

Then, following the template in **Appendix A**, generate `sanity_config.json` for each peripheral:

**Generation Principles:**
- Self-verifying types first: I2C/SPI sensors, ADC/DAC, UART (command-response type) use automatic determination
- Feedback type only for pure outputs: LED/Buzzer/Relay/Display/Motor require user questions
- Onboard LED tested first: If even the LED won't light → MCU power/reset issue
- Maximum 8 tests, single timeout 10s

```bash
python G:/MicroPython_Skills/upy-autofix/scripts/hardware_sanity.py \
  --config {project_dir}/sanity_config.json \
  --port {COM}
```

Outputs JSON to stdout, which the LLM captures.

#### Step 2.5B: LLM Interprets Results

```
Read JSON:
  ├── All PASS → Hardware is normal, problem is in code logic → Continue to Step 3, delegate fix
  │
  ├── Specific peripheral FAIL (I2C sensor scan failed / WHO_AM_I mismatch)
  │     → Output targeted troubleshooting guide for that peripheral (wiring/power/address conflict)
  │     → Do not continue fixing code
  │
  ├── Specific peripheral FAIL (user feedback type: LED not lit/buzzer not sounding)
  │     → Output targeted troubleshooting guide for that peripheral
  │     → Do not continue fixing code
  │
  ├── Onboard LED also FAIL → MCU power/reset/USB cable issue
  │     → Output basic troubleshooting guide (change USB cable/change power port/check EN pin)
  │
  └── pending_feedback == true
        → AskUserQuestion one by one (one question per item)
        → Re-evaluate after collecting answers
```

#### Step 2.5C: Handle User Feedback

`hardware_sanity.py` marks tests in `user_feedback` mode with `_pending_question` in the results. The LLM reads this:

```
For each pending item:
  AskUserQuestion(
    question: result._pending_question,
    header: "Hardware Diagnostics",
    options: ["Yes, it's normal", "No, no response"]
  )

After user answers:
  "Yes" → that peripheral status = "pass"
  "No" → that peripheral status = "fail"
```

**Re-evaluate after collecting all feedback**:
- All PASS → Continue with software fix
- Any FAIL → Output troubleshooting guide for that peripheral

**Maximum 3 user questions**. If there are 4+ feedback-type peripherals, prioritize testing the one "most likely to be faulty" (based on the triage error_type).

---

### Step 3: Delegate Fix to Upstream Skill

The LLM uses the `Skill` tool to call upstream skills, **packaging the error context**:

**When delegating to upy-generate, pass:**
- Original traceback (extracted from JSON or raw logs)
- Error file path + line number
- Involved driver name
- project-manifest.json path
- Previous modification content + failure reason (when attempt > 1)

**When delegating to upy-select-hw, pass:**
- Current pin conflict details
- project-manifest.json path

**When delegating to upy-analyze, pass:**
- Missing sensor/function description
- User's original requirement description

### Step 4: Verify Fix Result

Verification path after each fix:

```
Fix complete
  ↓
Optional: Skill("upy-simulate") PC-side quick verification (2-3s, avoids serial flash delay)
  ↓
Skill("upy-deploy") Re-flash and run
  ↓
Run triage.py again (--attempt N+1)
  ↓
Read JSON:
  ├─ status="pass" → Success, output PASS summary
  └─ status="fail" → Return to Step 2 for re-analysis (may escalate/fallback)
```

**Escalation Rule**: Consecutive failures with the same strategy → fallback to the next higher level (P0 direct edit → P0 delegate generate → P1 delegate select-hw → requirement-level analyze).

### Step 5: Hardware Issue — Output Troubleshooting Guide

When `i2c_ok: false` (software I2C + speed reduction already attempted, both ineffective), the LLM directly outputs the following Chinese guide, **do not fix code**:

```
I2C bus scan found no devices. Software I2C and low-speed mode have been attempted, both with no response. This is a hardware connection issue. Please troubleshoot in the following order:

1. Wiring Check:
   - Use a multimeter in continuity mode to confirm each wire (SDA/SCL/VCC/GND) is conductive
   - VCC must be connected to 3.3V (not 5V!)
   - GND must share a common ground with the MCU

2. Power Check:
   - Is the module's power indicator light on?
   - Is the VCC pin voltage 3.3V ± 0.1V?

3. Pull-up Resistors:
   - SDA/SCL each require a 4.7kΩ pull-up to 3.3V
   - Some modules have them built-in, some don't — check your module

4. Sensor Itself:
   - Is it abnormally hot?
   - Test with another sensor of the same model

5. Conflict Check:
   - Disconnect all other peripherals, test with only this sensor connected

After troubleshooting, send "redeploy" to retry.
```

### Step 6: All 3 Attempts Failed — Rollback + Summary

```bash
python G:/MicroPython_Skills/upy-autofix/scripts/triage.py --rollback --log-dir {deploy_logs_path}
```

Then the LLM outputs a Chinese bottleneck report:

```
Automatic fix failed 3 times.

Error Type: {error_type}
3 Attempts:
  1. {strategy1} → {result1}
  2. {strategy2} → {result2}
  3. {strategy3} → {result3}

Git has been rolled back to the pre-fix state.

Suggested manual troubleshooting direction: {specific suggestions}
```

### Step 7: Error Data Logging

`triage.py` automatically appends the history of each fix to `logs/error_report.json`, including: timestamp, MCU model, error type, traceback, strategy and result for each attempt, and the skill version used.

On the 3rd consecutive failure, the LLM additionally populates the `llm_analysis` field (root cause analysis + knowledge gap markers).

---

## Relationship with Other Skills

```
upy-deploy FAIL
    ↓
upy-autofix (this skill)
    ├── triage.py → Collect data
    ├── LLM analysis
    ├── [New] hardware_sanity.py → Hardware signal-driven verification
    ├── Delegate → upy-generate (code fix)
    ├── Delegate → upy-select-hw (pin/address fix)
    ├── Delegate → upy-analyze (requirement re-analysis)
    ├── Optional verification → upy-simulate (PC quick verification)
    ├── Re-deploy → upy-deploy
    └── Failure feedback → CI/CD feedback to each skill's SKILL.md
```

- ← `upy-deploy`: Receives FAIL judgment + deploy_logs/ logs
- ⇄ `upy-generate`: Delegates code fixes
- ⇄ `upy-select-hw`: Delegates pin/address reallocation
- ⇄ `upy-analyze`: Delegates requirement re-analysis
- ⇄ `upy-simulate`: Optional PC verification
- ⇄ `upy-deploy`: Re-flash after fix

---

## Hard Constraints

- **triage.py does not make fix decisions**: Only collects data and outputs JSON; the LLM reads JSON + raw logs and makes independent judgments
- **hardware_sanity.py does not make diagnostic decisions**: Only executes test code + collects results/user feedback; the LLM makes decisions based on the result JSON
- **Hardware check must be done first**: I2C scan empty → directly output troubleshooting guide, do not enter the fix loop
- **Hardware signal verification trigger conditions**: `attempt >= 2` with same consecutive error / `NoOutput` / `OSError_19/110` — do not waste user time
- **Self-verifying types take priority over feedback types**: Never ask the user if automatic determination is possible; a single sanity check can ask the user a maximum of 3 questions
- **Onboard LED tested first**: If even the LED won't light → MCU power/reset issue, do not test other peripherals
- **Peripheral clearly FAIL (self-test failed OR user answered NO)**: Immediately terminate the fix loop, output troubleshooting guide, do not continue fixing code
- **Git commit snapshot before each fix** (triage.py `--snapshot`)
- **Maximum 3 attempts**: All 3 failed → git rollback + output bottleneck report
- **LLM must read raw logs**: triage.py JSON may have `error_type: "unknown"`; in this case, the LLM must independently determine the error type from the raw logs
- **P0 spelling/import fixed directly by LLM Edit**: Not worth the overhead of starting an upstream skill
- **All other fixes must be delegated to upstream skills**: autofix does not write fix code itself
- **Error data feedback**: Each fix is recorded in `error_report.json`, driving continuous CI/CD improvement
- **Windows platform: Do not read device logs while main.py is running**: `mpremote fs cp` and `resume exec` enter raw REPL mode on Windows (sends Ctrl+C), which kills the running main.py. Furthermore, the killed process may not have flushed the log file, leading to reading an empty file and incorrectly judging "no log output". The correct approach is to let main.py run to its natural end or crash → then, after a soft reset (program stopped), use `fs cp` to fetch the logs. To check device status at runtime, use `hardware_sanity.py`'s I2C scan (independent of the main.py process)

---

## Appendix A: Peripheral Hardware Verification Code Templates

After the LLM reads the `firmware/` source code and identifies the peripheral type, it generates the test code in `sanity_config.json` according to the following templates.

**Template Structure:**
```json
{
  "id": "peripheral_name_sanity",
  "category": "i2c_sensor|spi_sensor|uart|gpio_out|display|adc|dac|input",
  "mode": "self_verify|user_feedback",
  "label": "Chinese name (pin/address)",
  "code": "Complete MicroPython code, executed via mpremote exec",
  "pass_pattern": "SCAN_OK|CHIP_OK|FRAME_OK|ADC_OK|DAC_OK|TEST_DONE",
  "fail_pattern": "SCAN_FAIL|CHIP_FAIL|FRAME_FAIL",
  "value_key": "TEMP|VOLT|RAW",
  "value_range": [min, max],
  "question": "Only for user_feedback mode: AskUserQuestion text",
  "timeout_ms": 10000
}
```

### A1: I2C Sensor (self_verify)

**Identification Signal**: `__init__(i2c, address=0x??)` or `I2C(0, scl=Pin(x), sda=Pin(y))`

**Self-test Steps**: i2c.scan() to confirm address → (optional) read WHO_AM_I/CHIP_ID → read data once → physically reasonable range check

**Generic Template**:
```python
from machine import I2C, Pin
i2c = I2C({bus_id}, scl=Pin({scl}), sda=Pin({sda}), freq=400000)
addrs = [hex(a) for a in i2c.scan()]
if '{expected_addr}' in addrs:
    print('SCAN_OK')
else:
    print('SCAN_FAIL:' + str(addrs))
# Read data once
from {driver_module} import {DriverClass}
s = {DriverClass}(i2c, address={addr_int})
try:
    r = s.{read_method}()
    print('VALUE:' + str(r))
except Exception as e:
    print('READ_ERR:' + str(e))
```

**Example — BMP280 (address 0x76)**:
```python
from machine import I2C, Pin
i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=400000)
addrs = [hex(a) for a in i2c.scan()]
if '0x76' in addrs:
    print('SCAN_OK')
else:
    print('SCAN_FAIL:' + str(addrs))
from bmp280_float import BMP280
s = BMP280(i2c, address=0x76)
data = s.read_compensated_data()
print('TEMP:' + str(data[0]) + ',' + 'PRESS:' + str(data[1]))
```
`pass_pattern: "SCAN_OK"`, `value_key: "TEMP"`, `value_range: [-40, 85]`

---

### A2: SPI Sensor (self_verify)

**Identification Signal**: `__init__(spi, cs)` or `SPI(1, sck=Pin(x), mosi=Pin(y), miso=Pin(z))`

**Self-test Steps**: Read WHO_AM_I / CHIP_ID / DEVICE_ID register → compare with datasheet expected value

**Generic Template**:
```python
from machine import SPI, Pin
spi = SPI({bus_id}, sck=Pin({sck}), mosi=Pin({mosi}), miso=Pin({miso}))
cs = Pin({cs}, Pin.OUT)
cs.value(1)
# Read WHO_AM_I register
cs.value(0)
spi.write(bytearray([0x80 | {who_am_i_reg}]))
resp = spi.read(1)
cs.value(1)
if resp[0] == {expected_id}:
    print('CHIP_OK:' + hex(resp[0]))
else:
    print('CHIP_FAIL: expected ' + hex({expected_id}) + ' got ' + hex(resp[0]))
```
`pass_pattern: "CHIP_OK"`, `fail_pattern: "CHIP_FAIL"`

**Example — ADXL345**:
```python
from machine import SPI, Pin
spi = SPI(1, sck=Pin(10), mosi=Pin(11), miso=Pin(12))
cs = Pin(9, Pin.OUT); cs.value(1)
cs.value(0)
spi.write(bytearray([0x80]))  # DEVID register
resp = spi.read(1)
cs.value(1)
if resp[0] == 0xE5:
    print('CHIP_OK')
else:
    print('CHIP_FAIL: got ' + hex(resp[0]))
```
`pass_pattern: "CHIP_OK"`, `fail_pattern: "CHIP_FAIL"`

---

### A3: UART Peripheral (self_verify)

**Identification Signal**: `__init__(uart)` or `UART(1, baudrate=9600, tx=Pin(x), rx=Pin(y))`

**Self-test Steps**: Send query command → wait for response → timeout determination

**Generic Template**:
```python
from machine import UART, Pin
import time
u = UART({uart_id}, baudrate={baud}, tx=Pin({tx}), rx=Pin({rx}), timeout=2000)
u.write(b'{query_cmd}')
time.sleep(0.5)
resp = u.read()
if resp and len(resp) > 0:
    print('FRAME_OK:' + str(resp[:32]))
else:
    print('FRAME_FAIL: no response')
```
`pass_pattern: "FRAME_OK"`, `fail_pattern: "FRAME_FAIL"`

**Example — PMS7003 (active mode, wait for data frame)**:
```python
from machine import UART, Pin
import time
u = UART(2, baudrate=9600, tx=Pin(17), rx=Pin(16), timeout=5000)
t0 = time.time()
while time.time() - t0 < 5:
    if u.any():
        d = u.read()
        if d and len(d) >= 2 and d[0] == 0x42 and d[1] == 0x4D:
            print('FRAME_OK: start bytes valid')
            break
else:
    print('FRAME_FAIL: no valid frame in 5s')
```
`pass_pattern: "FRAME_OK"`, `fail_pattern: "FRAME_FAIL"`

**Example — SIM800/SIM7600 (AT commands)**:
```python
from machine import UART, Pin
import time
u = UART(2, baudrate=115200, tx=Pin(17), rx=Pin(16), timeout=3000)
u.write(b'AT\r\n')
time.sleep(1)
resp = u.read()
if resp and b'OK' in resp:
    print('FRAME_OK: AT response')
else:
    print('FRAME_FAIL: ' + str(resp))
```

---

### A4: GPIO Output (user_feedback)

**Identification Signal**: `__init__(pin: int)` → Pin(x, Pin.OUT) or PWM(Pin(x))

**Test Steps**: PWM/level periodic change ×3 → ask user

**Generic Template**:
```python
from machine import Pin, PWM
import time
p = PWM(Pin({pin}), freq=1000)
for i in range(3):
    p.duty_u16(32768); time.sleep(0.3)
    p.duty_u16(0); time.sleep(0.3)
p.deinit()
print('TEST_DONE')
```
`pass_pattern: "TEST_DONE"`, `question: "{label} {action description}? (y/n)"`

**Variant — LED**:
```python
from machine import Pin, PWM
import time
p = PWM(Pin({pin}), freq=1000)
for _ in range(3):
    for d in range(0, 65535, 16384):  # fade in
        p.duty_u16(d); time.sleep(0.05)
    p.duty_u16(0); time.sleep(0.2)
p.deinit()
print('TEST_DONE')
```
`question: "Did the LED (pin {pin}) blink 3 times?"`

**Variant — Buzzer**:
```python
from machine import Pin, PWM
import time
p = PWM(Pin({pin}), freq=1000)
for f in [440, 660, 880]:  # A4, E5, A5
    p.freq(f); p.duty_u16(32768)
    time.sleep(0.25)
    p.duty_u16(0)
    time.sleep(0.1)
p.deinit()
print('TEST_DONE')
```
`question: "Did the buzzer sound 3 times?"`

**Variant — Relay**:
```python
from machine import Pin
import time
p = Pin({pin}, Pin.OUT)
for _ in range(3):
    p.value(1); time.sleep(0.3)
    p.value(0); time.sleep(0.3)
print('TEST_DONE')
```
`question: "Did you hear the relay click?"`

---

### A5: Display (user_feedback)

**Identification Signal**: `__init__(i2c/spi, ...)` + `.fill()` / `.show()` methods

**Test Steps**: Full-screen alternating fill → ask user

**Generic Template**:
```python
from machine import {bus_type}, Pin
{init_code}
# Full screen flash
display.fill(1); display.show()
import time; time.sleep(0.5)
display.fill(0); display.show()
time.sleep(0.5)
display.fill(1); display.show()
print('TEST_DONE')
```
`question: "Did the {label} screen flash?"`

**Example — SSD1306 (I2C)**:
```python
from machine import I2C, Pin
from ssd1306 import SSD1306
i2c = I2C(0, scl=Pin(22), sda=Pin(21))
display = SSD1306(128, 64, i2c)
display.fill(1); display.show()
import time; time.sleep(0.5)
display.fill(0); display.show()
time.sleep(0.5)
display.fill(1); display.show()
print('TEST_DONE')
```

**Example — ST7789 (SPI, Color)**:
```python
from machine import SPI, Pin
from st7789 import ST7789
spi = SPI(1, sck=Pin(10), mosi=Pin(11), miso=Pin(12))
dc = Pin(8, Pin.OUT); cs = Pin(9, Pin.OUT); rst = Pin(7, Pin.OUT)
display = ST7789(spi, 240, 240, dc=dc, cs=cs, rst=rst)
import time
for color in [0xF800, 0x07E0, 0x001F]:  # Red→Green→Blue
    display.fill(color); time.sleep(0.5)
print('TEST_DONE')
```
`question: "Did the screen display Red→Green→Blue?"`

---

### A6: ADC (self_verify)

**Identification Signal**: `__init__(i2c, address=0x??)` + `read()` with channel/gain parameters

**Self-test Steps**: Read floating channel vs read VCC or fixed voltage → difference determination

**Generic Template**:
```python
from machine import I2C, Pin
i2c = I2C({bus_id}, scl=Pin({scl}), sda=Pin({sda}))
from {driver_module} import {DriverClass}
adc = {DriverClass}(i2c, address={addr})
import time
r1 = adc.read(channel1=0)
time.sleep(0.1)
r2 = adc.read(channel1=0)
diff = abs(r1 - r2)
if diff < 50:
    # Reading too stable might be floating noise or short circuit
    # Try reading another channel
    r3 = adc.read(channel1=3)
    if abs(r1 - r3) > 100:
        print('ADC_OK: ch0=' + str(r1) + ', ch3=' + str(r3))
    else:
        print('ADC_FAIL: all channels same ~' + str(r1))
else:
    print('ADC_OK: fluctuation ' + str(diff))
```
`pass_pattern: "ADC_OK"`, `fail_pattern: "ADC_FAIL"`

---

### A7: DAC (self_verify)

**Identification Signal**: `__init__(i2c, address=0x??)` + `write()` + `read()` (with readback)

**Self-test Steps**: write(mid-value) → read() → compare

**Generic Template**:
```python
from machine import I2C, Pin
i2c = I2C({bus_id}, scl=Pin({scl}), sda=Pin({sda}))
from {driver_module} import {DriverClass}
dac = {DriverClass}(i2c, address={addr})
dac.write(2048)
import time; time.sleep(0.05)
state = dac.read()
if state and len(state) >= 2:
    val = (state[0] << 4) | (state[1] >> 4)
    if abs(val - 2048) < 100:
        print('DAC_OK: wrote=2048, read=' + str(val))
    else:
        print('DAC_FAIL: wrote=2048, read=' + str(val))
else:
    print('DAC_FAIL: no response')
```
`pass_pattern: "DAC_OK"`, `fail_pattern: "DAC_FAIL"`

**Variant — DAC without read()** (e.g., some MCP4725 implementations):
Do not test readback; instead, do `write(0) → write(2048) → write(0)` and measure voltage with a multimeter. In this case, downgrade to `user_feedback` mode. `question: "Does the voltage on the DAC output pin change between write(0) and write(2048)?"`

---

### A8: Input Device (user_feedback, semi-self-test)

**Identification Signal**: `__init__(pin, ...)` + includes callback/idle_state/debounce

**Test Steps**: Read initial value → print → wait → read final value

**Generic Template**:
```python
from machine import Pin
import time
p = Pin({pin}, Pin.IN, Pin.PULL_UP)
print('INIT_VAL:' + str(p.value()))
time.sleep(6)  # Give user time to act
print('FINAL_VAL:' + str(p.value()))
```
`question: "Please {action description} within 6 seconds (press button/rotate encoder/move in front of PIR). Did the INIT_VAL and FINAL_VAL values printed in the REPL change?"`

**Example — Button**:
```python
from machine import Pin
import time
p = Pin(5, Pin.IN, Pin.PULL_UP)
print('INIT:' + str(p.value()))
time.sleep(6)
print('FINAL:' + str(p.value()))
print('TEST_DONE')
```
`question: "Please press the button within 6 seconds. Did the INIT and FINAL values in the REPL change?"`

---

### A9: Onboard LED Basic Test (self_verify downgraded to headless)

**When to use**: The first step of any sanity check. If this fails, do not test other peripherals.

```python
from machine import Pin
import time
led = Pin({pin}, Pin.OUT)
for _ in range(4):
    led.value(1); time.sleep(0.2)
    led.value(0); time.sleep(0.2)
print('LED_OK')
```
`pass_pattern: "LED_OK"`, `question: "Did the onboard LED blink?"`

Note: Although the onboard LED is a GPIO output, it is the most basic indicator of MCU power/reset/flash success. If `LED_OK` is not printed → the device did not enter REPL.

---

### Decision Tree for Generating sanity_config.json

```
LLM reads firmware/ source code:
    │
    ├── Found Pin(x, Pin.OUT) / PWM(Pin(x)) and variable name contains "led"
    │     → A9: Onboard LED basic test (highest priority, place at config.tests[0])
    │
    ├── Found I2C(0, scl=Pin(x), sda=Pin(y))
    │     └── Iterate over all I2C driver class instances
    │           → A1: I2C sensor (self_verify)
    │
    ├── Found SPI(1, sck=Pin(x), ...)
    │     └── Iterate over all SPI driver class instances
    │           → A2: SPI sensor (self_verify)
    │
    ├── Found UART(1, baudrate=..., tx=Pin(x), rx=Pin(y))
    │     └── Iterate over all UART driver class instances
    │           → A3: UART peripheral (self_verify or user_feedback)
    │
    ├── Found Pin(x, Pin.OUT) / PWM(Pin(x)) and variable name contains "buzzer"/"relay"/"motor"
    │     → A4: GPIO output (user_feedback)
    │
    ├── Found I2C/SPI driver with .fill() / .show() methods
    │     → A5: Display (user_feedback)
    │
    ├── Found I2C driver with .read() + channel/gain parameters
    │     → A6: ADC (self_verify)
    │
    ├── Found I2C driver with .write() + .read()
    │     → A7: DAC (self_verify)
    │
    └── Found Pin(x, Pin.IN) with callback/idle_state
          → A8: Input device (semi_auto)
```

**Test Order**: Onboard LED → I2C sensor → SPI sensor → UART → ADC/DAC → Display → GPIO output → Input device. If the previous test FAILs and is a basic level (LED / power related), do not run the subsequent tests.
