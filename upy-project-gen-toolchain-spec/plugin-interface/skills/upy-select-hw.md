# upy-select-hw Interface Definition

> Status: ✅ Finalized
>
> Phase 2 — MCU Selection + Firmware Verification + Pin Assignment + BOM Generation. Input: project-manifest.json, Output: complete hardware solution.

---

## I. Skill Overview

| Item | Content |
|------|---------|
| Phase | select-hw |
| Upstream Skill | upy-analyze (auto-entry) or incremental trigger from any phase (user adds device) |
| Downstream Skill | upy-scaffold |
| One-line Responsibility | Match best MCU from boards/ database → verify firmware → assign pins → calculate BOM cost |

**Core Constraint:** Does not write code, search for drivers, or generate files. Only outputs a hardware solution JSON.

**Two Operating Modes:**

| Mode | Trigger | Behavior |
|------|---------|----------|
| `full` | upy-analyze completes | New selection + full pin assignment |
| `incremental` | User adds device in a subsequent phase | Assign pins only for new devices, leave existing pins untouched |

---

## II. Plugin Input → Skill (P→S)

The plugin sends **1 message** to start this skill:

```json
{
  "type": "start_phase",
  "phase": "select-hw",
  "session_id": "uuid-xxx",
  "payload": {
    "mode": "full",
    "manifest": { "...complete project-manifest.json..." },
    "pre_selected_board": {
      "id": "esp32-devkit-v1",
      "display_name": "ESP32 DevKit V1",
      "mcu": "ESP32-WROOM-32",
      "chip_family": "esp32",
      "firmware_url": "https://micropython.org/download/ESP32_GENERIC/"
    },
    "previous_pinout": [],
    "new_devices": []
  }
}
```

| Field | Type | Required | Source | Description |
|-------|------|----------|--------|-------------|
| `mode` | string | Yes | Server determines | `"full"` full selection / `"incremental"` incremental assignment |
| `manifest` | object | Yes | Upstream phase_complete | Complete content of project-manifest.json |
| `pre_selected_board` | object? | No | Plugin board selector | Has value if user pre-selected a board, null if LLM should recommend |
| `previous_pinout` | array | Required for incremental | Current manifest pinout | Existing pin assignments, not modified in incremental mode |
| `new_devices` | array | Required for incremental | List of devices added by user | Assign pins only for these devices |

**mode determination logic (server-side):**
- `manifest.phase === "analyze"` and first entry → `full`
- User clicks "Add Device" in a subsequent phase → starts a mini pipeline, select-hw receives `incremental`

**pre_selected_board behavior:**
- Has value → Skip MCU selection, skip firmware verification (firmware_url already determined), go directly to pin assignment
- null → Step 1 executes MCU recommendation + firmware verification

---

## III. Skill Output → Plugin (S→P)

### Message Sequence

```
full mode:
  Step 1 MCU Selection
    → status_update "Loading board database..."
    → status_update "Matching best MCU..."
    → approval_request #1: MCU recommendation card (only triggered when pre_selected_board=null)
  
  Step 2 Pin Assignment
    → status_update "Reading pin constraints from boards/{id}.json..."
    → status_update "Assigning pins... (1/N)"
    → script_run: pin-validator.py validates pin scheme
  
  Step 3 BOM
    → status_update "Generating bill of materials..."
  
  Step 4 Output
    → phase_complete: Result panel

incremental mode:
    → status_update "Assigning pins for new devices..."
    → script_run: pin-validator.py incremental validation
    → phase_complete (skips MCU selection and BOM recalculation)
```

### Message Details

#### approval_request #1 — MCU Recommendation Card (Conditional)

**Trigger Condition:** `pre_selected_board` is null and mode=`full`.

**Not Triggered:** User pre-selected a board → skip this card, go directly to pin assignment.

