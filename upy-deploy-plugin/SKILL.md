---
name: upy-deploy-plugin
description: Plugin-based workflow for MicroPython project deployment and runtime verification phase. Consumes the phase_complete from upy-generate-plugin, supports upload_only, clean_then_upload, erase_then_upload, uploads firmware, performs soft reset, captures REPL output, reads device logs, runs device-side tests, displays deployment results, and enters generate fix, autofix, or project library upload based on user feedback.
---

# upy-deploy-plugin Plugin Workflow

`upy-deploy-plugin` is the project deployment and runtime verification phase of the "one-sentence hardware creation" pipeline. It does not override the old `G:\MicroPython_Skills\upy-deploy`, nor does it re-flash the MicroPython interpreter firmware; the interpreter firmware phase remains the responsibility of `upy-flash-mpy-firmware-plugin`.

The official name for this phase is fully unified as:

```text
upy-deploy-plugin
```

All protocol messages, `phase_complete.payload.phase`, and `manifest_content.phase` must use this value. Do not mix in `deploy` or `upy-deploy`.

## Upstream and Downstream

Official main chain:

```text
upy-analyze-plugin
-> upy-select-hw-plugin
-> upy-flash-mpy-firmware-plugin
-> upy-scaffold-plugin
-> upy-generate-plugin
-> upy-deploy-plugin
```

When the upstream `upy-generate-plugin` succeeds and is deploy-ready, it must output `next_phase="upy-deploy-plugin"`. If `next_phase=null`, there must be a clear `next_phase_decision` explaining that the user chose to stop or that a blocker exists; the deploy main chain must not rely on manual patching.

After deployment completes, it must not end silently. It must display a deployment results tab and read user feedback:

| User Choice | Action |
| --- | --- |
| Regenerate | `upy-generate-plugin(mode=fix, source=user_feedback_after_deploy)` |
| Auto-debug | `upy-autofix-plugin` |
| Finish and upload project library | Enter project library upload/publish flow |

On FAIL, prioritize entering `upy-autofix-plugin`. If `upy-autofix-plugin` is not yet implemented, it may fall back to `upy-generate-plugin(mode=fix, source=deploy_fail)`.

## Input Contract

Start message:

```json
{
  "protocol_version": "1.0",
  "type": "start_phase",
  "phase": "upy-deploy-plugin",
  "session_id": "uuid",
  "idempotency_key": "upy-deploy-plugin:<session_id>:deploy:v1",
  "payload": {
    "phase": "upy-deploy-plugin",
    "source_phase": "upy-generate-plugin",
    "source_phase_complete_path": "sessions/<session_id>/phase_complete.upy_generate_plugin.json",
    "deploy_strategy": "clean_then_upload",
    "runtime_context": {
      "artifact_root": ".",
      "artifact_root_mode": "cwd",
      "session_root": "sessions/<session_id>",
      "project_root": "sessions/<session_id>/project",
      "resource_root": "<runtime-provided>"
    },
    "capabilities": {
      "approval_request": true,
      "file_operation": true,
      "script_run": true,
      "device_command": true,
      "serial_port_scan": true,
      "checkpoint_resume": true,
      "cancellation": true
    }
  }
}
```

The upstream `phase_complete` must satisfy:

```text
type == "phase_complete"
payload.result == "success"
payload.next_phase == "upy-deploy-plugin"
payload.manifest_content.phase == "generate"
```

## Deployment Strategies

Supported `deploy_strategy` values:

| Value | Meaning |
| --- | --- |
| `upload_only` | Do not clean device files; directly upload the current project |
| `clean_then_upload` | Perform a standard cleanup of old project files and business directories, then upload |
| `erase_then_upload` | Clean all listable files/directories on the device before uploading; requires a dry-run and a second confirmation |

`erase_then_upload` is not equivalent to re-flashing the MicroPython interpreter firmware. It only cleans files/directories within the MicroPython filesystem.

## Workflow

