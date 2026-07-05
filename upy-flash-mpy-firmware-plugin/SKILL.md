---
name: upy-flash-mpy-firmware-plugin
description: Plugin-based workflow for MicroPython firmware parsing, downloading, flashing, or manual confirmation. Used when Codex receives a phase_complete(select-hw) with next_phase=upy-flash-mpy-firmware-plugin; consumes the select-hw manifest_content, parses the latest firmware from micropython.org/download, assists with ESP32 esptool flashing, guides Pico UF2 copying, or provides manual flash links for other boards, and finally outputs next_phase=upy-scaffold-plugin.
---

# MicroPython Firmware Flashing Phase

## Role

`upy-flash-mpy-firmware-plugin` is the plugin phase after `select-hw`. It only consumes `phase_complete.select_hw.json`, does not re-analyze requirements, does not re-select boards, and does not generate business code.

Input fact source:

```text
sessions/<session_id>/phase_complete.select_hw.json
```

Successful output:

```text
phase_complete(payload.phase="upy-flash-mpy-firmware-plugin", payload.next_phase="upy-scaffold-plugin")
```

## Board Branches

| Branch | Condition | Behavior |
| --- | --- | --- |
| ESP32 | `firmware_board_name` starts with `ESP32_`, `firmware.port == "esp32"`, or `chip_family` starts with `esp32` | Parse the latest `.bin`, parse the install command from the MicroPython board page, scan/select a real serial port, and run `esp32_flash.py` only after user confirmation. |
| Pico | `firmware_board_name` starts with `RPI_PICO` | Parse the latest `.uf2`, prompt the user to hold BOOTSEL and copy the UF2, then wait for user confirmation. |
| Manual | Other MicroPython boards | Parse the MicroPython download/install link and display manual flashing instructions. Do not execute `dfu-util`, `teensy-loader`, ESP8266 esptool, or other tools; only wait for user confirmation. |

Only mock/sample tests may use a fixed `serial_port="COM3"` to validate JSON and command planning. Claude Code live use and real plugin use must scan real serial ports and require user selection.

## Input Contract

Recommended `start_phase` payload:

```json
{
  "protocol_version": "1.0",
  "msg_id": "uuid",
  "session_id": "<session_id>",
  "phase": "upy-flash-mpy-firmware-plugin",
  "timestamp": "<runtime-utc-now>",
  "type": "start_phase",
  "idempotency_key": "upy-flash-mpy-firmware-plugin:<session_id>:start:v1",
  "retry_of": null,
  "payload": {
    "phase": "upy-flash-mpy-firmware-plugin",
    "source_phase": "select-hw",
    "source_phase_complete_path": "sessions/<session_id>/phase_complete.select_hw.json",
    "runtime_context": {
      "artifact_root": ".",
      "artifact_root_mode": "cwd",
      "session_root": "sessions/<session_id>",
      "resource_root": "<runtime-provided>"
    },
    "capabilities": {
      "protocol_versions": ["1.0"],
      "approval_request": true,
      "script_run": true,
      "file_operation": true,
      "network_access": {
        "allowed": true,
        "domains": ["micropython.org", "docs.micropython.org"]
      },
      "web_search": true,
      "serial_port_scan": true,
      "device_flash": true,
      "relative_paths": true,
      "artifact_root": true
    },
    "firmware_action": null,
    "firmware_override": null
  }
}
```

Field rules:

| Field | Required | Meaning |
| --- | --- | --- |
| `source_phase_complete_path` | Required for file mode | Relative path to the upstream `phase_complete.select_hw.json`. |
| `payload.source_phase_complete` | Optional | Complete upstream message envelope; used when the host passes JSON directly instead of a path. |
| `runtime_context.artifact_root` | Yes | Artifact root directory for resolving relative artifact paths. |
| `runtime_context.artifact_root_mode` | Yes | `cwd` or `session_root`; do not mix path scopes within the same `phase_complete`. |
| `runtime_context.session_root` | Yes | Relative session directory, usually `sessions/<session_id>`. |
| `runtime_context.resource_root` | Yes | Root directory for installed skills/resources; use it to locate this skill's scripts. |
| `capabilities.script_run` | Yes | The host can run whitelisted skill scripts. |
| `capabilities.network_access` | Yes | The host can access MicroPython download pages. |
| `capabilities.serial_port_scan` | Required for real ESP32 flashing | The host can enumerate serial ports. |
| `capabilities.device_flash` | Required for real ESP32 flashing | The host allows erase/write after user confirmation. |
| `firmware_action` | Optional | `download_and_flash`, `download_only`, `already_flashed`, `use_local_firmware`, `save_partial`, or `cancel`. |
| `firmware_override` | Optional | User-provided `local_path`, `url`, `file_type`, and `source`. |

