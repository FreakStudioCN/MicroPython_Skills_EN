---
name: upy-gen-driver
description: Generate MicroPython drivers from PDF datasheets or Arduino/C++ code. Use when no driver is found on upypi or GitHub. Flow: extract → generate debug version → hardware verification loop → strip debug → normalize. Trigger: called by upy-analyze when no driver is found, or directly by user via /upy-gen-driver.
---

# MicroPython Driver Generation Skill

## Role

Generate standardized MicroPython drivers from non-MicroPython sources (PDF datasheets, Arduino/C++ code, chip model numbers). **Independent skill**, callable by `upy-analyze`, `upy-autofix`, or directly by the user.

Core flow: **Extract → Generate Debug Version → Hardware Verification → Strip Debug → Normalize**.

**Hard execution order constraint: Step 4 must not be executed before Step 3 is complete; Step 5 must not be executed before Step 3 is complete. Check prerequisites before entering each step.**

---

## Prerequisites

- One of the following: PDF datasheet / Arduino/C++ source code / Arduino GitHub repository URL / chip model number
- `mpremote` available (required for hardware verification phase)
- Python 3 environment + `pymupdf` (for PDF extraction)

---

## Execution Steps

### Step 0: Determine Input Type

Based on the material provided by the user, decide which path to take:

```
├─ PDF datasheet (.pdf)          → Step 1A
├─ Arduino/C++ code (.ino/.cpp)  → Step 1B
├─ GitHub Arduino repository URL → git clone → Step 1B
└─ Chip model number only        → WebSearch datasheet → Download PDF → Step 1A
```

---

### Step 1A: PDF Path — Text Extraction

```bash
python G:/MicroPython_Skills/upy-gen-driver/scripts/extract_pdf.py \
  --input {datasheet.pdf} \
  --output {chip}_text.json
```

Output JSON structure:
```json
{
  "source": "datasheet.pdf",
  "pages": [
    {"num": 1, "text": "Full page text..."},
    {"num": 2, "text": "Full page text..."}
  ]
}
```

The script only performs plain text extraction (pymupdf), without any interpretation. Page numbers are preserved for reference.

