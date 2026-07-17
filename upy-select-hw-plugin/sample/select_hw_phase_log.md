# select-hw Phase Log

Sample select-hw phase log.

## Runtime Context

- `artifact_root`: `.` (cwd)
- `artifact_root_mode`: `cwd`
- `session_root`: `sessions/022ad742-3269-42e9-ac20-c14f477ecdf2`
- `resource_root`: `<runtime-provided>`

## Step Timeline

| Step ID | Level | Message |
|---------|-------|---------|
| upstream_manifest_loaded | info | Analyzed manifest content loaded |
| board_matching | info | Matching boards based on MCU preferences and requirements |
| board_definition_loaded | info | Loaded upy-analyze-plugin/boards/esp32-c3-devkitm.json |
| board_selected | success | ESP32-C3-DevKitM-1 confirmed |
| firmware_check | info | Verifying MicroPython firmware |
| firmware_ok | success | Firmware entry ESP32_GENERIC_C3 available |
| pin_assignment | info | Assigning I2C/GPIO/I2S/power pins |
| pin_assignment_draft_ready | info | Pin assignment draft generated |
| pin_assignment_done | success | Pin assignment completed |
| bom_ready | success | BOM generated |
| manifest_validation | info | Running select_hw_manifest.py validation |

## Resource References

All paths below are repository-relative:

- `upy-analyze-plugin/boards`
- `upy-select-hw-plugin/scripts/select_hw_manifest.py`
- `upy-project-gen-toolchain-spec/scripts/workflow_time.py`

## Artifacts

All artifact paths are relative to artifact_root under artifact_root_mode=cwd:

- `sessions/<session_id>/select_hw_draft.json`
- `sessions/<session_id>/select_hw_validated.json`
- `sessions/<session_id>/phase_complete.select_hw.json`
- `sessions/<session_id>/pin_assignment_log.md`
- `sessions/<session_id>/select_hw_phase_log.md`
