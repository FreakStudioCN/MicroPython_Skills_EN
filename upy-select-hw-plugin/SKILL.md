---
name: upy-select-hw-plugin
description: Plugin-based workflow version of select-hw. Consumes phase_complete.payload.manifest_content from upy-analyze-plugin, completes board/MCU confirmation, MicroPython firmware verification, pin assignment, and BOM, and outputs phase_complete(select-hw) for the MPY firmware flashing stage.
---

# Plugin-based Workflow Hardware Selection and Pin Assignment Skill

## Role Definition

`upy-select-hw-plugin` is the `select-hw` phase in the long-running workflow protocol. It consumes the phase output from `upy-analyze-plugin` and prepares hardware facts for the subsequent "MicroPython firmware flashing step for the corresponding MCU".

This phase is only responsible for:

- Reading `phase_complete(analyze).payload.manifest_content`
- Selecting/confirming a MicroPython development board based on `requirements` and `devices`
- Verifying the firmware download entry point and flashing tool type
- Assigning pins based on the board's `pin_layout` and device interfaces
- Generating a BOM and estimated total cost
- Validating/normalizing via `select_hw_manifest.py`
- Outputting `phase_complete(select-hw)`, with `next_phase` fixed to `upy-flash-mpy-firmware-plugin`

This phase is NOT responsible for:

- Re-analyzing user natural language
- Searching for or generating drivers
- Generating business code
- Flashing devices
- Using local disk write results directly as phase fact sources

## Input Fact Source

The formal input is the upstream message:

```text
phase_complete(analyze).payload.manifest_content
```

During direct testing, reading from the session directory `phase_complete.analyze.json` is allowed, but `payload.manifest_content` must still be extracted. Do not infer project status from `manifest_draft.json`, logs, or old conversations.

The current real direct test output of `upy-analyze-plugin` uses session isolation:

```text
sessions/<session_id>/
  manifest_draft.json
  manifest_validated.json
  phase_complete.analyze.json
  driver_search_log.md
  analyze_phase_log.md
```

Formal consumption order:

1. Prefer `payload.manifest_content` from `phase_complete.analyze.json`
2. Direct test fallback can read `manifest_validated.json`
3. `manifest_draft.json`, `driver_search_log.md`, `analyze_phase_log.md` are for troubleshooting reference only

`start_phase.payload.user_pin_constraints` is optional. If the plugin or upstream analyze has already parsed user-specified pins, they must be passed to select-hw as a structured array. Each item must contain at least `device`, `device_pin`, `mcu_pin`, `signal`, and optionally `voltage` and `notes`.

## Path and Root Conventions

Three roots must be distinguished:

| root | Meaning | Allowed Content |
| --- | --- | --- |
| `resource_root` | Root where skill/resources reside, typically `G:\MicroPython_Skills` or the parent of an installed `.claude/skills` | `upy-select-hw-plugin`, `upy-analyze-plugin/boards` |
| `artifact_root` | Current project/test output root, e.g., user-provided `G:\test\test`; `phase_complete.payload.artifacts.file_list.files[].path` is resolved relative to this by default | `sessions/<session_id>` and phase outputs |
| `session_root` | Current session directory | `select_hw_*.json`, `phase_complete.select_hw.json`, logs |

Resource loading must use relative paths based on `resource_root`, for example:

```text
upy-analyze-plugin/boards
upy-analyze-plugin/sample/phase_complete.analyze.success.json
upy-select-hw-plugin/scripts/select_hw_manifest.py
upy-select-hw-plugin/sample/phase_complete.select_hw.success.json
```

Artifact writing must use relative paths based on `artifact_root` or `session_root`, for example:

```text
sessions/<session_id>/select_hw_draft.json
sessions/<session_id>/select_hw_validated.json
sessions/<session_id>/phase_complete.select_hw.json
```

`artifact_root` is the "root directory for this run's artifacts", not the skill/resource root. For example, when `artifact_root=G:\test\test`, the artifact path should be `sessions/<session_id>/select_hw_draft.json`; if the host sets `artifact_root` to the current `session_root`, then the artifact path should be `select_hw_draft.json`. Validation commands, phase logs, and file manifests must use the same root convention.

The user-provided project directory (e.g., `G:\test\test`) defaults to `artifact_root`, not `resource_root`. Do not copy skill scripts, the boards directory, or partial skills into `artifact_root` just because `upy-select-hw-plugin` or `upy-analyze-plugin` is missing from `artifact_root`.

Do not write `G:\MicroPython_Skills` as a business dependency in protocols, script parameters, or examples. Test commands can show absolute paths in documentation, but the implementation must use `resource_root / relative_path` to read resources and `artifact_root / relative_path` to write artifacts.

Phase logs, command history, and artifact descriptions must also use relative paths. Do not write the local plugin installation directory (e.g., the skill/plugin path under the user directory) as a business fact source.

If the host can only execute scripts within the artifact workspace, the `resource_root` must be explicitly passed as a read-only resource path, or the host must provide a script execution capability; do not fake relative paths by copying `upy-select-hw-plugin/scripts` into the artifact workspace.

### runtime_context Conventions

Claude Code / plugin runtime must pass the current working directory and session directory conventions via `phase_complete.payload.runtime_context`. The skill must not guess root directories:

```json
{
  "runtime_context": {
    "artifact_root": ".",
    "artifact_root_mode": "cwd",
    "session_root": "sessions/<session_id>",
    "resource_root": "<runtime-provided>"
  }
}
```

Field constraints:

| Field | Required | Meaning |
| --- | --- | --- |
| `artifact_root` | Yes | Artifact root directory, defaults to `.` (current working directory) |
| `artifact_root_mode` | Yes | `cwd` or `session_root` |
| `session_root` | Yes | Relative path to the current session directory |
| `resource_root` | Yes | Root where skill/resources reside (provided by runtime) |

