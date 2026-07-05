# upy-select-hw-plugin Common Issues and Improvement Plan

This document organizes common improvement suggestions for `upy-select-hw-plugin`. The goal is to make `select-hw` a maintainable, long-term hardware selection workflow protocol, rather than an ad-hoc implementation dependent on a single project, board, or component case.

## Conclusion

The current problem is not simply "the board library is not read." `SKILL.md` already mentions reusing `upy-analyze-plugin/boards`, but it does not define the complete board JSON as the hardware source of truth, nor does it make fields like `pin_layout.default_bus_pins`, `restricted_gpio`, and `onboard_peripherals` strong constraints for pin assignment and validation.

Consequently, an executor might assign pins based only on the board name or MCU common knowledge, without strictly adhering to the default buses, restricted pins, and onboard resource definitions in the board JSON.

## Issues Found

### 1. Board Library is a "Reference," Not the "Source of Truth"

Current Status:

- `SKILL.md` mentions reading `upy-analyze-plugin/boards`.
- However, there is no mandate to load `upy-analyze-plugin/boards/<board_id>.json` after the user confirms the board.
- The `selected_board` summary might be misused as the complete board truth.

Risk:

- Information such as firmware, default buses, restricted pins, and onboard peripherals may be ignored.
- Subsequent pinout may be inconsistent with the actual board definition.

Solution:

- Add a strong constraint in the "Board Data" section: the complete board JSON must be loaded after board confirmation.
- The complete board JSON is the source of truth for pinout, firmware, BOM main controller entries, and warnings.
- If the board JSON does not exist, the process must enter `board_unavailable` and cannot continue claiming success.

### 2. Pin Assignment Does Not Mandate Priority Use of Default Buses

Current Status:

- `SKILL.md` only states to assign pins according to `pin_layout`.
- It does not specify that I2C/SPI/UART/I2S should prioritize using `pin_layout.default_bus_pins`.
- There is no requirement to explain the reason when deviating from default pins.

Risk:

- A situation may arise where "the board library has a default I2C, but the output uses different pins."
- The downstream code generation phase might produce initialization code inconsistent with the board definition.

Solution:

- Add to "Pin Assignment Rules": bus-type interfaces should use `pin_layout.default_bus_pins` by default.
- Deviating from default pins must satisfy two conditions: it does not hit prohibited/high-risk pins; the reason is documented in `pinout[].notes` and warnings.
- If a default bus pin is occupied by another necessary function, the conflict decision must be recorded.

### 3. Ambiguous Semantics of `restricted_gpio` Levels

Current Status:

- The current rule only vaguely states "avoid boot/strapping, flash/PSRAM, USB OTG, read-only pins."
- It does not distinguish between levels such as prohibited, prohibited by default, usable but requires explanation, or only effective for specific pin types.

Risk:

- Strapping pins might be treated as ordinary available GPIOs.
- ADC/WiFi conflict pins might be incorrectly blanket-prohibited, or missed in ADC scenarios.
- USB serial pins might affect subsequent flashing, REPL, or debugging.

Solution:

Define a general risk classification in `SKILL.md`:

| board field | Default Strategy | Validation Level |
| --- | --- | --- |
| `flash_psram_occupied` | Prohibited | error |
| `reserved` / `internal_only` | Prohibited | error |
| `usb_serial_pins` | Prohibited by default, unless USB serial is explicitly not used | error or warning |
| `strapping` / `boot` | Avoid by default; user confirmation or strong warning required if used | warning or partial |
| `input_only` | Can only be used for input-type pin types | error |
| `adc_only` | Can only be used for ADC input | error |
| `adc2_wifi_conflict` | Conflicts only when `type=adc` and WiFi is enabled; digital input/output is usable but should be explained | error or warning |
| `onboard_peripherals[].occupied_pins` | Prohibited when `always_used=true`; otherwise avoid by default or explain release reason | error or warning |

### 4. Missing Rules for User-Provided Board Pins and Onboard Device Reuse

Current Status:

- The user may have already provided information about a board, onboard devices, or pin connections.
- The board JSON may also declare `onboard_peripherals` and their `occupied_pins`.
- The current rules do not clearly state: when an onboard device matches a user-specified or system-recommended device, the board's default pins should be reused; when they do not match, the onboard occupied pins should be excluded, and free pins should be assigned to external devices.

Risk:

- The same onboard device might be duplicated as an external device, causing BOM and pinout duplication.
- An external device might mistakenly use GPIOs occupied by an onboard device.
- User-provided wiring might be overwritten by automatic assignment.