Must validate the upstream message envelope:

```text
protocol_version == "1.0"
type == "phase_complete"
phase == "select-hw"
payload.phase == "select-hw"
payload.result == "success"
payload.next_phase == "upy-flash-mpy-firmware-plugin"
payload.manifest_content.phase == "select-hw"
```

Only local tests during the migration period may accept the legacy value `payload.next_phase == "flash-mpy-firmware"` when `--allow-legacy-next-phase` is explicitly passed; formal output must not use the legacy value.

## Board Facts

Only read board facts from `phase_complete.select_hw.json.payload.manifest_content`.

Field priority:

| Value | Preferred Source | Fallback Source |
| --- | --- | --- |
| Firmware URL | `hardware_selection.selected_board.firmware.url` | `mcu.firmware_url`; if both are missing, use the firmware board name to match the real download page on the MicroPython download index. |
| Firmware board name | `hardware_selection.selected_board.firmware.board_name` | `mcu.firmware_board_name` |
| Firmware port | `hardware_selection.selected_board.firmware.port` | Infer from the board name |
| Chip family | `hardware_selection.selected_board.chip_family` | `mcu.chip_family` |
| Flash tool hint | `mcu.flash_tool` | Infer from the board family |
| Display name | `hardware_selection.selected_board.display_name` | `mcu.display_name` |

Do not trust a cached `latest_version`. At runtime, must first use the upstream `hardware_selection.selected_board.firmware.url`, then `mcu.firmware_url`, to access the official MicroPython board page and parse the real `(latest)` firmware and installation instructions. Only when the upstream URL is missing or invalid, use `firmware_board_name` to match the real download page slug from `https://micropython.org/download/` homepage. Do not construct the URL directly from `display_name`, `board_id`, or MCU model.

Firmware page related field meanings:

| Field | Purpose |
| --- | --- |
| `firmware.url` / `mcu.firmware_url` | MicroPython official firmware page URL, normal main path. |
| `firmware.board_name` / `mcu.firmware_board_name` | MicroPython firmware board name, used for display and fallback matching when URL is missing. |
| `display_name` | Board name shown to the user, not used for constructing download URLs. |
| `board_id` | Local board library ID, not used for constructing download URLs. |
| `download_slug` | Real MicroPython download page slug extracted from the firmware URL path or matched from the download homepage. |
| `board_url` | Normalized MicroPython board page URL. |

## JSON Output Language Convention

Explanation paragraphs, rule explanations, and field meaning tables in `SKILL.md` should use Chinese as much as possible; JSON examples must maintain a mixed English/Chinese format consistent with `upy-analyze-plugin`, `upy-select-hw-plugin` samples and real session artifacts.

- JSON keys, protocol fields, enum values, action values, error codes, file names, script parameters, and paths remain in English or as-is, e.g., `payload`, `result`, `download_and_flash`, `firmware_action_select`, `missing_firmware_url`.
- User-visible UI text uses Chinese, e.g., `header`, `question`, `actions[].label`, `steps[]`, `links[].label`.
- Project semantic text prefers Chinese, e.g., `summary`, `description`, `message`, `warnings[]`, `notes`, `manual_flash_instructions` displayed to the user.
- Machine classifications and sources remain in English, e.g., `source`, `status`, `action`, `file_type`, `flash_method`, `reason`, `structured_errors[].code`.
- The upstream `source_phase_complete.payload.manifest_content` must preserve the language output of the `select-hw` phase as-is; do not translate project names, device names, driver package names, API names, or user input for the sake of language consistency.
- `errors[]` can retain the original English error from the script/validator; `structured_errors[].message`, if generated by the plugin/LLM for the user, prefers Chinese; if directly passing through the original script error, English can be retained.

