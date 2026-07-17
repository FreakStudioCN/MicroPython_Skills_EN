# Mock Messages

This directory contains JSON samples for each message type, used by plugin engineers for independent development and testing.

## Usage

Plugin engineers can use these JSON files to test UI rendering without connecting to the server:

```javascript
// Test approval card rendering
const approvalMsg = require('./approval-request.json');
webview.postMessage(approvalMsg);

// Test phase completion panel
const completeMsg = require('./phase-complete.json');
webview.postMessage(completeMsg);
```

## File List

| File | Corresponding Message Type | Status |
|------|---------------------------|--------|
| `approval-request.json` | approval_request | ✅ Created — Device confirmation card (device_confirm_001) |
| `status-update.json` | status_update | ✅ Created — 9 samples, including info/success/warn/error four levels |
| `device-command.json` | device_command | ✅ Created — 6 samples, including exec/cp/mkdir/ls/soft_reset/run |
| `file-operation.json` | file_operation | ✅ Created — 6 samples, including write/read/append/mkdir/list/delete |
| `script-run.json` | script_run | ✅ Created — 6 samples, including flake8/init_manifest/pack_driver/extract_pdf/run_on_device/flash_device |
| `phase-complete.json` | phase_complete | ✅ Created — Complete analyze phase results, including table/file_tree/markdown three artifact types |
| `stream.json` | stream | ✅ Created — 7 samples, including device_output and script_stdout two stream types |

After each skill's interface documentation is finalized, supplement the skill-specific mock message samples to this directory.