1. Validate the `start_phase` and the upstream `phase_complete`.
2. Read `project_root`, `project-manifest.json`, `firmware/`, and `tools/`.
3. First, run `scripts/check_environment.py` to check for the `mpremote` runtime; if missing, return `action_required` with installation instructions and do not proceed to touch the device.
4. Scan serial ports using the plugin's wrapper script `scripts/list_serial_ports.py`; this script only delegates to the shared implementation `shared-plugin-scripts/mpremote/list_serial_ports.py` and does not duplicate the serial port scanning logic.
5. Send an `approval_request(deploy_port_select)` for the user to select the real port.
6. Send an `approval_request(deploy_strategy_select)` for the user to select the deployment strategy.
7. If cleaning is selected:
   - `clean_then_upload`: Run `scripts/clean_device_project.py --mode project_files --dry-run`.
   - `erase_then_upload`: Run `scripts/clean_device_project.py --mode erase_all --dry-run`.
   - Display the list of files to be deleted and wait for confirmation.
   - After confirmation, run with `--execute`.
   - The `project_files` cleanup must cover old production-prohibited artifacts, including `conf.mpy`, `boot.mpy`, `main.mpy`, `board.mpy`, and `drivers/**/mock.py|mock.mpy`. Otherwise, even if the new upload is filtered correctly, the device might still run old files.
8. Install runtime dependencies declared by generate:
   - Read `project-manifest.json` or the upstream `phase_complete.payload.manifest_content.runtime_dependencies.mip`.
   - Call `scripts/install_mip_dependencies.py --project-root <project_root> --manifest <phase_complete_or_manifest> --port <port> --output-json ...`.
   - Only use `mpremote mip install` to install MicroPython/micropython-lib packages; do not vendor library source code into the project during the deploy phase.
   - After installation, use `mpremote fs ls` to verify that the target directory and package directories actually exist, e.g., `:lib`, `:lib/unittest`, and write the `fs_verify` into the result. mip may install pre-compiled `__init__.mpy` instead of `__init__.py`; this is a legitimate on-disk form. The verification script should accept `.py` or `.mpy`, but must retain the `matched_files` evidence.
   - If `mip install` fails due to network, proxy, or VPN environment unavailability, mark it as `runtime_dependency_install_network_unavailable`, prompt the user to fix the network and retry, and do not misclassify it as a generate code error.
   - Installation failure, import verification failure, or insufficient device space must be written as independent errors into `mip_install_result.json` and passed to `deploy_result.py --mip-install-json ...` for aggregation.
9. Run project tools:
   - `project/tools/flash_device.py --compile --upload --no-reset --port <port> --json-summary`
   - `--json-summary` is a required interface; the deploy-plugin only consumes structured results.
   - The upload summary must record `compiled_files`, `uploaded_files`, and `skipped_files`. `conf.py`, `boot.py`, and `main.py` should be uploaded as `.py`; do not deploy `:conf.mpy` or `:boot.mpy`. `firmware/drivers/**/mock.py`/`mock.mpy` are test doubles and must not be deployed to the device.
   - Even if the project tool returns success, if the upload summary or `mpremote fs cp` commands show that `:conf.mpy`, `:boot.mpy`, `:drivers/*/mock.py`, or `:drivers/*/mock.mpy` were uploaded, `deploy_result.py` must judge it as `FAIL`.
10. Soft reset and wait for reconnection:
    - `device_command(soft_reset)` or a whitelisted script.
    - `scripts/wait_for_device.py --port <port> --output-json ...`
11. Capture persistent REPL output using the dedicated `scripts/capture_repl.py`. It is recommended to call `scripts/capture_repl.py --reset-first --duration-ms <ms>` after upload, so the script first enters REPL, sends Ctrl-D for a soft reset, and continuously reads startup output; do not reset/wait first and then start listening, as this will miss the `main.py` startup traceback.
12. Read device-side logs:
    - Before deployment, provide log strategy options: keep old logs, read and download old logs, or clear old logs before deployment.
    - `project/tools/read_device_log.py`
    - `project/tools/log_report.py`
    - Log clearing can only be performed after user confirmation, by calling the project tool's `--clear` or the cleanup script's `--include-logs`; do not silently delete old logs by default.
13. Optionally run device-side contract tests:
    - First, send an `approval_request(run_device_tests)`.
    - If the user chooses to run, call `scripts/run_device_tests.py --project-root <project_root> --port <port> --output-json ... --log-file ...`.
    - Test file sources are `project/device/tests/test_*.py` and `project/test/device/test_*.py`.
    - If device tests require `firmware/drivers/**/mock.py`, these can only be uploaded by `run_device_tests.py` as temporary test artifacts, run, and then deleted, with `mpremote fs ls` used to verify deletion; do not include mocks in the production upload summary.