## Workflow

1. Send `status_update(upstream_select_hw_loaded)`.
2. Load and validate the upstream `phase_complete.select_hw.json`.
3. Send `status_update(firmware_board_resolved)`.
4. If `firmware_action` is missing, send `approval_request(firmware_action_select)` and wait for user input.
5. If the user selects `already_flashed`, output `success` and set `firmware.status="skipped_user_confirmed"`.
6. If the user selects `save_partial`, times out, or cancels, output `partial` with a checkpoint.
7. Use `scripts/firmware_page_resolve.py` to parse the MicroPython board page; normally pass `--board-url` from the upstream firmware URL; only use `--download-index-url` and `--board-name` for fallback matching when the URL is missing.
8. Unless `firmware_override.local_path` is provided or the branch is manual-only, download the firmware using `scripts/firmware_download.py`.
9. Enter the selected board branch.
10. Use `scripts/flash_mpy_firmware_manifest.py` to validate the phase output.
11. Output the final `phase_complete`.

## approval_request: firmware_action_select

Unless `start_phase.payload.firmware_action` already exists, this approval must be sent before any download or flash action.

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "firmware_action_select",
    "header": "MicroPython Firmware Preparation",
    "question": "Please select the action to perform for this firmware phase",
    "summary": {
      "board_name": "ESP32_GENERIC_C3",
      "display_name": "ESP32-C3-DevKitM-1",
      "firmware_page": "https://micropython.org/download/ESP32_GENERIC_C3/"
    },
    "actions": [
      {"label": "Download and Flash", "value": "download_and_flash", "primary": true},
      {"label": "Download Only", "value": "download_only"},
      {"label": "Already Flashed, Skip", "value": "already_flashed"},
      {"label": "Use Local Firmware File", "value": "use_local_firmware"},
      {"label": "Continue Later", "value": "save_partial"},
      {"label": "Cancel", "value": "cancel"}
    ]
  }
}
```

## Claude Code Local Run Notes

The `approval_request.actions` in the plugin protocol can retain the full action set; however, Claude Code's `AskUserQuestion` can only pass a maximum of 4 `options` per question. When running locally in Claude Code, do not map the 6 actions above directly into one `AskUserQuestion`.

When running `firmware_action_select` locally, prioritize displaying the 4 main actions:

```text
download_and_flash
download_only
already_flashed
use_local_firmware
```

`save_partial` and `cancel` are retained in the plugin approval UI; for local Claude Code runs, they can be handled via a second confirmation, normal conversation, or subsequent checkpoint writes.

When using a temporary Python one-liner to read JSON on Windows, you must explicitly specify UTF-8, e.g., `open(path, encoding="utf-8")`, or set `PYTHONUTF8=1` in the runtime environment. Do not use the default `open(path)` to read JSON containing Chinese characters.

When calling `firmware_download.py`, you must pass `--out-dir`; `--output-json` and `--out-json` are only aliases for the output manifest parameter and cannot replace the download directory.

`phase_complete.payload.artifacts` must be written as an array, and the array must contain objects with `type="file_list"`; do not write it as a `{ "file_list": [...] }` object, nor as a flat file array.

If helper scripts retain native absolute paths for execution, they must also output portable relative artifact fields for consumption by downstream protocols:
- When `firmware_download.py` receives `--artifact-root <artifact_root>`, it outputs `downloaded_artifact_path` relative to that artifact root.
- When `esp32_flash.py` receives `--artifact-root <artifact_root>`, it outputs `firmware_artifact_path`; the artifact path for the execution log itself is declared by the final `phase_complete.payload.firmware.flash_result.log`.
- Downstream plugins can only consume relative artifact fields and the final `phase_complete`; do not read native absolute execution paths from helper JSONs as project facts.

## ESP32 Flow

Must first parse the installation instructions from the MicroPython page. Do not use hardcoded offsets as the primary source; for example, `ESP32_GENERIC_C5` currently uses `write_flash 0x2000`, so a fixed C-series offset is incorrect.

Script order:

```text
script_run(firmware_page_resolve.py --board-family esp32 --out-json ...)
script_run(firmware_download.py --resolved-json ... --out-dir ... --output-json ...)
script_run(list_serial_ports.py --output-json ...)  # Real/plugin mode
approval_request(esp32_flash_confirm)
script_run(bootstrap_esptool.py --output-json ...)  # Check; status=missing means install permission needed, not failure
script_run(bootstrap_esptool.py --install --output-json ...)  # Only run if installation is needed and permission is granted
script_run(esp32_flash.py --plan-only --output-json ...)
script_run(esp32_flash.py --execute --output-json ...)  # Only allowed to execute after explicit user confirmation
```

`approval_request(esp32_flash_confirm)` must include:

- The firmware file name and MicroPython board page.
- The erase/write commands and `write_offset` parsed from the page.
- Serial port options from real scanning in real/plugin mode.
- Download mode hint: usually hold BOOT, press EN/RESET, then release BOOT; if the board instructions differ, remind the user to follow the board instructions.
- Explicit warning: Flashing will erase and rewrite the MicroPython firmware.

Expected approval response:

```json
{
  "type": "approval_response",
  "payload": {
    "approval_id": "esp32_flash_confirm",
    "action": "flash_now",
    "serial_port": "COM3",
    "baud": 460800
  }
}
```

The `COM3` above is just a Windows example; common Linux values are like `/dev/ttyUSB0` or `/dev/ttyACM0`, and common macOS values are like `/dev/cu.usbmodem1101` or `/dev/cu.usbserial-0001`. Real usage must use a scanned and user-selected serial port; do not hardcode port names based on the operating system.

## Pico Flow

Parse and download the latest `.uf2`; do not run flash commands.

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "pico_uf2_drag_drop",
    "header": "Copy Pico UF2 Firmware",
    "question": "Please hold BOOTSEL while connecting the Pico, copy the UF2 file to the RPI-RP2 disk, and confirm when done",
    "summary": {
      "board_name": "RPI_PICO_W",
      "firmware_file": "sessions/<session_id>/firmware/<file>.uf2"
    },
    "steps": [
      "Disconnect Pico USB",
      "Hold BOOTSEL and reconnect USB",
      "Copy the UF2 file to the RPI-RP2 disk",
      "Wait for the board to restart automatically"
    ],
    "actions": [
      {"label": "Copied and Restarted", "value": "copied_uf2", "primary": true},
      {"label": "Continue Later", "value": "save_partial"},
      {"label": "Cancel", "value": "cancel"}
    ]
  }
}
```