Solution:

- Add "User/Board Pin Fact Priority":
  1. User-provided wiring or plugin UI selections have the highest priority but must pass board restricted validation.
  2. If the board JSON's `onboard_peripherals` is functionally equivalent to the user-specified or system-recommended device, prioritize reusing the onboard device and its default/occupied pins, and do not add it again to the external BOM.
  3. If the onboard device is irrelevant to the current project, or the user specifies an external device, then `onboard_peripherals[].occupied_pins` should be considered occupied resources, and the external device must use free pins.
  4. If the user requests to release pins occupied by an onboard device, it must be confirmed that `always_used=false`, and the release reason must be documented in notes/warnings.

- Add device matching logic:
  - Match onboard devices with required devices by `type`, `interface`, `name` aliases, and functional tags.
  - On successful match, `pinout[].source` should be marked as `onboard_peripheral`.
  - For external devices, `pinout[].source` should be marked as `external_device` or `user_wiring`.

### 5. Validator Lacks Board Semantic Validation

Current Status:

- `select_hw_manifest.py` validates field completeness, enum values, GPIO duplicates, power/GND, and I2C address conflicts.
- It does not load the board JSON, nor does it validate `default_bus_pins`, `restricted_gpio`, or `onboard_peripherals`.

Risk:

- A solution that is schema-valid but hardware-infeasible will pass.
- The session output might show `status: ok`, but the actual wiring risk is high.

Solution:

- Add a `--board-root <path>` parameter, defaulting to the relative path `upy-analyze-plugin/boards`.
- Load the board JSON based on `hardware_plan.mcu.board_id`.
- Add new validations:
  - Board file exists.
  - `selected_board.firmware` matches the board JSON.
  - Board JSON has a `pin_layout`.
  - User-provided wiring does not override prohibited pins.
  - When an onboard device matches, the occupied/default pins declared in the board JSON must be reused.
  - External devices cannot occupy `occupied_pins` of unrelated onboard devices.
  - Pinout does not use prohibited pins.
  - Generate structured warning/error when pinout hits high-risk pins.
  - Bus pin deviations from `default_bus_pins` must have `notes`.

### 6. Incomplete Artifact/File Manifest

Current Status:

- The directly tested recommended artifacts in `SKILL.md` include `select_hw_draft.json`, `select_hw_validated.json`, `phase_complete.select_hw.json`, and `pin_assignment_log.md`.
- If a phase log or other artifacts are also generated, `phase_complete.payload.artifacts[].files` may not fully declare them.

Risk:

- Downstream phases or debugging tools cannot fully discover the artifacts.
- `phase_complete` may be inconsistent with the actual files in the session directory.

Solution:

- Require in `SKILL.md`: `phase_complete.payload.artifacts` must cover all formal artifacts written by this phase.
- The directly tested recommended artifacts should include:
  - `select_hw_draft.json`
  - `select_hw_validated.json`
  - `phase_complete.select_hw.json`
  - `pin_assignment_log.md`
  - `select_hw_phase_log.md`

### 7. Insufficiently Strict Log Path Conventions

Current Status:

- `SKILL.md` requires the protocol and examples to use relative paths.
- However, it does not explicitly require phase logs, command history, and artifact descriptions to also use relative paths.

Risk:

- Logs might contain local absolute paths, such as plugin installation directories or user directories.
- Readability and reproducibility decrease when the same session is migrated to another machine.

Solution:

- Add to "Relative Path Convention": logs, command history, and artifact descriptions must also use relative paths.
- Allow test reports to state the "local execution path," but it cannot serve as the business source of truth.
- Recommend unified recording of:
  - `upy-analyze-plugin/boards`
  - `upy-select-hw-plugin/scripts/select_hw_manifest.py`
  - `sessions/<session_id>/<artifact>`

### 8. Session Artifact Corrections Should Not Be Manual Local Patches

Current Status:

- If only a local part of a JSON or log is modified, it is easy to cause inconsistencies between draft, validated, phase_complete, and logs.

Risk:

- `compare-manifest` may fail.
- The log seen by the user may be inconsistent with the manifest read by the machine.

Solution:

- After modifying the pinout or board selection, rebuild artifacts in a fixed order:
  1. Update `select_hw_draft.json`
  2. Use the validator to generate `select_hw_validated.json`
  3. Generate `phase_complete.select_hw.json` from the validated manifest
  4. Update `pin_assignment_log.md`
  5. Update `select_hw_phase_log.md`
  6. Re-run the phase_complete validation