14. Run `scripts/deploy_result.py` to generate a structured deploy verdict.
15. Display the results tab:
    - PASS or PASS_WITH_WARNINGS: `approval_request(deploy_result_feedback)`
    - FAIL or NEEDS_USER_CONFIRMATION: `approval_request(deploy_fail_next_action)`
16. Before outputting `phase_complete`, you must run `scripts/deploy_manifest.py --input <phase_complete> --validate-phase-complete`; on failure, the deploy must not be judged as success.

## approval_request

### deploy_port_select

Must display the list of scanned ports. In real operation, do not hardcode `COM3`; a fixed port can only be used for samples/mocks.

### deploy_strategy_select

Must include:

```text
upload_only
clean_then_upload
erase_then_upload
save_partial
```

The recommended default selection is `clean_then_upload`.

### confirm_clean_device_project

Display the list of files to be deleted from `clean_device_project.py --mode project_files --dry-run`.

### confirm_erase_device_fs

Display the complete list of files/directories to be deleted from `clean_device_project.py --mode erase_all --dry-run`. The user must confirm a second time.

### run_device_tests

After upload, soft reset, waiting for device recovery, and reading device logs, it is recommended to ask whether to run device-side contract tests. The default suggestion is to run, but it must be possible to skip, because some projects' device tests may touch real hardware, or the user may only want a quick upload for observation.

This request must provide at least:

```text
run_device_tests
skip_device_tests
save_checkpoint
```

The run results are saved as:

```text
device_tests_result.json
device_tests_output.log
```

### deploy_result_feedback

After PASS or `PASS_WITH_WARNINGS`, display:

- Serial port / device.
- Deployment strategy.
- Cleanup result.
- `flash_device.py --json-summary`.
- Soft reset / wait result.
- REPL output summary.
- Device-side log report.
- Device tests result.
- Initial verdict: PASS or `PASS_WITH_WARNINGS`.

Optional user feedback text must be collected, such as actual device behavior, mpremote output, serial port errors, manually observed issues, and device log summaries. When the user chooses to regenerate, `error_context` must be passed.

### deploy_fail_next_action

After FAIL, display the same diagnostic summary and allow entry into `upy-autofix-plugin`, `upy-generate-plugin(mode=fix)`, or saving a checkpoint. When entering generate fix, the complete `error_context` must be carried.

Recommended payload:

```json
{
  "mode": "fix",
  "source": "user_feedback_after_deploy",
  "error_context": {
    "user_feedback": "<user feedback text>",
    "deploy_result_path": "sessions/<session_id>/phase_complete.upy_deploy_plugin.json",
    "serial_excerpt": "<REPL or serial excerpt>",
    "device_log_excerpt": "<device log excerpt>",
    "device_tests_result_path": "sessions/<session_id>/device_tests_result.json",
    "deploy_errors": [],
    "previous_generate_commit": "<commit>"
  }
}
```

## Result Determination

`scripts/deploy_result.py` must synthesize the upload summary, clean result, wait/probe result, REPL capture, device log report, device tests result, and user manual feedback.

Hard FAIL signals:

| Signal | Result |
| --- | --- |
| upload failed | `FAIL` |
| clean failed | `FAIL` |
| mip dependency install/verify failed | `FAIL` |
| forbidden runtime upload (`:conf.mpy`, `:boot.mpy`, `:drivers/*/mock.py`, `:drivers/*/mock.mpy`) | `FAIL` |
| wait/probe failed | `FAIL` |
| REPL Traceback/panic/MemoryError/ValueError/OSError/ImportError/AttributeError | `FAIL` |
| log_report.error_count > 0 | `FAIL` |
| device tests failed | `FAIL` |

An empty REPL output should not directly result in a FAIL. If the serial capture is empty, but the upload/cleanup succeeded, the log report has `error_count=0`, and device tests did not fail, then output `PASS_WITH_WARNINGS` and include a warning, e.g., `serial capture produced no output`. This is because the application might only write to a rotating file logger, or there is no stdout at runtime.

## Device Tools Area