Path convention rules:

- When `artifact_root_mode=cwd`, `file_list.files[].path` must be relative to the current working directory, formatted as `sessions/<session_id>/<filename>`.
- Only when `artifact_root_mode=session_root` are bare filenames (e.g., `select_hw_draft.json`) allowed.
- Do not mix the two path conventions within the same `phase_complete`.
- Missing `runtime_context` is treated as an error during validation.

## Time Rules

All time fields must come from a unified runtime time source. Hardcoded placeholder times are forbidden:

- `timestamp` — Injected by Claude Code / plugin runtime, or obtained via `upy-project-gen-toolchain-spec/scripts/workflow_time.py`.
- `pin_review.confirmed_at` — Must be the real UTC time when user confirmation occurred. Date zero or sample placeholder values are forbidden.
- `manifest_content.created_at` / `updated_at` — Automatically generated by `select_hw_manifest.py` during normalization.
- All time fields must be ISO-8601 format with UTC timezone (`Z` suffix).

`confirmed_at` write order: Must first be written back to `select_hw_draft.json`, then use the draft as the single fact source to generate `select_hw_validated.json` and `phase_complete.select_hw.json`.

`phase_complete.timestamp` must be obtained by re-calling `workflow_time.py --json` after script validation passes, ensuring it is >= all referenced artifacts' `updated_at` and `created_at`. Reusing `pin_review.confirmed_at` or an earlier time as `phase_complete.timestamp` is forbidden.

## Long-Running Protocol Requirements

All formal messages must use the complete envelope:

```json
{
  "protocol_version": "1.0",
  "msg_id": "uuid",
  "session_id": "uuid",
  "phase": "select-hw",
  "timestamp": "<runtime-utc-now>",
  "type": "status_update",
  "idempotency_key": "select-hw:<session_id>:step:v1",
  "retry_of": null,
  "payload": {}
}
```

Field constraints:

| Field | Required | Description |
| --- | --- | --- |
| `protocol_version` | Yes | V0 fixed to `"1.0"` |
| `msg_id` | Yes | UUID of the current message |
| `session_id` | Yes | Created by the plugin, inherited by phases |
| `phase` | Yes | Current phase, fixed to `select-hw` |
| `timestamp` | Yes | UTC ISO time |
| `type` | Yes | Message type enum |
| `idempotency_key` | Recommended | Remains unchanged during retries of the same action |
| `retry_of` | Optional | Points to the original failed message |

Message type enum:

```text
start_phase
status_update
approval_request
approval_response
script_run
script_result
file_operation
file_result
device_command
device_result
phase_complete
```

## Capability Negotiation

Host capabilities should be known before starting:

```json
{
  "capabilities": {
    "protocol_versions": ["1.0"],
    "approval_request": true,
    "script_run": true,
    "file_operation": true,
    "device_command": false,
    "artifact_root": true,
    "relative_paths": true
  }
}
```

V0 does not require `device_command`. If the host does not support `approval_request` or `script_run`, select-hw success must not be claimed.

## Standard Message Sequence

```text
Step 0 Read upstream manifest
  -> status_update(upstream_manifest_loaded)

Step 1 Board candidate generation
  -> status_update(board_matching)
  -> approval_request(board_select)  # Can be skipped if pre_selected_board comes from plugin UI; if board library is missing, change to board_unavailable
  <- approval_response

Step 1B Load full board definition
  -> status_update(board_definition_loaded)
  Load full board JSON from upy-analyze-plugin/boards/<selected_board.id>.json
  If it does not exist or lacks pin_layout:
    -> approval_request(board_unavailable or board_select)

Step 2 Firmware verification
  -> status_update(firmware_check)
  -> status_update(firmware_ok)

Step 3 Pin assignment
  -> status_update(pin_assignment)
  If the candidate board lacks pin_layout:
    -> Select a known board with similar functionality and pin_layout
    -> approval_request(board_select)
  If start_phase.payload.user_pin_constraints or approval_response.payload.user_pin_constraints exists:
    -> Prioritize generating pinout/pin_decisions based on user-specified pins; `pinout[].source` must be `user_wiring`
  -> status_update(pin_assignment_draft_ready)

Step 3B Pin plan review
  -> approval_request(pin_plan_review)
  After user confirmation:
    -> status_update(pin_assignment_done)
  If user requests adjustments or does not confirm:
    -> phase_complete(result=partial, checkpoint.resume_step=pin_assignment)

Step 4 BOM generation
  -> status_update(bom_ready)

Step 5 Manifest validation/normalization (1st: draft -> validated)
  -> script_run(select_hw_manifest.py --input <draft> --write-path <validated> --board-root ...)
  <- script_result

Step 6 Manifest content secondary validation (2nd: validate manifest_content)
  -> script_run(select_hw_manifest.py --validate-manifest-content --input <validated> --board-root ...)
  <- script_result

Step 7 Get phase completion timestamp
  -> Call workflow_time.py --json to get current UTC time
  <- phase_timestamp = <utc-now>

Step 8 Phase_complete final validation and output (3rd: validate phase_complete)
  -> script_run(select_hw_manifest.py --validate-phase-complete --input <phase_complete.json> --compare-manifest <validated> --artifact-root ... --expected-artifact ...)
  <- script_result
  -> After validation passes, output phase_complete(timestamp=<phase_timestamp>, result=success, next_phase=upy-flash-mpy-firmware-plugin)
```

## status_update Enum

Only use these levels:

```text
info
warn
error
success
```

step_id enum:

```text
upstream_manifest_loaded
board_matching
board_unavailable
board_definition_loaded
board_definition_invalid
board_selected
board_change_requires_restart
firmware_check
firmware_ok
pin_assignment
pin_assignment_draft_ready
pin_plan_review
pin_risk_detected
pin_conflict
pin_assignment_done
bom_ready
manifest_validation
```