In V0, user confirmation of `copied_uf2` is sufficient to consider it a success.

Cross-platform mount paths are only hints and optional auxiliary discovery; they do not change the V0 manual copy contract:

- Windows usually shows as a removable disk with volume label `RPI-RP2`.
- macOS is usually `/Volumes/RPI-RP2`.
- Common Linux paths are `/media/$USER/RPI-RP2`, `/run/media/$USER/RPI-RP2`, or `/mnt/RPI-RP2`.
- Optionally run `scripts/find_uf2_mount.py --output-json ...` to help locate the mount point; do not automatically fail the Pico flow because the mount point was not found, unless the user also did not confirm `copied_uf2`.

## Manual Board Flow

For non-ESP32/Pico boards, only parse the MicroPython board link and display manual instructions. Do not execute tool hints like `dfu-util`, `teensy-loader`, ESP8266 esptool, etc.

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "manual_firmware_flash_confirm",
    "header": "Please Manually Flash MicroPython Firmware",
    "question": "Please open the link below, follow the official instructions to complete the firmware flash, and click confirm when done.",
    "summary": {
      "board_name": "PYBV11",
      "firmware_page": "https://micropython.org/download/PYBV11/",
      "latest_firmware_url": "https://micropython.org/resources/firmware/<file>",
      "flash_method": "manual"
    },
    "links": [
      {"label": "MicroPython Firmware Download Page", "url": "https://micropython.org/download/PYBV11/", "source": "micropython_official"}
    ],
    "steps": [
      "Download the firmware marked as latest on the page.",
      "Put the board into firmware flashing mode according to the official instructions.",
      "Use the tool recommended on the page or by the manufacturer to complete the flash.",
      "After the device restarts, return to the plugin window and click confirm."
    ],
    "actions": [
      {"label": "Confirm Firmware Flashed", "value": "confirm_flashed", "primary": true},
      {"label": "Continue Later", "value": "save_partial"},
      {"label": "Cancel", "value": "cancel"}
    ]
  }
}
```

Manual flash approval field meanings:

| Field | Meaning |
| --- | --- |
| `approval_id` | Fixed to `manual_firmware_flash_confirm`. |
| `summary.board_name` | Upstream firmware board name. |
| `summary.download_slug` | The actual resolved MicroPython download page slug, optional. |
| `summary.firmware_page` | MicroPython official board page URL. |
| `summary.latest_firmware_url` | The main firmware link marked as latest on the page. |
| `summary.latest_version` | Latest firmware version, e.g., `v1.28.0`. |
| `summary.latest_date` | Latest firmware date, e.g., `2026-04-06`. |
| `summary.file_type` | Firmware type, e.g., `dfu`, `uf2`, `bin`, `hex`, or `zip`. |
| `summary.flash_method` | Fixed to `manual`. |
| `summary.tool_hint` | Tool or method extracted from the page instructions, e.g., `dfu-util`, `st-flash`, `uf2-drag-drop`, `teensy-loader`, `ftp-copy`, or `manual`. |
| `links[]` | Links to the download page, latest firmware, official documentation, tool documentation, etc. |
| `steps[]` | User-facing Chinese steps, derived from the official installation instructions summary. |
| `commands[]` | Optional; only display page commands, do not execute automatically; each item must be marked with `execute_allowed=false`. |
| `warnings[]` | Manual flash risk warnings. |
| `actions[]` | `confirm_flashed`, `save_partial`, `cancel`. |

## Scripts

Allowed whitelist scripts:

| Script | Purpose |
| --- | --- |
| `scripts/firmware_page_resolve.py` | Parse MicroPython download page, latest firmware URL, and installation instructions; supports `--html-file` for mock testing. |
| `scripts/firmware_download.py` | Download resolved firmware artifacts, or output a plan without downloading. |
| `scripts/list_serial_ports.py` | Enumerate serial ports for ESP32 real/plugin mode; prefer pyserial, fall back to platform-specific methods on Windows/macOS/Linux on failure. |
| `scripts/find_uf2_mount.py` | Optionally discover Pico/RP2040 UF2 mount point; only report candidate paths like `RPI-RP2`, do not automatically copy firmware. |
| `scripts/bootstrap_esptool.py` | Create/check the skill-internal `.venv-esptool` and install a pinned version of esptool after permission is granted. |
| `scripts/esptool_runner.py` | Run the skill-internal `python -m esptool`, independent of the global PATH. |
| `scripts/esp32_flash.py` | Plan or execute ESP32 erase/write using commands parsed from the MicroPython page. |
| `scripts/flash_mpy_firmware_manifest.py` | Validate start/state/phase_complete message envelopes and artifact paths. |

Minimal validation modes:

```text
flash_mpy_firmware_manifest.py --validate-start-phase --input <start_phase.json>
flash_mpy_firmware_manifest.py --validate-upstream --input <phase_complete.select_hw.json>
flash_mpy_firmware_manifest.py --validate-state --input <flash_mpy_firmware_state.json>
flash_mpy_firmware_manifest.py --validate-phase-complete --input <phase_complete.json> --artifact-root <artifact_root>
```

`scripts/requirements-esptool.txt` pins the esptool package version. Do not require the plugin to call the global `esptool` directly.

Script parameter conventions:

| Script | Required Input | Output Parameter |
| --- | --- | --- |
| `firmware_page_resolve.py` | `--board-name`, `--board-family`, usually also `--board-url` | `--out-json`; also compatible with `--output-json`. |
| `firmware_download.py` | `--resolved-json`, `--out-dir` | `--output-json`; also compatible with `--out-json`; it is recommended to pass `--artifact-root` to output a relative `downloaded_artifact_path`. |
| `list_serial_ports.py` | None; mock tests can add `--mode mock --mock-port COM3`, `--mock-port /dev/ttyUSB0`, or `--mock-port /dev/cu.usbmodem1101` | `--output-json`; also compatible with `--out-json`. |
| `find_uf2_mount.py` | Optional; defaults to finding `RPI-RP2`, tests can add `--candidate <path>` | `--output-json`; also compatible with `--out-json`. |
| `bootstrap_esptool.py` | None; add `--install` when installation is needed | `--output-json`; also compatible with `--out-json`. |
| `esp32_flash.py` | `--resolved-json`, `--firmware`, `--port` | `--output-json`; also compatible with `--out-json`; it is recommended to pass `--artifact-root` to output a relative `firmware_artifact_path`. |

When calling, prefer the canonical output parameters from the table above; compatibility aliases are only for fault tolerance, do not mix them in new documentation examples.

## Artifacts

Artifacts are written under `sessions/<session_id>/`:

```text
flash_mpy_firmware_state.json
firmware_page_resolved.json
firmware_download.json
firmware/<downloaded-file>
serial_ports.json
esptool_plan.json
flash_esp32_log.json
manual_flash_instructions.json
phase_complete.upy_flash_mpy_firmware_plugin.json
```

During debugging, `flash_mpy_firmware_phase_log.md` can be additionally written for local review of the complete execution process; it is not a required artifact and does not need to be included in `phase_complete.payload.artifacts`.

`esptool_bootstrap.json` is a local auxiliary debug file for recording the skill-internal esptool environment check or installation result; it is not a formal phase artifact and should not be included in `phase_complete.payload.artifacts` unless explicitly required by a subsequent protocol.

`phase_complete.payload.artifacts` must contain a `file_list` and list all formal artifacts produced by the current branch. Files referenced by `firmware.file`, `firmware.flash_result.log`, or `checkpoint.state_file` (such as firmware files, flash logs, checkpoint state) must be declared in artifacts. Artifact paths must be relative to `artifact_root`; do not write the native skill installation path into formal artifact paths.

The phase name in `phase_complete.payload.artifacts[].files[].description` must use the formal plugin name `upy-flash-mpy-firmware-plugin`. Do not use the old name `flash-mpy-firmware`; for example, the state file description should read `upy-flash-mpy-firmware-plugin phase state file`.

`bootstrap_esptool.py` without `--install` is a check mode; if the skill-internal `.venv-esptool` does not exist, the script outputs `status="missing"`, `action_required="install"` and returns 0. This is a recoverable state and should not be treated as a phase failure. Only run `bootstrap_esptool.py --install` after the user confirms permission to install. The recommended order for the ESP32 branch is: confirm flash -> bootstrap check/install -> `esp32_flash.py --plan-only` -> `esp32_flash.py --execute`, so that `esptool_plan.json.tool_version` reflects the real environment.

## State File

`flash_mpy_firmware_state.json` is used for recovery, retry, and troubleshooting; it is not a phase completion message. Do not write `status` as `phase_complete`; phase completion is expressed by the final `type="phase_complete"` file.

The state top-level fields must include:

| Field | Meaning |
| --- | --- |
| `protocol_version` | Fixed to `1.0`. |
| `msg_id` | Optional but recommended to write, unique message ID. |
| `session_id` | Current session ID. |
| `phase` | Fixed to `upy-flash-mpy-firmware-plugin`. |
| `status` | `in_progress`, `partial`, `success`, `failed`, or `cancelled`. |
| `type` | Optional but recommended to write `state`. |
| `source_phase_complete_path` | Relative path to the upstream `phase_complete.select_hw.json`. |
| `payload` | Current phase facts; it is recommended to place board, firmware, serial port, and flash results here. |
| `checkpoint` | Written when `partial`/`failed` and recoverable. |

Success state example:

```json
{
  "protocol_version": "1.0",
  "msg_id": "uuid",
  "session_id": "<session_id>",
  "phase": "upy-flash-mpy-firmware-plugin",
  "status": "success",
  "timestamp": "<runtime-utc-now>",
  "type": "state",
  "source_phase_complete_path": "sessions/<session_id>/phase_complete.select_hw.json",
  "payload": {
    "phase": "upy-flash-mpy-firmware-plugin",
    "firmware_action": "download_and_flash",
    "board_name": "ESP32_GENERIC_C3",
    "board_url": "https://micropython.org/download/ESP32_GENERIC_C3/",
    "download_slug": "ESP32_GENERIC_C3",
    "chip_family": "esp32c3",
    "firmware_file": "sessions/<session_id>/firmware/ESP32_GENERIC_C3-20260406-v1.28.0.bin",
    "firmware_version": "v1.28.0",
    "firmware_date": "2026-04-06",
    "file_type": "bin",
    "serial_port": "COM88",
    "flash_result": {
      "tool": "esptool",
      "tool_version": "4.11.0",
      "port": "COM88",
      "baud": 460800,
      "write_offset": "0",
      "erased_first": true,
      "chip": "esp32c3",
      "log": "sessions/<session_id>/flash_esp32_log.json"
    }
  }
}
```

## Checkpoints and Errors

When the user selects continue later/cancel, approval times out, selects download only, the network is temporarily unavailable, no serial port is selected, or manual flash is not confirmed, use `result=partial`, `next_phase=null`, and write a checkpoint.

Checkpoint structure:

```json
{
  "checkpoint": {
    "resume_step": "confirm_esp32_flash",
    "reason": "waiting_user_approval",
    "state_file": "sessions/<session_id>/flash_mpy_firmware_state.json"
  }
}
```

`resume_step` values:

```text
load_upstream_select_hw
select_firmware_action
resolve_firmware_page
download_firmware
scan_serial_ports
confirm_esp32_flash
run_esp32_flash
wait_pico_uf2_copy
manual_firmware_flash_confirm
phase_complete_validation
```

Structured error fields:

| Field | Meaning |
| --- | --- |
| `code` | Stable, machine-readable error code. |
| `message` | Error description for the user or developer; prefers Chinese when generated by the plugin/LLM, can retain English when directly passing through the original script error. |
| `severity` | `info`, `warning`, `error`, or `fatal`. |
| `recoverable` | Whether it can be retried/recovered. |
| `retryable` | Whether it can be retried with the same action and parameters. |
| `source` | The script or phase step that generated the error. |
| `field` | Optional JSON field path. |

Recommended error codes:

```text
invalid_upstream_phase
missing_firmware_url
firmware_page_lookup_failed
latest_firmware_not_found
download_failed
user_saved_partial
user_cancelled
serial_port_missing
esptool_failed
pico_copy_not_confirmed
manual_flash_not_confirmed
artifact_missing
absolute_path_in_artifact
phase_complete_invalid
```

## `phase_complete`

A successful payload must include both `firmware` and the complete `manifest_content`:

- `firmware` is the summary of this phase, retained for UI, logging, and lightweight validation.
- `manifest_content` must be copied from the complete `payload.manifest_content` of the upstream `select-hw`; do not discard project facts like `project_name`, `requirements`, `devices`, `mcu`, `hardware_selection`.
- On success, append/overwrite on the copied `manifest_content`:
  - `phase="upy-flash-mpy-firmware-plugin"`
  - `firmware_flash=<equivalent firmware facts from payload.firmware>`
  - `final_status="firmware_ready"`
  - `updated_at="<runtime-utc-now>"`
- `manifest_content.firmware_flash` must be consistent with the key fields of `payload.firmware`, at least including `status`, `action`, `board_name`, `board_url`, `source`, `flash_method`; if `latest_url`, `file`, `file_type`, `flash_result` have been resolved or generated, they must also be retained and kept consistent.
- A successful payload must include the source chain: `source_phase="select-hw"` and `source_phase_complete_path="sessions/<session_id>/phase_complete.select_hw.json"`.
- When the latest firmware has been resolved, `payload.firmware` and `manifest_content.firmware_flash` must include `latest_version` and `latest_date`.
- On successful ESP32 flash, `payload.firmware.flash_result` and `manifest_content.firmware_flash.flash_result` must include `baud`, `chip`, and a unified `write_offset`; `write_offset` uses the original command parameter value parsed from the MicroPython page, e.g., `"0"`; do not mix `"0"` and `"0x0"` within the same phase.
- `partial` and `failed` may omit `manifest_content`, but if they include it, `next_phase` must not be set to `upy-scaffold-plugin`.

```json
{
  "phase": "upy-flash-mpy-firmware-plugin",
  "result": "success",
  "summary": "MicroPython firmware flashing phase completed",
  "next_phase": "upy-scaffold-plugin",
  "firmware": {
    "status": "flashed",
    "action": "download_and_flash",
    "board_name": "ESP32_GENERIC_C5",
    "board_url": "https://micropython.org/download/ESP32_GENERIC_C5/",
    "latest_url": "https://micropython.org/resources/firmware/ESP32_GENERIC_C5-20260406-v1.28.0.bin",
    "file": "sessions/<session_id>/firmware/ESP32_GENERIC_C5-20260406-v1.28.0.bin",
    "file_type": "bin",
    "source": "micropython_latest",
    "flash_method": "esptool.py",
    "flash_result": {
      "tool": "esptool",
      "tool_version": "4.11.0",
      "port": "COM3",
      "write_offset": "0x2000",
      "erased_first": true,
      "log": "sessions/<session_id>/flash_esp32_log.json"
    }
  },
  "manifest_content": {
    "schema_version": "1.0",
    "phase": "upy-flash-mpy-firmware-plugin",
    "project_name": "esp32-c5-demo",
    "requirements": {
      "mcu_specified": "ESP32-C5",
      "network": "wifi"
    },
    "devices": [],
    "mcu": {
      "model": "ESP32-C5",
      "board_id": "esp32-c5-devkitc",
      "display_name": "ESP32-C5 DevKit",
      "firmware_url": "https://micropython.org/download/ESP32_GENERIC_C5/",
      "firmware_board_name": "ESP32_GENERIC_C5",
      "flash_tool": "esptool.py",
      "chip_family": "esp32c5"
    },
    "hardware_selection": {
      "selected_board": {
        "id": "esp32-c5-devkitc",
        "display_name": "ESP32-C5 DevKit",
        "chip_family": "esp32c5",
        "firmware": {
          "url": "https://micropython.org/download/ESP32_GENERIC_C5/",
          "board_name": "ESP32_GENERIC_C5",
          "port": "esp32"
        }
      }
    },
    "firmware_flash": {
      "status": "flashed",
      "action": "download_and_flash",
      "board_name": "ESP32_GENERIC_C5",
      "board_url": "https://micropython.org/download/ESP32_GENERIC_C5/",
      "latest_url": "https://micropython.org/resources/firmware/ESP32_GENERIC_C5-20260406-v1.28.0.bin",
      "file": "sessions/<session_id>/firmware/ESP32_GENERIC_C5-20260406-v1.28.0.bin",
      "file_type": "bin",
      "source": "micropython_latest",
      "flash_method": "esptool.py",
      "flash_result": {
        "tool": "esptool",
        "tool_version": "4.11.0",
        "port": "COM3",
        "write_offset": "0x2000",
        "erased_first": true,
        "log": "sessions/<session_id>/flash_esp32_log.json"
      }
    },
    "final_status": "firmware_ready",
    "updated_at": "<runtime-utc-now>"
  }
}
```

`firmware.status` values:

```text
downloaded
flashed
uf2_copied
manual_confirmed
skipped_user_confirmed
partial_download_only
failed
```

For `partial` and `failed`, `next_phase` must be null.
## Session Boundary Addendum

- Treat `runtime_context.session_root` and `source_phase_complete_path` as the `workflow_session_root` for firmware-download, flash logs, and final phase_complete files.
- A reopened chat or a separate session containing deployment logs is a `diagnostic_log_session` only. It must not change where firmware artifacts are written.
- Do not infer the active session from the latest `sessions/*` directory. Use the explicit start payload or user command argument.
- Preserve upstream `manifest_content` from the workflow session. Reference diagnostic sessions only in `artifacts`, `warnings`, or notes.
- Final `phase_complete.payload.runtime_context.session_root` must match the workflow session that will feed scaffold.