**LLM reads the extracted text, understands it, and explicitly outputs the following checklist (writes to `{chip}_understanding.json`):**
- Communication protocol (I2C address / SPI mode / UART baud rate)
- **Chip identification method**: Is there an ID/WHO_AM_I/CHIP_ID register? (Yes → address + expected value; No → mark N/A, use register write-read-back as a substitute)
- **Data ready notification method**: Status register polling / Hardware pin interrupt / Fixed delay / None (specify the exact register bit or pin)
- Register map (address + bit definition + read/write permissions), **mark which registers are write-only (read-back will fail)**
- Initialization sequence (steps + timing + delay requirements)
- Data format (big-endian/little-endian, unsigned/two's complement, conversion formula)
- AT command format (send format + response format + timeout)
- **State variable mapping**: Which hardware configurations need shadow tracking (e.g., `_gain` → GAIN bit, `_vref` → VREF bit, `_mode` → CM bit), the independent boundary of each setter (who manages what, no cross-boundary interference)
- **Waiting strategy**: Does the chip have a ready flag/status bit? → Poll that flag + timeout; No ready flag? → Fixed delay (datasheet conversion time + margin)
- **Data integrity**: Does the chip communication have CRC/checksum? → Must verify when reading data; No? → Mark N/A

---

### Step 1B: Arduino Path — API Mapping + Source Code Analysis

```bash
python G:/MicroPython_Skills/upy-gen-driver/scripts/convert_arduino.py \
  --input {source.ino} \
  --output {chip}_mapping.json
```

Output JSON structure:
```json
{
  "source": "source.ino",
  "includes": ["Wire.h", "SPI.h"],
  "global_vars": [{"name": "sensor_addr", "value": "0x44"}],
  "functions": [
    {"name": "readSensor", "return_type": "float", "params": [], "line": 42}
  ],
  "api_mapping": [
    {"arduino": "Wire.beginTransmission(0x44)", "mpy": "i2c.writeto(0x44, buf)", "line": 45}
  ],
  "has_setup_loop": true,
  "logic_summary": "Initializes Wire in setup(), reads sensor data every 2 seconds in loop() and prints via Serial"
}
```

The script performs API mapping table lookup + code structure extraction, **it does not translate code**.

**LLM reads both:**
1. The original Arduino source code (to understand the logic intent, control flow, and error handling)
2. The mapping JSON (to assist in locating API correspondences)

**Translation principles:**
- Do not mechanically translate line-by-line — understand the original code logic, then rewrite using idiomatic MicroPython
- Polling in Arduino `loop()` → replace with callback or timer in MPY
- Arduino `delay()` → use `time.sleep_ms()` or asynchronous methods in MPY
- Arduino `Serial.print()` → use `print()` or logging in MPY

---

### Step 2: LLM Generates "Debug Version" Single-File Driver

Output file: `firmware/drivers/{chip}_driver/{chip}_debug.py`

**Before generation, determine the following branches based on `{chip}_understanding.json`, then apply the corresponding template:**

```
Communication protocol?
├─ I2C  → Self-test includes i2c.scan() + address verification
├─ SPI  → Self-test includes CS pin toggling + read-back test
└─ UART → Self-test includes AT command round-trip verification

Chip identification?
├─ Has ID register → Read and compare with expected value
└─ No ID register → Use register write-read-back as substitute (write known value → read back → assert)

Data ready?
├─ Status register polling → while not (read_status() & MASK): sleep_ms(N), add timeout limit
├─ Hardware pin interrupt → Wait for pin.value() == 0, add timeout limit
└─ Fixed delay     → time.sleep_ms(conversion_time + margin)

Data integrity?
├─ Has CRC/checksum → Verify integrity after reading data, raise RuntimeError on checksum failure
└─ No CRC/checksum → Skip
```

**The debug version must include the following self-test steps (choose based on actual chip situation, skip if not applicable):**

```python
# === File header: chip info + data source ===
print("=" * 50)
print("Driver: {chip} ({protocol}: {detail})")
print("Source: {datasheet.pdf Page X / Arduino code}")
print("=" * 50)

# === [Connection Verification] Choose based on protocol ===
# I2C: Scan bus
print("[INIT] I2C scan...")
i2c_devices = i2c.scan()
print("  Found: %s" % [hex(a) for a in i2c_devices])
if 0xXX not in i2c_devices:
    print("  [FAIL] Device 0x%02X not found!" % 0xXX)
    print("  [HINT] Check wiring / power / pull-up resistors")

# SPI: Read known register (e.g., WHO_AM_I or default value of a config register)
# UART: Send AT and check response

# === [Initialization] Reset + default value verification ===
print("[INIT] Reset device...")
reset()  # or send RESET command / pull RESET pin low
time.sleep_ms(N)  # Post-reset wait time specified in datasheet
# Read default config register, verify it matches datasheet default value

# === [Identity Verification] Verify if ID register exists, skip if not ===
# If ID register exists:
print("[INIT] Read ID register (0x%02X)..." % ID_REG)
val = read_reg(ID_REG)
print("  Value: 0x%02X (expected: 0x%02X)" % (val, EXPECTED_ID))
if val != EXPECTED_ID:
    print("  [FAIL] ID mismatch! Got 0x%02X, expected 0x%02X" % (val, EXPECTED_ID))
    print("  [HINT] Check protocol config / wiring / datasheet Page X")

# If no ID register: use register write-read-back as substitute
print("[INIT] Communication sanity check (write → read-back)...")
test_patterns = [0x00, 0x55, 0xAA]  # Choose registers that can be safely written
for pat in test_patterns:
    write_reg(CONFIG_REG, pat)
    rb = read_reg(CONFIG_REG)
    if rb != pat:
        print("  [FAIL] Wrote 0x%02X, read-back 0x%02X" % (pat, rb))
    else:
        print("  [OK] Write 0x%02X → read-back 0x%02X" % (pat, rb))

# === [Initialization Sequence] Write registers step by step, read-back each item ===
print("[INIT] Configuration sequence...")
init_seq = [(REG_A, VAL_A, "Description A"), (REG_B, VAL_B, "Description B"), ...]
for reg, val, desc in init_seq:
    write_reg(reg, val)
    rb = read_reg(reg)
    if rb != val:
        print("  [FAIL] %s: reg 0x%02X wrote 0x%02X, read-back 0x%02X" % (desc, reg, val, rb))
        # Write-only register: mark "write-only, skipping read-back"
    else:
        print("  [OK] %s (reg 0x%02X = 0x%02X)" % (desc, reg, val))

# === [Functional Test] Read data once / Send command once ===
print("[TEST] Functional test...")
try:
    data = read_sensor()
    print("  Reading: %s" % str(data))
except Exception as e:
    print("  [FAIL] %s" % e)
    import sys
    sys.print_exception(e)

# === Final Verdict ===
print("=" * 50)
print("SELF_TEST_PASS")   # or print("SELF_TEST_FAIL: <reason>")
```

**Key requirements:**
- All print strings in English
- On failure, print specific expected value vs actual value
- On failure, print troubleshooting hints (which register/page/wire to check)
- Single file, no package splitting, easy to modify repeatedly
- **Write-only registers marked "write-only", no read-back verification**
- **`__init__` must put the chip into a known state** (call reset or read current configuration to confirm)
- **`__init__` top-level parameter validation**: Check bus type (I2C/SPI/UART), address range (0x00-0x7F for I2C), parameter validity, immediately `raise TypeError` / `ValueError` on failure
- **If the class provides convenience methods that depend on configuration (e.g., voltage conversion depends on gain/VREF), use instance variables like `_gain`/`_vref` to track the current value, and each setter only modifies its own responsible state, prohibiting cross-setter contamination**
- **All polling must have a timeout**: No infinite `while True` polling. Use `ticks_ms()`/`ticks_diff()` or `for _ in range(max_iterations)` to bound. On timeout, `raise RuntimeError` with troubleshooting hints
- **Communication exception translation**: Wrap I2C/SPI/UART operations in `try`/`except OSError`, convert to `RuntimeError` with descriptive message (device address/register/expected operation)
- **Pre-allocate bytearray**: For repeated I2C/SPI read/write operations, use pre-allocated `buf = bytearray(N)` to avoid MicroPython heap fragmentation
- **Prefer polling ready bit**: If the chip has a status register/ready flag, use polling+timeout to wait; only use fixed delay when the chip has no ready flag at all

---

### Step 3: Hardware Verification Loop

**Loop limit: 10 rounds.**

**This step cannot be skipped. If no MicroPython device is available in the current environment, you must pause and ask the user, and are prohibited from directly entering Step 4.**

#### 3.0 Device Pre-check (Must be executed first every time Step 3 is entered)

```bash
mpremote devs
```

| Output | Action |
|------|----------|
| COM port list present | Record COM port, proceed to 3A |
| No output (no device) | **Pause**, output `[HALT] No MicroPython device detected. Please connect device and tell me the COM port, or type "skip" to skip hardware verification.` **Do not proceed before user confirmation.** |

If the user explicitly types "skip" to skip hardware verification, jump directly to Step 4, and mark "⚠️ Not hardware verified" in the final output.

#### 3A. Flash and Run

```bash
mpremote connect {COM} resume run firmware/drivers/{chip}_driver/{chip}_debug.py
```

Use `resume run` instead of `fs cp` — send execution via REPL, no flash write, sub-second feedback.

#### 3B. LLM Analyze Output

| Output | Judgment | Action |
|------|----------|------|
| `SELF_TEST_PASS` | All self-tests passed | Exit loop, proceed to Step 4 |
| `ID/known-value mismatch` | Chip identification failed or communication error | Check datasheet to confirm ID register address/expected value; if chip has no ID register, check if register write-read-back test used a write-only register |
| `read-back mismatch` | Timing issue or register is write-only | Add delay / Check datasheet to confirm register read/write permissions |
| AT response format mismatch | Command/baud rate/parsing error | Adjust command format / Try different baud rate |
| Initialization stuck | A step timed out or ready bit not set | Add timeout / Replace fixed delay with polling / Check if data ready method is correct |
| Device crash no response | Code caused crash | `mpremote connect {COM} soft-reset` → Next round |
| Bus scan empty / Device no response | Hardware connection issue or protocol configuration error | Output troubleshooting guide (wiring/power/pull-up/CS pin/baud rate), pause loop |

#### 3C. LLM Directly Edit File → Return to 3A

---

### Step 4: Strip Debug → Production Driver

**Prerequisites (all must be met, otherwise Step 4 is prohibited):**

| # | Condition | Verification Method |
|---|----------|-------------------|
| 1 | Step 3 hardware verification loop has been executed | The most recent `mpremote resume run` has output |
| 2 | The last run output `SELF_TEST_PASS` | Output contains `SELF_TEST_PASS` |
| 3 | Debug version file exists | `firmware/drivers/{chip}_driver/{chip}_debug.py` exists |

**If any condition is not met → Return to Step 3 to complete hardware verification, do not skip.**

After hardware verification passes:

1. Remove all line-by-line debug prints (`[INIT] [1/5]...`, `[TX]/[RX]`, etc.)
2. Keep SELF_TEST logic but change to `_self_test()` private method (not called by default)
3. Keep critical error information (exception messages, key register verification failures)
4. Keep connection verification methods based on protocol (I2C: `scan()` public method; SPI: read known register; UART: AT probe)
5. Organize by standard structure: Class constants → `__init__` → Public methods → Private methods → `deinit()`
6. Follow dependency injection (I2C/SPI/UART instances passed in from outside)
7. **`__init__` must put the chip into a known state** (call hardware reset / send RESET command / read and confirm default register values)
8. **Internal state consistency**: If the class provides convenience methods that depend on chip configuration (e.g., voltage conversion depends on gain/VREF), use instance variables like `_gain`, `_vref` to track the current value. Each setter only modifies its own responsible tracking variable, prohibiting cross-setter contamination. For example, `set_gain()` must not modify `_vref`; `set_vref(VREF_EXTERNAL)` should prompt the user to set `_vref` themselves or provide a parameter to pass the external reference voltage value
9. **`deinit()` method**: If the chip datasheet supports low-power/sleep/standby modes, implement `deinit()` to send POWERDOWN/STANDBY command and release resources
10. **Optional `__del__`**: Can add `__del__` to automatically call `deinit()` during GC, for low-power scenarios

Output: `firmware/drivers/{chip}_driver/{chip}.py`

---

### Step 5: Normalization

```bash
Skill("upy-norm-driver")
```

Pass in `{chip}.py`, execute all 38 P0 rule checks and fixes.

Output the normalized driver. **Driver ready.**

---

## Relationship with Other Skills

```
upy-analyze (upypi + GitHub no results)
    ↓
upy-gen-driver (this skill)
    ├── scripts/extract_pdf.py       ← PDF text extraction
    ├── scripts/convert_arduino.py   ← Arduino API mapping
    ├── mpremote resume run          ← Hardware verification (mpremote-device-interaction)
    └── Skill("upy-norm-driver")     ← Normalization
    ↓
Output: firmware/drivers/{chip}_driver/{chip}.py
    ↓
Available for upy-generate (Phase 4)
```

- ← `upy-analyze`: Calls this skill when no driver is found
- ← `upy-autofix`: Calls this skill when diagnosis indicates missing driver
- → `upy-norm-driver`: Normalizes the generated driver
- → `upy-generate`: Uses the generated driver to continue the main flow

---

## Hard Constraints

- **extract_pdf.py only does text extraction**: Does not parse register tables, does not determine protocol type; all understanding is done by the LLM
- **convert_arduino.py only does mapping + structure extraction**: Does not translate code; translation is done by the LLM after understanding the original logic
- **Debug version driver must perform full self-test**: Every register/command/read operation has an expected value comparison; on failure, print expected vs actual
- **Hardware verification must come first (hard constraint, violation is a process error)**: Prohibited from executing Step 4/5 before Step 3 is complete and outputs `SELF_TEST_PASS`. If no device is available in the current environment, must pause and wait for user confirmation; must not skip Step 3 with the excuse "no device currently available". Step 4 has a prerequisite checkpoint; must self-check every time before entering Step 4.
- **Verification uses `mpremote resume run`**: Does not modify flash, fast iteration
- **Arduino translation cannot be mechanical**: Must understand the original code logic, then rewrite using idiomatic MPY
- **Dependency injection is a hard requirement**: I2C/SPI/UART instances must be passed in from outside, not created inside the class
- **Datasheet page number references preserved**: Annotate data source in comments `(Datasheet Page X, Table Y)`
- **Maximum 10 verification rounds**: Abandon if exceeded, output troubleshooting summary
- **print/raise strings in English**
- **No infinite polling**: All wait/polling loops must have a timeout (use `ticks_diff()` or `for _ in range(N)`), `raise RuntimeError` with troubleshooting hints on timeout
- **No cross-setter contamination of shadow state**: `set_gain()` must not modify `_vref`, `set_vref()` must not modify `_gain`. The boundary of each setter is explicitly defined in `{chip}_understanding.json`