```
┌──────────────────────────────────────────┐
│  Hardware Selection Recommendation        │
│                                          │
│  Your Requirements: Temp/Humidity Monitor + Display + Buzzer Alarm │
│                                          │
│  ★ Recommended: ESP32 DevKit V1          │
│    Reason: WiFi+BLE, Sufficient GPIO(26), Largest Ecosystem │
│    Price: ~¥25                           │
│    Firmware: ESP32_GENERIC (v1.24.1)     │
│                                          │
│  Alternatives:                           │
│  ┌────────────────────────────────────┐  │
│  │ ○ Raspberry Pi Pico W              │  │
│  │   WiFi, USB Drag-and-Drop Flashing, ¥15 │
│  ├────────────────────────────────────┤  │
│  │ ○ ESP32-S3-DevKitC-1               │  │
│  │   WiFi+BLE+AI, PSRAM, ¥35          │  │
│  └────────────────────────────────────┘  │
│                                          │
│  [Use Recommended ESP32] [Select Pico W] [Select S3] │
└──────────────────────────────────────────┘
```

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "mcu_select",
    "header": "Hardware Selection Recommendation",
    "question": "Based on your requirements, we recommend the following MCU solutions",
    "summary": {
      "project_name": "Temp/Humidity Monitor Alarm",
      "description": "Temp/Humidity monitoring + OLED display + Buzzer alarm, 3 devices total"
    },
    "items": [
      {
        "id": "board_1",
        "name": "ESP32 DevKit V1",
        "subtitle": "WiFi+BLE | GPIO×26 | I2C×2 | SPI×2 | ¥25",
        "meta": "★ Recommended",
        "selected": true
      },
      {
        "id": "board_2",
        "name": "Raspberry Pi Pico W",
        "subtitle": "WiFi | GPIO×26 | I2C×2 | SPI×2 | ¥15",
        "meta": "Alternative",
        "selected": false
      },
      {
        "id": "board_3",
        "name": "ESP32-S3-DevKitC-1",
        "subtitle": "WiFi+BLE+AI | GPIO×45 | I2C×2 | SPI×3 | ¥35",
        "meta": "Alternative",
        "selected": false
      }
    ],
    "allow_add": false,
    "allow_remove": false,
    "multi_select": false,
    "actions": [
      { "label": "Use Recommended ESP32", "value": "confirm", "primary": true },
      { "label": "Select Pico W", "value": "select_board_2" },
      { "label": "Select S3", "value": "select_board_3" }
    ]
  }
}
```

**MCU Recommendation Algorithm (Executed by Server-Side LLM):**

1. Load all `boards/*.json` (except _template.json and matching-rules.json)
2. Load `boards/matching-rules.json`, trigger rules based on manifest.requirements
3. For each board: boost rule matches chip_family → +1 point; exclude rule matches → exclude
4. `beginner_friendly=true` adds an extra +1 in mode=beginner
5. Top 1 score is the recommendation, Top 2~3 are alternatives
6. Excluded boards are not necessarily bad boards, they may just be unsuitable for the current scenario

#### status_update List

| step_id | message | level | Trigger Timing |
|---------|---------|-------|----------------|
| load_boards | Loading board database... | info | full mode Step 1 start |
| match_mcu | Matching best MCU (considering WiFi/GPIO/I2C/Budget...) | info | full mode MCU scoring |
| mcu_done | ✓ Recommended ESP32 DevKit V1 (WiFi+BLE, GPIO×26, ¥25) | success | full mode MCU selected |
| mcu_skipped | ✓ MCU selected by user: ESP32 DevKit V1 | success | When pre_selected_board has value |
| load_pin_layout | Reading pin constraints from boards/{id}.json... | info | Step 2 start |
| assign_pins | Assigning pins... (1/3) | info | Each device assignment |
| pin_assigned | ✓ SHT30 → I2C0 (SDA=21, SCL=22, addr=0x44) | success | Single device assignment complete |
| pin_conflict | ⚠ SHT30 and SSD1306 share I2C0 bus, addresses do not conflict ✓ | warn | I2C shared bus notification |
| validate_pins | Validating pin scheme... | info | Before pin-validator.py runs |
| validate_ok | ✓ Pin scheme validation passed (no conflicts, no restricted pins) | success | Validation passed |
| validate_fail | ✗ Pin validation failed: GPIO12 occupied by both strapping and LED | error | Validation failed (triggers reassignment) |
| bom_gen | Generating bill of materials... | info | Step 3 |
| bom_done | ✓ BOM has 6 items, estimated ¥52 (within budget) | success | BOM complete |
| incremental_start | Assigning pins for new device DHT22... | info | incremental mode |
| incremental_done | ✓ DHT22 → GPIO14 (free), did not affect existing 6 pins | success | incremental complete |

#### script_run — pin-validator.py

**This is a new validation script.** Path: `G:\MicroPython_Skills\upy-select-hw\scripts\pin-validator.py`

```json
{
  "type": "script_run",
  "payload": {
    "script_id": "pv_001",
    "interpreter": "python",
    "script": "pin-validator.py",
    "args": ["--board", "boards/esp32-devkit-v1.json", "--pinout", "{pinout_json_stdin}"],
    "cwd": "{skill_dir}",
    "timeout_ms": 5000
  }
}
```

**Script Responsibilities (Deterministic Validation, LLM Cannot Do):**

| Validation Item | Description |
|-----------------|-------------|
| GPIO Overlap Detection | Same GPIO cannot be assigned to two non-shared devices |
| restricted_gpio Violation | Assigned GPIO cannot appear in any category of restricted_gpio (strapping/flash/input_only etc., unless LLM explicitly states a reason) |
| I2C Address Conflict | Two devices on the same I2C bus cannot have the same address |
| Bus Count Exceeded | Number of assigned I2C/SPI/UART cannot exceed hardware count in specs (if exceeded, must mark as SoftI2C/SoftSPI) |
| fixed Model Compliance | For RP2040 etc., pins must be in the pin_options selectable list, and SDA/SCL must be paired |
| onboard_occupied Conflict | Assigned GPIO cannot conflict with pins where onboard_peripherals has always_used=true |

**On Validation Failure:** Script returns non-zero exit code + error details JSON. LLM reads the error and reassigns, up to 3 retries.

#### phase_complete

```json
{
  "type": "phase_complete",
  "payload": {
    "phase": "select-hw",
    "result": "success",
    "summary": "Hardware solution determined: ESP32 DevKit V1, 6/26 GPIO assigned, BOM estimated ¥52",
    "next_phase": "scaffold",
    "artifacts": [
      {
        "type": "table",
        "title": "Pin Assignment Table",
        "headers": ["Device", "Pin Function", "GPIO", "Physical Pin", "Type", "Notes"],
        "rows": [
          ["SHT30", "I2C SDA", "21", "—", "i2c_data", "Shared I2C0"],
          ["SHT30", "I2C SCL", "22", "—", "i2c_clock", "Shared I2C0"],
          ["SSD1306", "I2C SDA", "21", "—", "i2c_data", "Shared I2C0 (0x3C)"],
          ["SSD1306", "I2C SCL", "22", "—", "i2c_clock", "Shared I2C0 (0x3C)"],
          ["Buzzer", "GPIO OUT", "4", "—", "gpio_out", "PWM driven"],
          ["Power", "3V3", "3V3", "—", "power_3v3", "For I2C devices"],
          ["Power", "GND", "GND", "—", "gnd", ""]
        ]
      },
      {
        "type": "table",
        "title": "Bill of Materials (BOM)",
        "headers": ["#", "Name", "Model", "Qty", "Unit Price", "Notes"],
        "rows": [
          ["1", "MCU", "ESP32 DevKit V1", "1", "¥25", "Includes USB cable"],
          ["2", "Temp/Humidity Sensor", "SHT30", "1", "¥8", "I2C"],
          ["3", "OLED Display", "SSD1306 0.96\"", "1", "¥12", "I2C"],
          ["4", "Buzzer Module", "Active Buzzer", "1", "¥2", "GPIO"],
          ["-", "Breadboard", "830 holes", "1", "¥8", "Optional"],
          ["-", "Dupont Wires", "Male-Female 20pcs each", "1", "¥5", ""]
        ]
      }
    ],
    "warnings": [
      "Buzzer occupies strapping GPIO4, may briefly sound at startup (does not affect functionality)"
    ],
    "errors": [],
    "manifest_content": "{Complete updated project-manifest.json JSON text}"
  }
}
```

**manifest_content New/Updated Fields:**
- `phase`: `"select-hw"`
- `mcu`: `{model, board, chip_family, firmware_url, flash_tool}`
- `pinout`: `[{device, pin_name, gpio, physical_pin, type, side, pos, bus, i2c_addr, notes}]`
- `bom`: `[{name, model, quantity, unit_price_yuan, notes}]`

---

## IV. SKILL.md Modification Points

10 changes total, arranged by execution step:

| No. | Location | Current Behavior | Change To | Reason |
|-----|----------|-----------------|-----------|--------|
| 1 | Pre-checks | `python --version` | Delete. Dependency checks guaranteed by server environment | Plugin users cannot see server environment |
| 2 | Step 1 Case A | Check built-in `KNOWN_FIRMWARE` table + WebSearch for unknown models | `pre_selected_board` has value → skip entire Step 1. null but `mcu_specified` exists → read boards/*.json to find matching chip_family + built-in firmware_url, no WebSearch | Board database already has firmware info; WebSearch is unstable and slow |
| 3 | Step 1 Case B | LLM freely recommends, scoring logic hardcoded in SKILL.md | Change to read `matching-rules.json` for scoring + filter `beginner_friendly` | Rules managed centrally; adding new boards on plugin side does not require changing SKILL.md |
| 4 | Step 1 Case B | Plain text output recommendation | Change to `approval_request` #1 (MCU recommendation card), includes 1 recommendation + 2 alternatives | Plugin side cannot render command-line text |
| 5 | Step 2A Get Pin Diagram | Ask user to upload pin diagram → WebSearch | **Delete entire section.** Change to read `pin_layout` + `onboard_peripherals` from `boards/{id}.json` | Board database already has complete pin constraints; no need for user to upload images |
| 6 | Step 2B Multimodal Recognition | LLM extracts pin info from image | **Delete entire section.** Structured data reading replaces multimodal recognition | Structured data is 100% more accurate than image recognition |
| 7 | Step 2C Assign Pins | LLM assigns based on training data, does not consider existing onboard peripherals | Read `pin_layout.restricted_gpio` (avoid restricted areas) + `onboard_peripherals` (avoid pins occupied by onboard peripherals). flexible models prioritize `default_bus_pins`, fixed models select from `pin_options` | Based on structured data, avoid assigning pins already occupied by onboard peripherals |
| 8 | Step 2C Conflict Detection | LLM manually checks (unreliable) | After LLM assigns, call `pin-validator.py` for deterministic validation. Failure → read error details → reassign (up to 3 times) | LLM enumeration validation is unreliable; script is the last line of defense |
| 9 | New incremental mode | No such mode | Add mode determination: `incremental` only assigns pins for `new_devices`, does not touch `previous_pinout`, does not rerun MCU selection, does not recalculate full BOM (appends rows) | Support users adding devices during deploy phase |
| 10 | Step 4 Update manifest | `python update_manifest.py --project-dir {dir} --input {json}` writes local file | LLM generates updated manifest JSON → validated by `update_manifest.py` (stdin/stdout) → placed in `phase_complete.manifest_content` | Server side does not write to local disk |

---

## V. Validation Script Changes

### update_manifest.py (Existing, Needs Modification)

**Path:** `G:\MicroPython_Skills\upy-select-hw\scripts\update_manifest.py`

| Change | Content |
|--------|---------|
| Output method | `--project-dir` made optional. If not provided, reads existing manifest from stdin + LLM output, validates, merges, and outputs to stdout |
| New I2C address conflict detection | Already exists (in merge_manifest), keep unchanged |
| New restricted_gpio violation detection | Read restricted_gpio from boards JSON, check if each assigned GPIO is in the restricted area |
| New onboard_occupied conflict detection | Read onboard_peripherals (always_used=true) from boards JSON, check for conflicts |
| Support incremental mode | Use `--mode incremental --previous-pinout {json}` to validate only new pins |

### pin-validator.py (New Script)

**Path:** `G:\MicroPython_Skills\upy-select-hw\scripts\pin-validator.py`

```python
#!/usr/bin/env python3
"""
Pin assignment deterministic validator.

Validation items:
  1. GPIO overlap (non-shared pins cannot be assigned repeatedly)
  2. restricted_gpio violation (cannot assign restricted pins)
  3. I2C address conflict (same bus cannot have duplicate addresses)
  4. Bus count exceeded (assigned I2C/SPI/UART count vs specs hardware count)
  5. fixed model pin compliance (RP2040 etc., pins within pin_options and paired)
  6. onboard_occupied conflict (cannot occupy pins with always_used=true onboard peripherals)

Usage:
  python pin-validator.py --board boards/esp32-devkit-v1.json --pinout pinout.json
  python pin-validator.py --board boards/esp32-devkit-v1.json --pinout - < pinout.json
  python pin-validator.py --board boards/esp32-devkit-v1.json --pinout - --mode incremental --previous-pinout prev.json
  
Exit codes: 0=pass, 1=validation failed (error JSON output to stdout), 2=input format error
"""
```

**Is it mandatory: Yes.** LLM enumeration validation is unreliable; pin conflicts are catastrophic in hardware (short circuit/burn board).

---

## VI. UI Components Required on Plugin Side

| Component | Corresponding Message | Key Functionality |
|-----------|----------------------|-------------------|
| Progress Timeline | status_update × 4~8 messages | Reuse analyze timeline component |
| MCU Recommendation Card | approval_request #1 (conditional) | Single select + recommendation reason + spec summary + alternatives |
| Pin Assignment Table | phase_complete artifact[0] | Table: Device/Function/GPIO/Type/Notes, I2C shared rows highlighted |
| BOM Table | phase_complete artifact[1] | Table: Name/Model/Qty/Unit Price/Notes, total price highlighted |
| Warning Notification | phase_complete.warnings | Yellow warning bar, e.g., strapping pin warning |

---

## VII. Independent Test Scenarios

### Plugin-Side Testing (No Server)

1. Manually send `approval_request` #1 (MCU recommendation card) → Verify:
   - 3 board options render correctly (spec summary, price, reason)
   - Single select + three button behaviors
2. Manually send `phase_complete` (with pinout table + BOM table) → Verify:
   - I2C shared rows highlighted in pin table
   - BOM total price calculation displayed

### Skill-Side Testing (No Plugin)

1. **full mode + no pre-selected board:**
   - mock_plugin sends start_phase (mode=full, pre_selected_board=null, manifest=temp/humidity project)
   - Auto-reply to approval_request #1 with `{"action": "confirm"}`
   - Check all message JSON conforms to 02-protocol.md Schema
   - Check manifest passes update_manifest.py validation
2. **full mode + pre-selected board:**
   - mock_plugin sends start_phase (mode=full, pre_selected_board=esp32-devkit-v1)
   - Confirm approval_request #1 is skipped
   - Confirm pin assignment uses pin_layout from boards/esp32-devkit-v1.json
3. **incremental mode:**
   - mock_plugin sends start_phase (mode=incremental, previous_pinout=[6 existing pins], new_devices=[DHT22])
   - Confirm only DHT22 pins are assigned
   - Confirm existing 6 pins remain unchanged
4. **pin-validator validation failure + retry:**
   - LLM intentionally assigns GPIO12 (strapping) to LED
   - pin-validator.py returns error
   - LLM reads error, reassigns to GPIO13
   - pin-validator.py validates again and passes
