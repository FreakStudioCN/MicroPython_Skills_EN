# upy-analyze Interface Definition

> Status: ✅ Finalized
>
> Phase 1 — Requirements Analysis + Driver Search. Reads user natural language and plugin context, outputs project-manifest.json.

---

## I. Skill Overview

| Item | Content |
|------|---------|
| Phase | analyze |
| Upstream Skill | None (triggered by user input) |
| Downstream Skill | upy-select-hw |
| One-line Responsibility | Natural language → Intent decomposition → Device confirmation → Driver search → Output manifest |

**Core constraint:** No selection, no code generation, no pin assignment. MCU firmware verification is handled by upy-select-hw.

---

## II. Plugin Input → Skill (P→S)

The plugin sends **1 message** to the server to start this skill:

```json
{
  "type": "start_phase",
  "phase": "analyze",
  "session_id": "uuid-xxx",
  "payload": {
    "user_description": "Build a temperature and humidity monitor, buzzer alarm when threshold is exceeded",
    "pre_selected_board": {
      "id": "esp32-devkit-v1",
      "display_name": "ESP32 DevKit V1",
      "mcu": "ESP32-WROOM-32",
      "chip_family": "esp32",
      "firmware_url": "https://micropython.org/download/ESP32_GENERIC/"
    },
    "preferences": {
      "mode": "beginner",
      "locale": "zh"
    },
    "existing_hardware": []
  }
}
```