## Suggested Modification Locations in `SKILL.md`

### "Standard Message Sequence"

It is recommended to add an explicit step after `board_select`:

```text
Step 1B Load Complete Board Definition
  -> status_update(board_definition_loaded)
  Load the complete board JSON from upy-analyze-plugin/boards/<selected_board.id>.json
  If it does not exist or lacks pin_layout:
    -> approval_request(board_unavailable or board_select)
```

### "status_update Enumeration"

It is recommended to add:

```text
board_definition_loaded
board_definition_invalid
pin_risk_detected
```

### "Board Data"

It is recommended to add these strong constraints:

- `selected_board.id` must correspond to `upy-analyze-plugin/boards/<id>.json`.
- The complete board JSON is the source of truth for `firmware`, `pin_layout`, `restricted_gpio`, and `onboard_peripherals`.
- Pin assignment based solely on the MCU name or board summary is not allowed.
- The phase cannot be completed successfully if the board JSON or `pin_layout` is missing.

### "Pin Assignment Rules"

It is recommended to add these strong constraints:

- Bus interfaces should prioritize using `pin_layout.default_bus_pins`.
- If the board JSON has `pin_options`, remapping is only allowed within the permitted range of `pin_options`.
- If the board JSON is a flexible matrix, `restricted_gpio` must still be avoided.
- Deviations from default buses must be explained.
- Hitting high-risk pins must result in warnings or partial status, not be written as "safe."
- When user-provided wiring exists, prioritize preserving it, but it must pass the restricted/occupied validation of the board JSON.
- When an onboard device matches the required device, reuse the onboard default pins; do not allocate external GPIOs or add it again to the BOM.
- When an onboard device does not match the required device, the onboard occupied pins are considered occupied resources; recommended/user external devices can only use free pins.

### "Script Validation"

It is recommended to add:

```text
--board-root <path>
--strict-board-pins
```

And require validation of:

- Board JSON exists.
- `selected_board` matches the board JSON.
- Pinout adheres to `restricted_gpio`.
- Pinout adheres to `onboard_peripherals[].occupied_pins`.
- The three sources (user wiring, onboard device reuse, external device auto-assignment) must be distinguishable.
- Pinout deviations from default buses have notes.
- `phase_complete.payload.artifacts` covers all formal artifacts.

## Validator Modification Suggestions

It is recommended to add general-purpose functions:

- `load_board_definition(board_root, board_id)`
- `validate_selected_board_against_definition(selected_board, board_definition)`
- `match_onboard_peripherals(devices, board_definition)`
- `validate_pinout_against_board(pinout, board_definition, requirements)`
- `validate_user_wiring_against_board(user_wiring, board_definition)`
- `validate_artifact_completeness(artifacts, expected_files)`

It is recommended to add structured error/warning codes:

```text
board_definition_not_found
board_definition_invalid
restricted_gpio_used
default_bus_pin_deviation
onboard_peripheral_pin_used
onboard_peripheral_reused
user_wiring_invalid
occupied_pin_conflict
artifact_missing
absolute_path_in_artifact
```

## General Acceptance Criteria

After the modifications are complete, at least the following should be satisfied:

1. For any known board, the complete board JSON is loaded before pin assignment.
2. The default bus priority rule can be reproduced in tests.
3. Using flash/PSRAM occupied pins will fail.
4. Using strapping/boot pins will not silently succeed; at least a structured warning is generated, and partial status is used when necessary.
5. ADC2/WiFi conflicts only apply to ADC scenarios and do not incorrectly affect ordinary digital input/output.
6. User-provided wiring is preserved and validated; illegal wiring does not pass silently.
7. When an onboard device matches the requirement, the onboard pins are reused, and it is not added again to the external BOM.
8. When an onboard device does not match the requirement, external devices only use free pins.
9. The `file_list` in `phase_complete` covers all formal artifacts.
10. Logs and artifacts do not contain the local absolute path of the plugin installation.
11. The content of draft, validated, phase_complete, and logs is consistent.

## Suggested Implementation Order

1. First, modify `SKILL.md` to write the board library source of truth, default buses, restricted_gpio, and artifact completeness as general strong constraints.
2. Then, enhance `select_hw_manifest.py` to add board-root semantic validation.
3. Update samples and smoke tests to cover default buses, restricted pins, unknown boards, user-provided wiring, onboard device reuse, artifact completeness, and relative paths.
4. Finally, regenerate existing session artifacts to avoid manual local patches.