In addition to the main workflow, the UI can provide an independent "Device Tools" area:

- Scan serial ports.
- Connect / listen to output.
- Execute probe commands.
- Read device logs.
- Run device tests.
- Clean project files (dry-run).
- Full erase (dry-run).

These buttons do not necessarily advance the main chain, but their outputs should all be attachable to `deploy_result_feedback`, `deploy_fail_next_action`, and `upy-generate-plugin(mode=fix).error_context`.

## Scripts

| Script | Purpose |
| --- | --- |
| `scripts/check_environment.py` | Check for `mpremote`, optional `pyserial`, and provide installation hints |
| `scripts/mpremote_runtime.py` | The sole `mpremote` process adaptation layer within the deploy plugin; parses `UPY_MPREMOTE`, PATH, `python -m mpremote` |
| `scripts/list_serial_ports.py` | Serial port scanning entry point within the deploy plugin; a thin wrapper around the shared serial port scanning script |
| `shared-plugin-scripts/mpremote/list_serial_ports.py` | Shared serial port scanning implementation, referenced by both flash and deploy |
| `scripts/deploy_manifest.py` | Validate start/upstream/phase_complete |
| `scripts/clean_device_project.py` | Dry-run/execute cleanup of device files |
| `scripts/install_mip_dependencies.py` | Execute `mpremote mip install` based on `runtime_dependencies.mip` and verify imports |
| `scripts/wait_for_device.py` | Wait for device recovery after soft reset |
| `scripts/capture_repl.py` | Persistent REPL output capture |
| `scripts/run_device_tests.py` | Execute device-side unittest files via `mpremote run` and output JSON |
| `scripts/deploy_result.py` | Aggregate upload/mip install/serial/log/device tests report, determine PASS/FAIL |
| `scripts/requirements-runtime.txt` | Runtime pip dependency list: `mpremote`, `pyserial` |

## mpremote Constraints

- Do not vendor the pip-installed `mpremote` package source code into the plugin; the plugin encapsulates "how to discover, invoke, report errors, and prompt for installation".
- All scripts within the deploy plugin must call `mpremote` via `scripts/mpremote_runtime.py`; do not scatter `["mpremote", ...]` across individual scripts.
- `mpremote` resolution order: `UPY_MPREMOTE` environment variable, `mpremote` in PATH, current Python's `python -m mpremote`. If missing, return `action_required` and `python -m pip install mpremote`.
- MicroPython runtime packages must be installed using `mpremote mip install`, typically sourced from `micropython-lib` or the official mip index; deploy does not default to fetching source code into the local project.
- `scripts/install_mip_dependencies.py` first probes with `verify_import`, installs if missing, and probes again after installation. The result must be passed to `deploy_result.py --mip-install-json`.
- `mpremote mip install` may fail due to network, proxy, or VPN environment unavailability. Such failures must be classified as `runtime_dependency_install_network_unavailable`, retain the stdout/stderr summary, and have `deploy_result.py` clearly indicate network/proxy/VPN issues, rather than conflating them with a normal device test failure.
- mip installation cannot be considered complete based solely on an import probe. After installation, use `mpremote fs ls` to verify the target directory and package directories, e.g., `fs ls :lib`, `fs ls :lib/unittest`, confirming that key files like `__init__.py` or `__init__.mpy` are on disk; recursive subdirectories need to be listed level by level. The filesystem verification result must be written into `mip_install_result.json.records[].fs_verify`.
- Serial port enumeration must uniformly call `scripts/list_serial_ports.py`; this script is a thin wrapper around `shared-plugin-scripts/mpremote/list_serial_ports.py`, and the implementation is no longer duplicated.
- Upload and filesystem operations should preferentially use `mpremote connect <port> resume fs ...` to avoid an implicit soft reset before file transfer.
- `scripts/mpremote_runtime.py` supports a passthrough for manual debugging, e.g., `mpremote_runtime.py --run --port <port> -- resume exec "print('hello')"`.
- Long-duration listening, post-run output capture, and multi-round interactions must use a persistent session model; `scripts/capture_repl.py` is the dedicated entry point for the deploy phase.
- `mpremote resume exec` should only be used for short probes or one-time actions like pre-deployment cleanup; do not use repeated `resume exec` as a substitute for persistent REPL listening.
- On Windows, use explicit `COMn`; on macOS, use `/dev/tty.usbmodem*` or `/dev/tty.usbserial*`; on Linux, prefer `/dev/serial/by-id/*` or the stable path resolved by mpy-dev.