| Field | Type | Required | Source | Description |
|-------|------|----------|--------|-------------|
| `user_description` | string | Yes | User input field | Natural language, Chinese or English |
| `pre_selected_board` | object? | No | Plugin board selector | Has value if user pre-selected a board, null if select-hw should recommend |
| `pre_selected_board.id` | string | Yes | boards/*.json | Unique board ID |
| `pre_selected_board.display_name` | string | Yes | Same as above | UI display name |
| `pre_selected_board.mcu` | string | Yes | Same as above | MCU model |
| `pre_selected_board.chip_family` | string | Yes | Same as above | Chip family, passed to downstream select-hw |
| `pre_selected_board.firmware_url` | string | Yes | Same as above | Firmware URL already determined |
| `preferences.mode` | string | No | Plugin settings | "beginner" / "custom", default "beginner" |
| `preferences.locale` | string | No | Plugin settings | Default "zh" |
| `existing_hardware` | string[] | No | User profile | List of existing hardware, appended to device list |

**About pre_selected_board:**
- Has value → Device confirmation card shows selected main controller (greyed out, unchangeable), no MCU prompt
- null → Card shows "Not selected, will recommend intelligently", MCU left for select-hw to recommend
- User mentions board model in description but hasn't selected via plugin → LLM extracts to `mcu_specified` string written to manifest

---

## III. Skill Output → Plugin (S→P)

### Message Sequence

```
Step 1 Intent Decomposition
  → status_update "Analyzing requirements..."
  → status_update "Extracted N devices: xxx, xxx"

Step 2 Interactive Confirmation
  → approval_request #1: Device confirmation card

Step 3 Driver Search
  → status_update "Searching for drivers... (1/N)"
  → status_update "✓ SSD1306 → upypi" or "⚠ Buzzer → No driver"

Step 3B Alternative Recommendation (conditional, see below)
  → approval_request #2: Alternative device recommendation

Step 4 Output
  → phase_complete: Results panel
```

### Message Details

#### approval_request #1 — Device Confirmation Card

The only card the user must interact with. Combines device list + mode hint + board status.

**Structure Design (ASCII mockup):**

```
┌──────────────────────────────────────────┐
│  Confirm Project Plan                     │
│                                          │
│  Project  Temperature & Humidity Monitor  │
│  Function Periodic temp/humidity → Display → Alarm on threshold │
│  Main MCU ESP32 DevKit V1 (Selected)  ← If present               │
│          Not selected, will recommend  ← If absent                │
│                                          │
│  Device List:                            │
│  ☑ SHT30    I2C Temp/Humidity Sensor    User-specified           │
│  ☑ SSD1306  I2C OLED Display            System-recommended       │
│  ☑ Buzzer   GPIO Actuator               System-recommended       │
│  [+ Add Device]                          │
│                                          │
│  [Confirm, Start Driver Search]  [Modify Device List]             │
└──────────────────────────────────────────┘
```

**JSON Structure:**

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "device_confirm",
    "header": "Confirm Project Plan",
    "question": "Please confirm the following devices are correct",
    "summary": {
      "project_name": "Temperature & Humidity Monitor",
      "description": "Periodic temp/humidity collection → Display → Buzzer alarm on threshold",
      "board": {
        "status": "selected",
        "display_name": "ESP32 DevKit V1",
        "mcu": "ESP32-WROOM-32"
      }
    },
    "items": [
      {
        "id": "d1",
        "name": "SHT30",
        "subtitle": "I2C Temp/Humidity Sensor",
        "meta": "User-specified",
        "selected": true
      },
      {
        "id": "d2",
        "name": "SSD1306",
        "subtitle": "I2C OLED Display",
        "meta": "System-recommended",
        "selected": true
      },
      {
        "id": "d3",
        "name": "Buzzer",
        "subtitle": "GPIO Actuator",
        "meta": "System-recommended",
        "selected": true
      }
    ],
    "allow_add": true,
    "allow_remove": true,
    "multi_select": true,
    "actions": [
      { "label": "Confirm, Start Driver Search", "value": "confirm", "primary": true },
      { "label": "Modify Device List", "value": "modify" }
    ]
  }
}
```

**summary.board Field Description:**

| board.status | Meaning | Card Behavior |
|-------------|---------|---------------|
| `"selected"` | User pre-selected a board | Show main controller name + MCU model, greyed out, unchangeable |
| `"none"` | User did not select a board | Card does not show main controller, recommendation in select-hw phase |

#### approval_request #2 — Alternative Device Recommendation (Conditional)

**Trigger condition:** Device `source = "system_recommended"` (system-recommended) and driver search returns no results.

**Non-trigger condition:** `source = "user_specified"` (user-specified) and no driver → Cold hardware path, no alternative card, only hint in phase_complete warnings.

```
┌──────────────────────────────────────────┐
│  Temp/Humidity Sensor: Recommended Alt    │
│                                          │
│  SHT30: No MicroPython driver found       │
│                                          │
│  Same category (I2C Temp/Humidity Sensor) │
│  with existing drivers:                   │
│  ┌────────────────────────────────────┐  │
│  │ ★ HDC1080   upypi  Recommended      │  │
│  │   Accuracy ±0.2°C  Price ~¥8        │  │
│  ├────────────────────────────────────┤  │
│  │   AHT20    upypi                    │  │
│  │   Ultra-small package  Price ~¥5    │  │
│  └────────────────────────────────────┘  │
│                                          │
│  [Use HDC1080] [Use AHT20] [Keep SHT30]  │
└──────────────────────────────────────────┘
```

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "alternative_device",
    "header": "Temp/Humidity Sensor: Recommended Alternative",
    "question": "SHT30: No MicroPython driver found. Recommend the following alternatives in the same category with existing drivers:",
    "items": [
      {
        "id": "alt1",
        "name": "HDC1080",
        "subtitle": "Accuracy ±0.2°C, Price ~¥8",
        "meta": "upypi ★ Recommended",
        "selected": false
      },
      {
        "id": "alt2",
        "name": "AHT20",
        "subtitle": "Ultra-small package, Price ~¥5",
        "meta": "upypi",
        "selected": false
      }
    ],
    "allow_add": false,
    "allow_remove": false,
    "multi_select": false,
    "actions": [
      { "label": "Use HDC1080 (Recommended)", "value": "accept_alt1", "primary": true },
      { "label": "Use AHT20", "value": "accept_alt2" },
      { "label": "Keep SHT30 (Cold Hardware Path)", "value": "cold_driver" }
    ]
  }
}
```

#### status_update List

| step_id | level | message | Trigger Timing |
|---------|-------|---------|----------------|
| intent_extraction | info | Analyzing requirements... | Step 1 Start |
| intent_done | success | Extracted 3 devices: SHT30, SSD1306, Buzzer | Step 1 Complete |
| driver_search | info | Searching for drivers... (1/3) | Step 3 Start |
| driver_found | success | ✓ SSD1306 → upypi (ssd1306-driver v1.3.0) | Each device driver found |
| driver_fallback | success | ✓ SHT30 → GitHub (fallback) | Driver from GitHub |
| driver_none | warn | ⚠ Buzzer → No driver needed (standard GPIO) | No driver |
| driver_cold | warn | ⚠ SHT30 → No driver, will use cold hardware path | User-specified device without driver |

#### phase_complete

```json
{
  "type": "phase_complete",
  "payload": {
    "phase": "analyze",
    "result": "success",
    "summary": "Device analysis complete: 2 drivers found for 3 devices, 1 requires no driver",
    "next_phase": "select-hw",
    "artifacts": [
      {
        "type": "table",
        "title": "Device Driver Status",
        "headers": ["Device", "Type", "Interface", "Driver Source", "Status"],
        "rows": [
          ["SSD1306", "OLED", "I2C", "upypi", "✓ ssd1306-driver v1.3.0"],
          ["SHT30", "Temp/Humidity", "I2C", "none", "⚠ Cold hardware path (user-specified)"],
          ["Buzzer", "Actuator", "GPIO", "—", "✓ No driver needed"]
        ]
      }
    ],
    "warnings": [
      "SHT30 is a user-specified device with no existing driver; will use cold hardware path. To switch to a temp/humidity sensor with an existing driver, manually replace it in the next step and re-analyze"
    ],
    "errors": [],
    "manifest_content": "{Complete project-manifest.json JSON text}"
  }
}
```

---

## IV. SKILL.md Modification Points

12 changes total, arranged by execution step:

| No. | Location | Current Behavior | Change To | Reason |
|-----|----------|-----------------|-----------|--------|
| 1 | Pre-check | `python --version` + `python -c "import requests"` | Delete. Dependency check guaranteed by server environment | Plugin user cannot see server environment |
| 2 | Step 1 | No change | Logic unchanged. New: read `pre_selected_board` and `preferences` | Receive plugin context |
| 3 | Step 2A | AskUserQuestion select beginner/custom | **Delete entire section**. Change to read `preferences.mode`, default "beginner" | Mode is user preference, should not ask every time |
| 4 | Step 2B Q1 | AskUserQuestion select MCU | **Conditionalize**: `pre_selected_board` has value → skip; null → do not ask at this stage, write `mcu_specified=null` | MCU confirmation deferred to select-hw |
| 5 | Step 2B Q2 | AskUserQuestion confirm devices | Change to `approval_request` #1 | Merge into one card |
| 6 | Step 2C | AskUserQuestion scenario/power/performance/output (max 4 questions) | **Beginner mode: skip, use defaults. Custom mode: optional second card**. Append an approval_request when `preferences.mode="custom"` | Simplify interaction |
| 7 | Default values summary table | Table lists 13 default values | Table changes to split by `preferences.mode`: `beginner` → fill all defaults; `custom` → fill after user confirmation | Logic adjustment |
| 8 | Step 3 Driver search | Silent search, no output | Send `status_update` after each device search | Provide progress signal to plugin |
| 9 | Step 3B Alternative recommendation | Plain text output to command line | Text table → structured `approval_request` #2 (includes device.source check for trigger condition) | Plugin cannot render command-line text |
| 10 | Step 3 Device source marker | No such field | devices[i] adds `source` field, enum `"user_specified"` / `"system_recommended"` | Distinguish user-specified vs system-recommended |
| 11 | Step 4 Output manifest | `python init_manifest.py --project-dir {dir} --input {json}` writes local file | LLM generates manifest JSON → `script_run(init_manifest.py --stdin)` plugin-side validation → LLM reads validation result → places in `phase_complete.manifest_content`. init_manifest.py runs on plugin side, not server side | Server does not execute scripts |
| 12 | Strong constraints | Item 2 "Must not assume device model when unclear" | Keep unchanged | Core constraint unchanged |

---

## V. Validation Script Changes

### init_manifest.py

**Path:** `G:\MicroPython_Skills\upy-analyze\scripts\init_manifest.py` (copied by scaffold to `{project}/.upy/scripts/init_manifest.py`, executed on plugin side)

**2 changes needed:**
- Add `--stdin`: Read manifest JSON from stdin, validate, output `{"status":"ok","manifest":{...}}` or `{"status":"fail","errors":[...]}` to stdout
- Remove file writing: No longer write to local disk, validation result processed by LLM and placed in `phase_complete.manifest_content`

**Mandatory: Yes.** Last line of defense — ensures invalid enum values do not reach downstream. init_manifest.py runs on plugin side, server LLM cannot directly execute scripts.

---

## VI. UI Components Required on Plugin Side

| Component | Corresponding Message | Key Functionality |
|-----------|----------------------|-------------------|
| Progress Timeline | status_update × 3~8 messages | Completed (✓) / In Progress (spinning) / Warning (⚠) / Failed (✗) |
| Approval Card | approval_request #1 | Device multi-select add/remove + board status display + "Confirm"/"Modify" buttons |
| Alternative Recommendation Card | approval_request #2 (conditional) | Alternative device single-select + "Keep original device" option |
| Results Panel | phase_complete | Device status table + warning info + next step preview |

---

## VII. Independent Test Scenarios

### Plugin-Side Testing (No Server)

1. Manually send `status_update` sequence → Verify timeline renders each message
2. Manually send `approval_request` #1 (JSON above) → Verify:
   - Devices can be selected/deselected
   - "Add Device" button pops up input box
   - Clicking "Confirm" sends `approval_response`
3. Manually send `approval_request` #2 (alternative device) → Verify single-select + three buttons
4. Manually send `phase_complete` → Verify table + warning display

### Skill-Side Testing (No Plugin)

1. Use mock_plugin.py to simulate plugin responses:
   - Manually construct start_phase message to LLM (with user_description + pre_selected_board + preferences)
   - Auto-reply to approval_request #1 with `{"action": "confirm"}`
   - Auto-reply to approval_request #2 with `{"action": "accept_alt1"}`
2. Check all sent message JSON conforms to 02-protocol.md Schema
3. Check manifest JSON passes init_manifest.py validation