## approval_request: board_select

`requirements.mcu_specified` indicates the MCU/chip/module model, which is not equivalent to a specific development board. Therefore, `board_select` must be triggered by default. If `pre_selected_board` already comes from the plugin UI, it can be skipped, but the reason for skipping must be recorded, and the board's firmware and `pin_layout` must be validated.

Board confirmation boundaries:

- `select-hw` is only allowed to confirm a specific development board within the compatible range of the MCU/chip/module already determined by the upstream `requirements`, or to select from the candidate pool when no MCU is specified.
- If the user requests a board change in `select-hw` that crosses MCU, chip family, firmware target, or significantly changes the main controller's capability boundary, the upstream requirements must not be silently rewritten and `success` output.
- Such changes must output `partial`, `next_phase=null`, `checkpoint.resume_step=load_upstream_manifest` or `board_select`, `reason=board_change_requires_analyze`, and remind the user to start a new conversation or re-run analyze/select-hw.
- The general judgment basis is whether the upstream `requirements.mcu_specified`, the confirmed `pre_selected_board`, and the candidate board's `mcu`, `chip_family`, and `firmware.board_name` are still compatible. The rules must not be written as special cases for a specific MCU.

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "board_select",
    "header": "Confirm Main Board",
    "question": "Please confirm the MicroPython development board for this project",
    "summary": {
      "project_name": "Voice Dialogue Assistant",
      "mcu_specified": "ESP32-C3",
      "source_phase": "analyze"
    },
    "items": [
      {
        "id": "esp32-c3-devkitm",
        "name": "ESP32-C3-DevKitM-1",
        "subtitle": "WiFi/BLE, MicroPython ESP32_GENERIC_C3",
        "meta": "Matches upstream MCU preference",
        "selected": true
      }
    ],
    "multi_select": false,
    "actions": [
      {
        "label": "Confirm Board",
        "value": "confirm",
        "primary": true
      },
      {
        "label": "Continue Later",
        "value": "save_partial"
      }
    ]
  }
}
```

When the user cancels or chooses to continue later, output:

```text
result = partial
next_phase = null
checkpoint is required
```

## Device Consistency Boundary

`select-hw` can only consume real functional devices already confirmed in the upstream `upstream_manifest.devices[]`. It must not silently add/replace sensors, displays, actuators, audio modules, communication modules, or other functional hardware in pin assignments, BOM, or the final `manifest_content.devices[]`. If the user requests adding or replacing functional devices during this phase, output `partial` and return to analyze/select-hw to re-confirm the device list.

Hard rules:

- `hardware_plan.pinout[].device` must match an upstream `devices[].name`. Only power/system items like `power`, `GND`, `3V3`, `5V`, `board`, `mcu` are exceptions.
- Functional hardware in `hardware_plan.bom[]` must be mappable to upstream `devices[]` via `name`, `model`, `device`, or `selected_model`. Supporting materials like Dupont wires, breadboards, resistors, capacitors, pin headers, USB cables, enclosures, battery holders, power modules, and adapter boards do not need to be in the upstream device list.
- BOM fields like `url`, `link`, `product_url`, `shop_url`, `datasheet_url`, `supplier`, `sku` can be retained, but they are not a strong contract passed to generate/deploy and must not be a prerequisite for downstream success.
- All physical BOM items must retain `product_url`, `shop_url`, `datasheet_url`, `supplier`, `sku` according to `references/bom_item_link_index.template.json`; write empty strings for unknowns. Empty links must not block select-hw success or progression to the next phase.
- Do not fabricate real store URLs, suppliers, or SKUs to fill in links. The link index is only a procurement display template and does not change the hardware fact boundary.
- If `select-hw` supplements a specific model for an upstream generic name, e.g., `OLED display` -> `SSD1306 OLED`, it should be recorded as a model supplement or BOM/model information for the original device, not appended as a new `devices[]` item. If the model affects the driver/API, it must remain mappable back to the original upstream device.

## approval_request: board_unavailable

When the user-specified specific board or `pre_selected_board.id` is not in `upy-analyze-plugin/boards`, do not fail immediately. First, sort by the same series, same `chip_family`, same firmware port, and similar functional requirements, recommend a known alternative board with `pin_layout`; also give the user the option to manually describe the wiring.

These mutually exclusive actions must be provided:

| action value | Meaning | Subsequent Behavior |
| --- | --- | --- |
| `use_recommended_similar` | Use the system-recommended known board from the same series/similar functionality | Continue with firmware verification and pin assignment |
| `select_known_board` | User selects another known board from the board library | Re-enter `board_select` |
| `manual_wiring_description` | User manually describes "MCU pin -> device pin" | Output partial/checkpoint, wait for user to provide structured wiring |
| `save_partial` | Pause | Output partial/checkpoint |

Manual wiring description requires an array, each record specifying `mcu_pin`, `device`, `device_pin`, `signal`, `voltage`, `notes`. Example: `GPIO21 -> AHT20 SDA`, `3V3 -> AHT20 VCC`, `GND -> AHT20 GND`.

## approval_request: pin_plan_review

Pin assignment must not rely solely on a one-time LLM inference. V0 adopts a simplified strategy of "script only catches hard errors + user confirms the plan": after generating the preliminary `pinout` and `pin_decisions`, the user must confirm the pin plan, and be reminded to check the official schematic, board silkscreen, module version, and peripheral datasheet. `phase_complete(result=success)` must not be output before user confirmation.

Users must be reminded to focus on checking:

- Whether default bus pins are actually exposed and conflict with other devices
- `restricted_gpio`, boot/strapping, flash/PSRAM, USB/JTAG/REPL/UART occupancy
- Whether `onboard_peripherals` actually occupy that GPIO and whether it can be released
- Whether the peripheral module's VCC/GND/configuration pins are MCU-controlled or hardwired to power/ground
- Board variant differences, schematic version, silkscreen, and actual module consistency

Action enum:

| action value | Meaning | Subsequent Behavior |
| --- | --- | --- |
| `confirm_pin_plan` | User confirms the pin plan can proceed with the current draft | Continue with BOM and final validation |
| `revise_pin_plan` | User requests reassignment of one or more pins | Return to `pin_assignment` |
| `manual_wiring_description` | User manually describes wiring | Output partial/checkpoint, wait for structured wiring |
| `save_partial` | Pause | Output partial/checkpoint |

When the user selects `revise_pin_plan` or `manual_wiring_description`, the plugin should return structured pin constraints in `approval_response.payload.user_pin_constraints`:

```json
{
  "approval_id": "pin_plan_review",
  "action": "revise_pin_plan",
  "user_pin_constraints": [
    {
      "device": "AHT20",
      "device_pin": "SDA",
      "mcu_pin": "GPIO21",
      "signal": "i2c_data",
      "voltage": "3.3V",
      "notes": "User specified SDA"
    }
  ]
}
```

`user_pin_constraints[]` field meanings:

| Field | Required | Meaning |
| --- | --- | --- |
| `device` | Yes | Device name, must correspond to upstream `devices[].name` or current `pinout[].device` |
| `device_pin` | Yes | Device-side pin/signal name, e.g., `SDA`, `SCL`, `OUT`, `DIN`, `VCC`, `GND` |
| `mcu_pin` | Yes | User-specified MCU/board-side pin. GPIO can be written as `GPIO21` or `21`; power/ground can be written as `3V3`, `5V`, `GND` |
| `signal` | Yes | Pin function type, should map to `pinout[].type`, e.g., `i2c_data`, `i2c_clock`, `gpio_in`, `gpio_out`, `uart_tx`, `uart_rx`, `power_3v3`, `gnd` |
| `voltage` | No | Voltage description, e.g., `3.3V`, `5V`, `0V`, only used for validation hints and notes |
| `notes` | No | User description or plugin UI remarks |

Processing rules:

- Convert valid `user_pin_constraints[]` to `pinout[]`, setting `pinout[].source="user_wiring"`.
- Simultaneously generate `pin_decisions[]`, setting `decision_type="user_wiring"`, `source="user_wiring"`.
- `GPIO21` and `21` in `mcu_pin` are treated as the same GPIO; power/ground must remain as `3V3`, `5V`, `GND`.
- If required fields are missing, do not proceed to success; output `partial`, `checkpoint.resume_step=pin_assignment`.
- User-specified pins must still pass board JSON validation; illegal pins must not be silently rewritten.

The payload must include:

```json
{
  "approval_id": "pin_plan_review",
  "header": "Confirm Pin Assignment",
  "summary": {
    "board_id": "selected-board-id",
    "board_definition": "upy-analyze-plugin/boards/<board_id>.json",
    "requires_schematic_review": true
  },
  "pinout": [],
  "pin_decisions": [],
  "warnings": []
}
```

After confirmation, write to `hardware_plan.pin_review`:

| Field | Required | Meaning |
| --- | --- | --- |
| `approval_id` | Yes | Fixed to `pin_plan_review` |
| `confirmed` | Yes | Whether the user confirmed; must be `true` before `success` |
| `confirmed_by` | Required when confirmed=true | User, plugin UI, or approval source |
| `confirmed_at` | Required when confirmed=true | Real UTC time of this confirmation, ISO-8601 format; must not use sample placeholder times or date zero |
| `source` | Yes | `approval_response`, `plugin_ui_confirmed`, `user_confirmed` |
| `note` | Optional | User confirmation or adjustment description |

## Board Data

V0 reuses relative paths:

```text
upy-analyze-plugin/boards
```

Do not copy board data to `artifact_root`, unless subsequent select-hw needs to independently extend the schema. If test staging requires copying, the complete `boards` directory must be copied (at least all board JSONs and `matching-rules.json`), not just the single selected board JSON; otherwise, it will break the flow for unspecified MCU, candidate sorting, similar board recommendation, and board_unavailable.

Processing strategy:

- The candidate generation phase must enumerate the complete board library from `resource_root/upy-analyze-plugin/boards/*.json`, skipping `_template.json` and documentation files; do not load only the selected board.
- When `requirements.mcu_specified` exists, match candidates by `mcu`, `chip_family`, `firmware.board_name`.
- If `pre_selected_board` already comes from the plugin UI, confirmation can be skipped, but validation is still required.
- `selected_board.id` must correspond to `upy-analyze-plugin/boards/<id>.json`. After confirming the board, the full board JSON must be loaded; pin assignment based solely on MCU name or `selected_board` summary is not allowed.
- The full board JSON is the fact source for `firmware`, `pin_layout`, `restricted_gpio`, `onboard_peripherals`. `selected_board` can only be a UI summary.
- When no MCU is specified, the candidate pool must prioritize the Pico/RP2 series and ESP32 series; unless the requirements clearly need another series, do not prioritize boards like STM32, Teensy, Pyboard.
- Default sorting for unspecified MCU: Pico/Pico W, ESP32 DevKit, ESP32-S3, ESP32-C3; output Top 1 and Top 2 alternatives after scoring based on requirements.
- When WiFi/BLE is needed, boost ESP32 series and Pico W; when AI/voice/camera is needed, boost ESP32-S3; low power/battery power boosts ESP32-C3; pure GPIO or beginner-friendly boosts Pico/Pico W; extreme low cost can boost ESP8266/Pico, but ESP8266 should not outweigh Pico/ESP32 unless budget is the sole primary constraint.
- When the user-specified board is not in the board library, prioritize recommending a known board from the same series or with similar functionality and `pin_layout`; simultaneously send `approval_request(board_unavailable)`, allowing the user to select a known board or manually describe wiring.
- When the user requests a board change in `select-hw` that crosses MCU/chip family/firmware target, do not continue with a successful output; output partial/checkpoint, requiring a new conversation or returning to the analyze phase to re-confirm requirements.
- When `pin_layout` is missing, default to swapping to a known board with similar functionality and `pin_layout`.
- `cold-driver` does not affect MCU recommendation, pin assignment, or BOM; however, `devices[].driver.source="cold-driver"` must be normalized to `devices[].driver.status="cold_driver_required"` for use by the pre-generate gate and `upy-gen-driver-plugin`. `driver.source` is retained as provenance, not as a workflow gate.
- select-hw is not responsible for confirming the real-time latest version of the MicroPython firmware. Firmware versions in the board library are only cache information; the formal flashing stage should check `firmware.url` for the latest release. The select-hw output focuses on retaining `firmware_url`, `firmware_board_name`, `flash_tool`.

## Pin Assignment Rules

Basic rules:

- I2C devices share one I2C bus by default and prioritize using `pin_layout.default_bus_pins`; if `i2c_addr` conflicts, use a second I2C or output partial.
- SPI devices share MOSI/MISO/SCK, each device has its own CS, and prioritize using `pin_layout.default_bus_pins`.
- UART avoids REPL/USB serial ports.
- I2S requires BCK/WS/DIN/DOUT allocation; microphones and amplifiers can share BCK/WS, but data directions differ.
- ADC can only use ADC-capable pins.
- GPIO avoids boot/strapping, flash/PSRAM, USB OTG, read-only pins; conditionally usable pins can enter the plan, but must prompt the user to verify in warnings/notes and `pin_plan_review`.
- Power and GND must be included in `pinout`.
- If the board JSON has `pin_options`, remapping is only allowed within the `pin_options` range; if it is a flexible matrix, hard-disabled pins must also be avoided. Conditionally usable pins are not schema hard failures; they are handled via warnings and user confirmation.
- Deviations from `pin_layout.default_bus_pins` must be explained in `pinout[].notes` and warnings.
- When the user provides wiring, prioritize retaining it, but it must pass the board JSON's restricted/occupied validation; illegal user wiring must not silently succeed.
- User-specified pins only indicate preferences/constraints, not a bypass of safety checks. If a specified pin hits a hard forbidden rule, input/output direction mismatch, or is occupied by an `always_used` onboard peripheral, output `partial`, `checkpoint.resume_step=pin_assignment`, and explain the conflict reason in `structured_errors`; illegal pins must not be silently rewritten, and do not automatically swap pins and continue to success.
- When an onboard device matches a user-specified or system-recommended device, reuse the onboard default/occupied pins declared in `onboard_peripherals`, do not allocate additional external GPIOs, and do not add duplicate entries to the BOM.
- When an onboard device enters the project device list, do not change `devices[].source` to a new enum; keep `source` as `user_specified` or `system_recommended`, and additionally write `physical_source="board_onboard"`, `onboard_peripheral_ref={board_id,index,name,type,model,occupied_pins,verification}`. This allows downstream legacy plugins to still read `devices[]` while identifying it as not an external BOM item.
- If the user's requirements include a display, IMU, microphone, camera, SD/storage, LoRa, Ethernet, LED/button, power management, etc., and the selected board JSON's `onboard_peripherals` has a matching item, prioritize reusing the onboard peripheral; if `occupied_pins` is empty or `verification` is marked `needs_pin_mapping`, only record it as a non-hard constraint and require verification in notes/warnings.
- Onboard peripheral drivers only use deterministic mappings or existing `devices[].driver`. Do not guess package names for unverified models; set `driver.source="cold-driver"` and `driver.status="cold_driver_required"`, leaving it to `upy-gen-driver-plugin`.
- When an onboard device does not match the current requirements, `onboard_peripherals[].occupied_pins` are considered occupied resources, and external devices can only use free pins.
- If the user requests releasing pins occupied by an onboard device, `always_used=false` must be confirmed, and the release reason must be explained in notes/warnings.
- GPIO summaries in `pin_assignment_log.md` and phase logs must be calculated from the full board JSON and the final `pinout`, not hardcoded static lists. V0 must include at least `used_gpio`, `unused_gpio`, `restricted_or_occupied_gpio` groups; do not package conditionally usable pins as absolutely safe, just explain the limitations and confirmation points in warnings.
- If WiFi is enabled and GPIOs from `adc2_wifi_conflict` are used, all related GPIOs must be fully listed. Conflict only occurs when `pinout[].type=adc`; using them as digital signals like I2C/I2S/GPIO is allowed, but warnings or notes must state "WiFi only affects ADC readings, not digital usage".
- When using board default UART/REPL/USB serial port related pins as regular GPIOs, it must be confirmed that the serial port is not used for debugging/communication, or the occupancy must be written into a warning with reassignment suggestions.

`restricted_gpio` levels:

| board field | Default Strategy | Validation Level |
| --- | --- | --- |
| `flash_psram_occupied` | Forbidden | error |
| `reserved` / `internal_only` | Forbidden | error |
| `usb_serial_pins` | Forbidden by default, unless USB serial is explicitly not used or user explicitly wires | error or warning |
| `strapping` / `boot` | Avoid by default; if must be used, write warning and pass to pin_plan_review | warning; error in strict mode |
| `input_only` | Can only be used for input-type pin types | error |
| `adc_only` | Can only be used for ADC input | error |
| `adc2_wifi_conflict` | Only conflicts when `type=adc` and WiFi is enabled; digital input/output is allowed but should be noted | error for ADC; warning for digital use |
| `onboard_peripherals[].occupied_pins` | Forbidden when `always_used=true`; otherwise avoid by default or explain release reason | error or warning |

`pinout[].type` enum:

```text
power_3v3
power_5v
gnd
i2c_data
i2c_clock
spi_mosi
spi_miso
spi_sck
spi_cs
uart_tx
uart_rx
gpio_out
gpio_in
gpio_in_pullup
adc
pwm
i2s_bck
i2s_ws
i2s_data_in
i2s_data_out
wifi_internal
reserved
```

`pinout[]` field meanings:

| Field | Required | Meaning |
| --- | --- | --- |
| `device` | Yes | Connected device name, `power` can be used for power items |
| `pin_name` | Yes | Device-side signal name, e.g., `SDA`, `SCL`, `VCC`, `GND`, `OUT` |
| `gpio` | Yes | MCU-side GPIO number or power name, e.g., `8`, `3V3`, `GND` |
| `type` | Yes | Pin electrical type, can only be from the `pinout[].type` enum above |
| `bus` | Optional | Bus number, e.g., `i2c0`, `spi0`, `uart1`, `i2s0` |
| `i2c_addr` | Optional | I2C address, used for conflict detection |
| `physical_pin` | Optional | Board silkscreen/physical pin number, filled when board library has data |
| `side` | Optional | Which side of the board the pin is on, suggested `left/right/top/bottom` |
| `pos` | Optional | Sequential position on the `side`, suggested 0-based |
| `notes` | Optional | Limitations, reuse, or alternative reasons |
| `source` | Recommended | Pin source, can only be `default_bus`, `auto_assigned`, `user_wiring`, `onboard_peripheral`, `power` |

## pin_decisions and deviation

Structured `pin_decisions[]` must be generated and retained in the final `manifest_content`. Default bus, auto-assignment, user wiring, onboard device reuse, fixed power/ground all require corresponding decisions; natural language notes can only supplement, not replace structured evidence.

`pin_decisions[]` fields:

| Field | Required | Meaning |
| --- | --- | --- |
| `device` | Yes | Device name |
| `pin_name` | Yes | Device-side signal name |
| `assigned_gpio` | Yes | Final MCU GPIO or power/ground |
| `decision_type` | Yes | Decision type enum |
| `source` | Yes | `board_default`, `auto_assigned`, `user_wiring`, `onboard_peripheral`, `fixed_power` |
| `evidence` | Yes | Structured evidence from board JSON or user wiring |
| `requires_user_review` | Yes | Whether the user is advised to focus on confirming this in `pin_plan_review`; V0 does not require precise coverage of every risky pin |
| `review_prompt` | Optional | Prompt for the user to check the schematic/silkscreen/module documentation |
| `deviation` | Optional | Structured explanation of deviation from default or occupied release |

`decision_type` enum:

```text
use_default_bus
auto_assign_free_gpio
remap_default_conflict
avoid_restricted_gpio
avoid_onboard_occupied
reuse_onboard_peripheral
fixed_power_tie
user_wiring
manual_review_required
```

`deviation` fields:

| Field | Required | Meaning |
| --- | --- | --- |
| `from_gpio` | Yes | Original default/candidate GPIO |
| `to_gpio` | Yes | Remapped GPIO or power/ground |
| `reason_code` | Yes | Deviation reason enum |
| `evidence_path` | Yes | Evidence path in board JSON, e.g., `pin_layout.default_bus_pins.i2s.data_out` |
| `evidence_value` | Yes | Value of the evidence field |
| `validator_action` | Yes | `error`, `warning`, `manual_review` |

`reason_code` enum:

```text
restricted_gpio
default_bus_conflict
onboard_occupied
not_exposed
user_requested
fixed_power_tie
insufficient_board_data
```

If `reason_code=onboard_occupied`, `evidence_path` must point to `onboard_peripherals[].occupied_pins`, and `evidence_value` must match `from_gpio`; otherwise, it should be treated as `pin_decision_invalid` or enter `manual_review_required`, and cannot rely on LLM notes to self-assert that a GPIO is occupied by an onboard device.

Power/ground connections must be recorded by actual rail: when `pinout.gpio` is `GND`, `3V3`, `5V`, `pinout.type` must be `gnd`, `power_3v3`, `power_5v` respectively; do not disguise `VDD`, `3V3`, `GND` as regular MCU GPIOs.

`fixed_power_tie` only indicates that a device-side pin is fixed to power or ground. Regular power/ground pins (e.g., `VCC`, `VDD`, `VDDIO`, `VIN`, `VBUS`, `GND`) connected to `3V3/GND` are normal power connections; configuration, mode, enable, address, gain, startup, and other control pins (e.g., `ADDR`, `BOOT`, `CFG`, `CONFIG`, `EN`, `GAIN`, `MODE`, `SEL`) fixed to `3V3/GND/5V` must use `decision_type=fixed_power_tie`, `source=fixed_power`, and it is recommended to provide a `review_prompt` for the user to check the module documentation.

V0 does not treat `requires_user_review` as a complex policy engine. The script only validates field types, enums, pinout correspondence, hard-disabled pins, conflicts, and power/ground types; risks like conditionally usable GPIOs, hardwired configuration pins, default bus deviations, and insufficient board data are uniformly placed in warnings/notes and handled via the overall `pin_plan_review` for user confirmation or pin changes.

## select-hw Draft Schema

`select_hw_manifest.py` only supports the new draft schema, not the old `update_manifest.py` input shape.

```json
{
  "protocol_version": "1.0",
  "session_id": "uuid",
  "source_phase": "analyze",
  "upstream_manifest": {},
  "selected_board": {},
  "hardware_plan": {
    "mcu": {},
    "pinout": [],
    "pin_decisions": [],
    "pin_review": {},
    "bom": [],
    "estimated_total_yuan": 0
  },
  "warnings": [],
  "metadata": {
    "idempotency_key": "select-hw:<session_id>:manifest-validation:v1"
  }
}
```

Draft field meanings:

| Field | Required | Meaning |
| --- | --- | --- |
| `protocol_version` | Yes | Currently fixed to `"1.0"` |
| `session_id` | Yes | Current workflow session ID |
| `source_phase` | Yes | Fixed to `"analyze"` |
| `upstream_manifest` | Yes | From `phase_complete(analyze).payload.manifest_content` |
| `selected_board` | Yes | Summary of the board object confirmed from the board library |
| `hardware_plan.mcu` | Yes | MCU, firmware entry point, and flashing tool |
| `hardware_plan.pinout` | Yes | Pin assignment array |
| `hardware_plan.pin_decisions` | Yes | Structured decisions and evidence for each pin selection; script must validate and retain in the final `manifest_content` |
| `hardware_plan.pin_review` | Yes | User `pin_plan_review` confirmation status; `confirmed=true` is required before `success` |
| `hardware_plan.bom` | Yes | BOM array |
| `hardware_plan.estimated_total_yuan` | Recommended | BOM total price; if missing, the script calculates from BOM and issues a warning |
| `warnings` | Recommended | Non-blocking risks |
| `metadata.idempotency_key` | Recommended | Idempotency key for manifest validation action |

## Output manifest_content

The output must retain the core analyze fields and add:

```text
phase = "select-hw"
mcu
hardware_selection
pinout
bom
estimated_total_yuan
final_status = "hardware_selected"
```

`mcu.flash_tool` enum:

```text
esptool.py
uf2-drag-drop
dfu-util
teensy-loader
unknown
```

BOM prices in V0 temporarily accept LLM common-sense estimates, not connected to store data sources.

## phase_complete

`phase_complete.select_hw.json` must be consistent with analyze and use the complete envelope.

On success:

```text
payload.result = "success"
payload.next_phase = "upy-flash-mpy-firmware-plugin"
payload.manifest_content.phase = "select-hw"
```

Success prerequisites:

- The board selection has not crossed the upstream MCU/chip family/firmware target boundary; if a cross-boundary change occurred, it must return to analyze or start a new conversation.
- `pin_plan_review` has been confirmed, or `pre_selected_board`/plugin UI has explicitly provided confirmed structured wiring.
- All items in `pin_decisions` with `validator_action=error` or `manual_review` have been resolved.

result enum:

| result | Meaning | next_phase | checkpoint |
| --- | --- | --- | --- |
| `success` | MCU/firmware/pinout/BOM all complete | `upy-flash-mpy-firmware-plugin` | Not required |
| `partial` | Recoverable interruption | `null` | Required |
| `failed` | Invalid input or protocol output | `null` | Optional |

## checkpoint/resume

partial must include a checkpoint:

```json
{
  "checkpoint_id": "uuid",
  "resume_phase": "select-hw",
  "resume_step": "board_select",
  "resume_label": "Continue selecting MicroPython development board",
  "reason": "user_cancelled",
  "state_ref": {
    "artifact": "select_hw_draft.json"
  }
}
```

`resume_step` enum:

```text
load_upstream_manifest
board_select
firmware_check
pin_assignment
bom_generation
manifest_validation
phase_complete_validation
```

`reason` enum:

```text
user_cancelled
board_change_requires_analyze
missing_pin_layout
firmware_unknown
pin_conflict
pin_plan_review_rejected
script_failed
timeout
permission_denied
```

## retry / timeout / idempotency

- Retry must use the same `session_id`.
- When retrying the same local action, `idempotency_key` remains unchanged.
- `retry_of` points to the `msg_id` of the original failed message.
- Each message that requires waiting for an external action must define `timeout_ms`.
- `on_timeout` enum: `retry_once / partial_checkpoint / failed`.

## structured_errors

Retain `errors: string[]`, and support:

```json
{
  "code": "missing_pin_layout",
  "message": "selected board lacks pin_layout",
  "severity": "error",
  "recoverable": true,
  "retryable": false,
  "source": "select_hw_manifest.py",
  "field": "mcu.board_id"
}
```

`severity` enum:

```text
info
warning
error
fatal
```

Suggested `code` enum:

```text
invalid_upstream_manifest
missing_required_field
invalid_enum
board_not_found
firmware_unknown
missing_pin_layout
pin_conflict
i2c_address_conflict
board_definition_not_found
board_definition_invalid
board_change_requires_analyze
restricted_gpio_used
default_bus_pin_deviation
pin_review_required
pin_review_rejected
pin_decision_invalid
onboard_peripheral_pin_used
onboard_peripheral_reused
user_wiring_invalid
occupied_pin_conflict
artifact_missing
absolute_path_in_artifact
permission_denied
script_failed
timeout
phase_complete_invalid
```

## artifact/file manifest

`phase_complete.payload.artifacts` must be an array. `file_list.files[].path` must be a path relative to the artifact root.

`artifact.type` enum:

```text
table
file_tree
markdown
html
code_diff
file_list
```

`file_list.files[].status` enum:

```text
created
updated
unchanged
skipped
error
```

Direct test formal artifacts:

```text
select_hw_draft.json
select_hw_validated.json
phase_complete.select_hw.json
pin_assignment_log.md
select_hw_phase_log.md
```

During direct testing, the `file_list` in `phase_complete.payload.artifacts` must declare all the above files, and `--validate-phase-complete` must use `--expected-artifact` to validate each one. Missing any formal artifact declaration is considered a failure.

### Log Template Rules

`pin_assignment_log.md` must list GPIOs in the following groups:

```text
## GPIO Usage Summary

Used GPIO: GPIO4, GPIO5, GPIO6, GPIO7, GPIO10, GPIO11, GPIO20, GPIO21
Unused GPIO: GPIO0, GPIO1, GPIO2, GPIO3, GPIO8, GPIO9, GPIO12, GPIO13, GPIO18, GPIO19
Conditional/Reserved GPIO: GPIO2, GPIO8, GPIO9 (strapping boot pins)
Forbidden GPIO: (none)

## Pin Assignment Details
...
```

Ambiguous names like "unused (idle)" are forbidden. `select_hw_phase_log.md` must record the complete step timeline, `runtime_context` parameters, and path conventions.

## Permission Prompts

V0 allows low-risk actions:

- Reading upstream phase_complete files
- Reading `upy-analyze-plugin/boards` from `resource_root`
- Writing `sessions/<session_id>/select_hw_*.json`
- Running the whitelisted script `upy-select-hw-plugin/scripts/select_hw_manifest.py` from `resource_root`

Actions requiring a separate permission prompt:

- Any non-whitelisted script
- Copying `upy-select-hw-plugin`, `upy-analyze-plugin`, or partial skill/resource copies into `artifact_root`
- Copying only a single board JSON as the candidate board library
- Deleting files
- Accessing device serial ports
- Flashing firmware
- Connecting to the internet to check store prices

## Script Validation

Must use:

```text
upy-select-hw-plugin/scripts/select_hw_manifest.py
```

It is a validator/normalizer, not a default write-to-disk script.

Must support:

```text
--stdin
--input <path>
--write-path <path>
--validate-manifest-content
--validate-phase-complete
--compare-manifest <path>
--artifact-root <path>
--board-root <path>
--strict-board-pins
--expected-artifact <relative-path>
```

Must validate:

- Draft schema only accepts the new format
- Upstream manifest meets at least the analyze minimum delivery fields
- MCU, pinout, BOM required fields are complete
- Enum values are valid
- Pinout conflicts
- `pin_decisions` fields, enums, evidence, and deviations are valid and correspond to the final `pinout`
- `pin_review.approval_id=pin_plan_review`; `pin_review.confirmed=true` when `result=success`
- phase_complete envelope is valid
- `manifest_content` core fields are consistent with the compare manifest, and `pin_decisions` / `pin_review` are not lost
- Declared relative paths in file artifacts actually exist
- `selected_board` is consistent with the full board JSON
- `pinout` adheres to the board JSON's `restricted_gpio`
- `pinout` adheres to the board JSON's `onboard_peripherals[].occupied_pins`
- Three sources (user wiring, onboard device reuse, external device auto-assignment) are distinguishable
- Bus pin deviations from `pin_layout.default_bus_pins` must have notes/warnings
- `phase_complete.payload.artifacts` covers all formal artifacts written by this phase
- WiFi + `adc2_wifi_conflict` digital usage must generate a complete warning, not just hint at some GPIOs

Formatted output validation flow:

```text
python upy-select-hw-plugin/scripts/select_hw_manifest.py --input upy-select-hw-plugin/sample/select_hw_draft.json --write-path <artifact-root>/select_hw_validated.json --board-root upy-analyze-plugin/boards
python upy-select-hw-plugin/scripts/select_hw_manifest.py --validate-manifest-content --input <artifact-root>/select_hw_validated.json --board-root upy-analyze-plugin/boards
```

The second command validates that the script output still conforms to the normalized `manifest_content` schema; the formal phase completion still requires `--validate-phase-complete` to validate `phase_complete.select_hw.json`.

## Local Testing

Subsequent tests must cover:

1. Start from `payload.manifest_content` in `G:\test\test\sessions\022ad742-3269-42e9-ac20-c14f477ecdf2\phase_complete.analyze.json`, treating `G:\test\test` as `artifact_root`.
2. Use the complete board library from `resource_root/upy-analyze-plugin/boards` to match `ESP32-C3` candidate boards, without creating `upy-analyze-plugin` or `upy-select-hw-plugin` copies under `G:\test\test`.
3. Trigger `approval_request(board_select)` when `mcu_specified` exists but `pre_selected_board` does not.
4. Skip board_select when `pre_selected_board` comes from the plugin UI.
5. When pin_layout is missing, swap to a known board with similar functionality and pin_layout.
6. `cold-driver` does not block MCU recommendation and pinout, but must output `driver.status="cold_driver_required"`; a `ready` status with hardware verification and `driver.path` and `hardware_marker` can be retained.
7. When no MCU is specified, prioritize recommending Pico/RP2 and ESP32 series.
8. When the user-specified board is not in the board library, send `approval_request(board_unavailable)`, providing four options: similar board, select known board, manual wiring description, save checkpoint.
9. The formatted manifest generated by `select_hw_manifest.py --write-path` can be read and validated again by the script.
10. `phase_complete.select_hw.json` passes script validation, and `--expected-artifact` covers all direct test formal artifacts.
11. Validator covers board-root, restricted pins, default bus deviations, user wiring, onboard device reuse, ADC2/WiFi digital usage warnings.
12. `phase_complete.payload.artifacts` covers all formal artifacts, and logs/artifacts do not contain absolute paths to the local plugin installation directory.
13. `approval_response.payload.user_pin_constraints` from `pin_plan_review` can be converted to `user_wiring` pinout/pin_decisions.
14. When the user specifies an illegal GPIO, it must not silently auto-swap pins; must output partial/checkpoint or validation failure.

## Maintenance Principles

Subsequent updates will be based on the contents of the `upy-select-hw-plugin` directory, and course documentation will be updated retroactively.
## Session Boundary Addendum

- Treat `runtime_context.session_root` and explicit upstream `source_phase_complete_path` as the `workflow_session_root`.
- A log-only or diagnostic session may be referenced for evidence, but it must not relocate select-hw artifacts or change the workflow session.
- Write `select_hw_draft.json`, `select_hw_validated.json`, `phase_complete.select_hw.json`, logs, and artifact paths under the active `workflow_session_root`.
- Keep `resource_root`, `artifact_root`, and `workflow_session_root` separate. Do not copy skill resources into the artifact workspace to make paths appear valid.
- If session evidence is mixed, record the secondary session as `diagnostic_log_session` in warnings/artifacts and continue writing to the workflow session.