## phase_complete

A success payload must include:

- `phase="upy-deploy-plugin"`
- `result="success"`
- `deploy_result`
- `manifest_content.phase="upy-deploy-plugin"`
- `manifest_content.deploy` or `manifest_content.deploy_result`
- `artifacts[]`
- `next_phase` chosen based on user feedback

`manifest_content` must retain the complete upstream manifest and then append deploy facts; it must not write only a summary.

`phase_complete.payload.deploy_result` must come from the structured result of `scripts/deploy_result.py` or be consistent with it field by field. An LLM may summarize the result, but must not manually rewrite blocking failures from the underlying `mip_install_result.json`, upload summary, device tests, log report, or REPL capture into a PASS.

A success `payload.artifacts` must reference independent raw evidence files: `deploy_result.json`, `upload_summary.json`, `clean_result.json`, `mip_install_result.json`, `device_tests_result.json`, as well as serial/REPL capture and device log reports. Listing only a narrative summary or the `phase_complete` itself as an artifact is not acceptable.

## Strong Constraints

- Do not override the old `upy-deploy`.
- Do not re-flash the MicroPython interpreter firmware.
- Do not modify generated code; fixes are left to generate/autofix.
- Do not hardcode `COM3` in real operation.
- All local actions go through `script_run`, `device_command`, `file_operation`, or `approval_request`.
- `erase_then_upload` must have a dry-run and a second confirmation.
- Long-duration serial output capture must use a persistent session approach, avoiding repeated `resume exec`.
## Final Boundary Addendum

- Treat `runtime_context.session_root`, `runtime_context.project_root`, and the explicit `source_phase_complete_path` as the `workflow_session_root`. A separate session containing logs is a `diagnostic_log_session` and must not receive deploy artifacts unless the user explicitly makes it the workflow target.
- Deploy success means deployment-observation success, not code-generation correctness. A PASS requires upload/clean/mip/device probes/log report/device tests to have no blocking errors. A PASS does not authorize manual source edits during deploy.
- Deploy must not fix generated source code or mark success after ad-hoc debugging changes. Runtime code fixes go through `upy-autofix-plugin` or `upy-generate-plugin(mode=fix)` with a structured `error_context`.
- Deploy must not add broad Timer or peripheral semantic preflight. Timer and peripheral API correctness are generate gates. Deploy records evidence from upload summary, REPL capture, device logs, device tests, and user observation.
- Empty REPL output, COM re-enumeration, or missing logs after the user unplugged/replugged a device is observation-incomplete, not proof of firmware failure. If upload succeeded and there is no traceback/log error/test failure, return `PASS_WITH_WARNINGS` and record the observation limitation.
- Forbidden runtime uploads are blocking even if the project upload tool says success: `:main.mpy`, `:boot.mpy`, `:conf.mpy`, `:firmware/**`, `__pycache__`, `*.pyc`, and `drivers/**/mock.py|mock.mpy`.
- Before upload, project_files clean must remove old deploy artifacts such as `main.mpy`, `boot.mpy`, `conf.mpy`, `board.mpy`, stale `drivers/**/mock.py|mock.mpy`, and old wrong-root `firmware/**` paths when present on device.
- MicroPython runtime packages from micropython-lib must be installed with `mpremote mip install`, then verified with import probes and `mpremote fs ls` on the relevant target folders such as `:lib` and `:lib/unittest`. Network/proxy/VPN failure is `runtime_dependency_install_network_unavailable`, not a generate code bug.
- Device-side unittest mocks are temporary test artifacts only. `scripts/run_device_tests.py` must record upload, cleanup, and cleanup verification for `firmware/drivers/**/mock.py`; production upload must still reject mocks.
- REPL capture should prefer reset-first capture when safe so startup tracebacks are visible. Device file logs supplement REPL output; they do not replace startup traceback capture.
- `deploy_fail_next_action` and `deploy_result_feedback` must carry `error_context` with deploy result path, serial excerpt, device log excerpt/report, device tests result path, mip install result, forbidden upload list, user observation, and previous generate commit when available.
