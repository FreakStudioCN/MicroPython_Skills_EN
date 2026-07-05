---
name: upy-gen-driver-plugin
description: Plugin-based workflow skill for generating missing MicroPython hardware drivers from datasheets, Arduino/C/C++ sources, GitHub repositories, chip models, or current project cold-driver items. Applicable when the global plugin tool "Generate Missing Hardware Driver" is triggered, when `devices[].driver.status=cold_driver_required` exists in the manifest, or when deploy/autofix feedback indicates a missing or broken hardware driver, and the flow requires session/checkpoint/resume, retry, timeout, cancellation, permission prompts, structured errors, and artifact manifests.
---

# upy-gen-driver-plugin

Generate missing MicroPython drivers, but do not modify the legacy `upy-gen-driver` skill. This skill is the plugin-based workflow version: local files, scripts, devices, and user confirmation actions are all expressed via protocol messages, while outputting recoverable artifacts for both plugin execution and local mock testing.

## Operating Modes

- `pipeline`: Enter from an existing project session, typically after scaffold and before generate. Reads upstream `manifest_content`, writes project driver files, then returns to `upy-generate-plugin`.
- `standalone`: Enter from the global plugin tool "Generate Missing Hardware Driver". Requires the user to provide a PDF, Arduino/C/C++ source, GitHub URL, chip model, or image input, then generates a standalone driver package and test materials.
- `resume`: Continue from `session_state.upy_gen_driver_plugin.json`. Before reusing any checkpoint, the artifact hash must be verified first.
- `fix`: Repair a generated driver based on deploy/autofix feedback, prioritizing minimal changes.

## Required References

Only read these references when needed:

- `references/protocol_fields.md`: message envelope, start payload, checkpoint, phase_complete, file_manifest, permissions, structured errors.
- `references/legacy_upy_gen_driver_rules.md`: Legacy driver-generation rules that must be preserved.
- `references/norm_driver_p0_rules.md`: Production driver normalization checklist.

## Core Rules

- Do not overwrite or edit `G:\MicroPython_Skills\upy-gen-driver`.
- Use `phase="upy-gen-driver-plugin"` for the envelope, and `gen-driver` for the payload/domain phase.
- The protocol identity of this skill is fixed as `upy-gen-driver-plugin`. Do not abbreviate, rename, alias, or infer it as `upy-driver-plugin`, `driver`, `gen-driver-plugin`, or any other name; if legacy artifacts using these names are found, they must be treated as stale/wrong-phase artifacts and regenerated with the correct identity.
- The final protocol file name must be `phase_complete.upy_gen_driver_plugin.json`; the session state file name must be `session_state.upy_gen_driver_plugin.json`. Do not output `phase_complete.upy_driver_plugin.json`, `session_state.upy_driver_plugin.json`, or any other phase file name.
- All phase-scoped `idempotency_key`, `checkpoint_id`, `resume_phase`, and permission action keys must use the `upy-gen-driver-plugin` prefix; the `phase` and `domain_phase` in the business payload must only use `gen-driver`.
- Plugin invocations and local skill-call tests must use the same message contract. Local tests can execute files directly, but must still write the `session_state`, permissions, file manifest, structured errors, and `phase_complete` artifacts that the plugin host would receive.
- Formal artifact paths must be relative to `artifact_root` or `project_root`; do not write Windows drive paths into `phase_complete`.
- Treat `runtime_context.session_root` as the source of truth for the workflow session. Do not infer the current session from the latest `sessions/*` directory.
- Default device addresses in MicroPython I2C driver code, debug drivers, test scripts, and wiring docs must use 7-bit addresses. Do not pass 8-bit transfer addresses containing the R/W bit to `scan()`, `readfrom_mem()`, `writeto_mem()`, or similar I2C APIs.
- Write/read addresses like `0x3C/0x3D` from a datasheet can only be recorded as datasheet evidence; they must be normalized to a 7-bit address before code generation, e.g., `0x3C >> 1 == 0x1E`.
- Use `permission_request` for local file, script, network, and device operations; use `approval_request` for user business choices.
- Every local action must have a stable `idempotency_key`.
- Every script/device/approval wait must have a `timeout_ms`.
- On user cancellation, no device, timeout, stale artifact, missing capability, or hardware verification exhaustion, output `result="partial"` with a checkpoint and `structured_errors[]`; do not claim success.
- Use `HOST_CAPABILITY_MISSING` when a host capability is missing, and specify the capability name in `details.missing_capability`; only use `DEVICE_NOT_FOUND` when the host supports and has actually executed a device scan/run but still cannot find the device. Do not put `missing_capability=device_command` inside `DEVICE_NOT_FOUND`.
- Hardware verification can only be skipped when the user explicitly chooses to do so, and the final result must include a warning. The default behavior is to save a checkpoint and wait for a subsequent resume.
- When partial and verification is incomplete, must write `hardware_verified=false` and `verification_mode="none"`. Do not mark a partial result with no device, cancellation, timeout, or ordinary unverified status as `verification_mode="mock"`.
- `verification_mode="mock"` is only allowed for success results where a local mock self-test actually returned `SELF_TEST_PASS`; it still cannot set `hardware_verified=true`.
- File write actions for an unverified `{chip}.py` must use the `write_driver_artifact` idempotency key; `write_production_driver` is only allowed when `file_manifest.files[].role="production_driver"`.
- When retrying the same action, keep the same `session_id` and action-level `idempotency_key`; set `retry_of` to the original message id, and append a state event with `status="retrying"`.
- Cancellation is a recoverable partial result by default, unless the user explicitly discards artifacts. Keep the last trusted artifact and set the checkpoint to `cancelled`.
- Timeouts must not be handled silently. Host, script, approval, and device timeouts must all be converted into structured errors; if it is possible to continue from a checkpoint, set `retryable=true`.

## Start Phase Contract

Receives a `start_phase` from the plugin. The envelope must contain `protocol_version`, `msg_id`, `session_id`, `phase="upy-gen-driver-plugin"`, `type="start_phase"`, and a stable `idempotency_key`.

Envelope fields:

| Field | Meaning |
|---|---|
| `protocol_version` | Protocol schema version. Use `"1.0"` before any breaking changes; refuse to continue if an unknown major version is encountered, do not guess compatibility. |
| `msg_id` | Unique id for the current protocol message. Used for logging, `retry_of`, and user-visible diagnostics. |
| `session_id` | Stable workflow id. Must remain the same across retries, resumes, cancellations, and timeout recovery. |
| `phase` | Plugin envelope phase. Must be `upy-gen-driver-plugin`, do not write `gen-driver`. |
| `timestamp` | UTC ISO timestamp, used for ordering and auditing. |
| `type` | Message type, e.g., `start_phase`, `permission_request`, `script_run`, `device_command`, `status_update`, or `phase_complete`. |
| `idempotency_key` | Stable action key. Reuse the same key when retrying the same action to avoid duplicate file writes or duplicate device actions. |
| `retry_of` | When the current message is retrying or completing a failed action, fill in the previous `msg_id`. Use `null` for the first attempt. |

Payload fields:

| Field | Meaning |
|---|---|
| `mode` | Execution mode: `pipeline`, `standalone`, `resume`, or `fix`. |
| `phase` / `domain_phase` | Business phase. Must be `gen-driver`; it is used to distinguish from the plugin envelope phase. |
| `source_phase` | The upstream phase requesting the driver generation, typically `upy-scaffold-plugin`, `upy-generate-plugin`, or deploy/autofix feedback. |
| `source_phase_complete_path` | Relative path to the upstream `phase_complete` artifact, used as a source of evidence. |
| `manifest_content` | Current project manifest object. In `pipeline` mode, use it to find `devices[].driver.status == "cold_driver_required"` and update the generated driver path. |
| `source` | Driver evidence source: PDF, Arduino/C/C++ file, GitHub URL, chip model, image, or current cold-driver item. If missing, ask the user via `approval_request(gen_driver_input)`. |
| `runtime_context.artifact_root` | Root directory for session artifacts. Formal paths in the output must be relative to this root. |
| `runtime_context.session_root` | Canonical session directory. State, logs, and the final `phase_complete` are written here. |
| `runtime_context.project_root` | Project directory, where generated driver files and manifest updates are placed. |
| `runtime_context.file_operation_root` | Maximum root directory allowed for file writes initiated via the plugin. |
| `runtime_context.resource_root` | Skill resource directory, used to locate bundled scripts and references. |
| `capabilities` | Operational capabilities supported by the host. Must check before using upload, file operations, scripts, device commands, cancellation, checkpoint resume, idempotency cache, or network. |
| `timeouts` | Timeout budgets for each operation, in milliseconds. If missing, use explicit defaults and write the final timeout used into the message. |
| `resume_from` | Checkpoint descriptor for `resume` mode. Must verify hash and session identity before continuing. |

If `mode` is missing, infer `pipeline` only if the current manifest contains `driver.status=cold_driver_required`; otherwise use `standalone`.

## Output Field Meanings

`phase_complete.payload` fields:

| Field | Meaning |
|---|---|
| `result` | `success`, `partial`, or `failed`. Use `partial` for no device, user cancellation, timeout, missing capability, stale artifact, or verification exhaustion but still recoverable. |
| `summary` | Short human-readable result description. Must state whether hardware verification was completed. |
| `next_phase` | Typically `upy-generate-plugin` after success; use `null` for partial/failure waiting for resume or user action. |
| `checkpoint` | Resume anchor, containing `checkpoint_id`, `resume_phase`, `resume_step`, and `state_file`; `checkpoint_id` must use `upy-gen-driver-plugin:<session_id>:<checkpoint_name>`. |
| `permissions[]` | Audit trail of file/script/device/network/manifest permission requests or local mock auto-grants. |
| `file_manifest.files[]` | Formal manifest of generated, updated, skipped, or failed files. Each path must be relative and must include a `role`. |
| `artifacts[]` | User-facing artifact groupings. Must contain at least one non-empty `file_list` entry, and each entry must have `files[]` or `items[]`. |
| `structured_errors[]` | Machine-readable errors. Can be empty only for `success`. |
| `manifest_content` | Updated project manifest in `pipeline` mode; use `null` for `standalone` without a project manifest. |

`file_manifest.files[]` roles:

| Role | Meaning |
|---|---|
| `source` | Source file provided by the user or fetched from the network. |
| `extracted_text` | PDF extraction output generated by `extract_pdf.py`. |
| `mapping` | Arduino/C/C++ structure and API mapping generated by `convert_arduino.py`. |
| `understanding` | `driver_understanding.json`, structured hardware facts for driver generation. |
| `debug_driver` | `{chip}_debug.py`, a verbose single-file driver for hardware verification. |
| `production_driver` | `{chip}.py`, a normalized driver for project integration. |
| `test` | `test_{chip}.py` standalone validation script. |
| `wiring` | `wiring_{chip}.md` wiring and usage instructions. |
| `verify_log` | Hardware or mock verification log. |
| `manifest` | Project manifest update. |
| `state` | `session_state.upy_gen_driver_plugin.json`. |
| `phase_complete` | Final protocol result artifact. |

When real hardware verification has not been completed, if `{chip}.py` is retained, `file_manifest.files[].role` must use `artifact`, and the user-facing `artifacts[].file_list` text must be `Driver artifact (unverified)` or `Unverified driver artifact`, not `Production driver (unverified)`.

When real hardware verification has not been completed, if `{chip}.py` is written, the permission/action `idempotency_key` must use `upy-gen-driver-plugin:<session_id>:write_driver_artifact:<chip>:v1`. The `write_production_driver` key is only allowed after real hardware verification passes, the user explicitly skips verification, or a local mock success.

`structured_errors[]` fields:

| Field | Meaning |
|---|---|
| `code` | Stable uppercase error code, e.g., `DEVICE_RUN_TIMEOUT`. |
| `severity` | `warning`, `error`, or `fatal`. |
| `phase_step` | Step where the error occurred, e.g., `source_preprocess` or `hardware_verify`. |
| `retryable` | Whether it can be continued via retry/resume without creating a new session. |
| `message` | Human-readable description. |
| `details` | Machine-readable context, e.g., timeout, command, port, path, source hash, missing capability, or log path. |
| `next_action` | Suggested next action, e.g., `connect_device_and_resume`, `retry_device_run`, or `request_pdf_or_arduino_source`. |

## Plugin and Local Compatibility

Both execution forms use the same contract:

- Plugin host: Sends protocol messages and waits for `approval_response`, `permission_response`, `file_result`, `script_result`, `device_result`, or `cancellation`.
- Local mock test: Executes equivalent actions locally, then writes events of the same protocol shape to `sessions/<session_id>/gen_driver/message_log.jsonl` or final artifacts.
- Both forms must generate `sessions/<session_id>/session_state.upy_gen_driver_plugin.json`.
- Both forms must generate `phase_complete.upy_gen_driver_plugin.json` for success, partial, failed, cancelled, and timeout outcomes.
- Local tests must not bypass permission semantics. Even if mock auto-grant is used, file/script/device permissions must be recorded in `payload.permissions[]`.
- Local tests must not treat mock `SELF_TEST_PASS` as real hardware proof. Only mark `verification_mode="mock"` when a local mock self-test actually returned `SELF_TEST_PASS`; partial results with no device, cancellation, timeout, or without running a mock self-test must be marked `verification_mode="none"`.

## Workflow

1. Validate envelope, runtime roots, and capabilities.
2. If source is missing, issue `approval_request(gen_driver_input)` to let the global tool input card collect materials.
3. Collect one source type: PDF, Arduino/C/C++ source, GitHub URL, chip model, image, or current project cold-driver item.
4. Preprocess the source via protocol `script_run`:
   - PDF: `scripts/extract_pdf.py --input <path> --output <json> --json-summary`
   - Arduino/C/C++: `scripts/convert_arduino.py --input <path> --output <json> --json-summary`
5. Write `driver_understanding.json` via `file_operation(write)`. Content must include protocol, addressing, ID register, ready strategy, data integrity, register map, source evidence, and ambiguity notes. I2C `addressing` must distinguish `address_7bit`, datasheet write/read transfer address, derivation, and evidence source.
6. Generate `{chip}_debug.py` via `file_operation(write)`. The debug driver must include self-test prints and bounded polling.
7. Update session state checkpoint to `debug_driver_written`.
8. Request permission for device scan and debug run. If no device is available, issue `approval_request(gen_driver_no_device)`, offering `retry`, `save_partial`, and `cancel`.
9. Run up to 10 rounds of hardware verification, command form: `scripts/run_on_device.py --com <port> --file <debug.py> --capture --timeout-ms 30000 --json-summary`.
10. If `SELF_TEST_PASS` occurs, checkpoint to `hardware_verify_passed`. Otherwise, analyze the log, edit the debug driver, and retry until the limit is reached.
11. Only after verified pass or user explicit skip with warning, generate the production `{chip}.py`. Remove debug prints, retain meaningful exceptions, and maintain dependency injection.
12. Normalize the production driver using `references/norm_driver_p0_rules.md`.
13. Generate `test_{chip}.py` and `wiring_{chip}.md` for standalone hardware validation.
14. Optionally run the standalone test after `approval_request(gen_driver_standalone_test)`.
15. In `pipeline` mode, update `project/project-manifest.json` and `manifest_content.devices[].driver` to point to the generated local driver.
16. Only issue `approval_request(gen_driver_next_step)` when a user choice is needed. Common choices include proceeding to `upy-generate-plugin`, ending the flow, or publishing later.
17. Write `phase_complete.upy_gen_driver_plugin.json`, validate it with `scripts/validate_phase_complete.py`, then output as the final result.

## Driver Understanding Contract

Before writing any driver file, write `gen_driver/docs/driver_understanding.json`. This object is the evidence bridge between source material and generated code.

Must include at least:

| Field | Meaning |
|---|---|
| `chip` | Normalized chip/module id used for filenames. |
| `source_evidence[]` | Datasheet pages, Arduino lines, URLs, or user notes used as evidence. |
| `protocol` | `i2c`, `spi`, `uart`, `onewire`, or another explicit bus type. |
| `addressing` | I2C address, SPI mode/CS notes, UART baud rate, or equivalent connection facts. For I2C, must write `address_7bit`; if the datasheet gives 8-bit write/read transfer addresses, must also write `datasheet_write_8bit`, `datasheet_read_8bit`, `derivation`, and `code_address_rule="Use address_7bit for MicroPython I2C APIs."`. |
| `chip_identification` | ID/WHO_AM_I/CHIP_ID register and expected value; if none, write `N/A` and provide a fallback read/write sanity check. |
| `ready_strategy` | Status-bit polling, interrupt pin, fixed delay, or no ready signal. Must include timeout and datasheet timing. |
| `register_map[]` | Address, name, bit fields, read/write permissions, reset/default value, and write-only notes. |
| `init_sequence[]` | Reset/configuration steps, timing, and read-back expectations. |
| `data_format` | Endianness, signedness, scaling formula, units, and CRC/checksum rules. |
| `shadow_state` | Internal variables like `_gain`, `_vref`, `_mode`, and which setter each variable belongs to. |
| `ambiguities[]` | Unresolved facts requiring user confirmation or conservative handling. |

Do not generate a production driver based solely on unstructured notes. If the understanding is incomplete, return `DATASHEET_PARSE_INSUFFICIENT` and save a checkpoint.

## Debug Driver Requirements

Generate `project/firmware/drivers/<chip>_driver/<chip>_debug.py` as a single-file, fast-iteration version. It should be verbose, runnable on the device, and easy to fix.

Must satisfy:

- Print ASCII/English self-test messages.
- If using `const(...)`, must `from micropython import const` or provide a MicroPython-safe fallback; do not rely on an implicit global `const`.
- `const(...)` is only for integer constants. Float constants, scale factors, and sensitivity values must use ordinary variables, e.g., `_MGAUSS_PER_LSB = 1.5`, do not generate `const(1.5)`.
- Print source evidence at the file header, e.g., datasheet page/table or Arduino line.
- Validate constructor arguments before using the bus.
- Use externally injected I2C/SPI/UART objects; do not instantiate board pins inside the driver class.
- For I2C, do not restrict the bus type with `isinstance(i2c, I2C)`; use capability/duck typing checks so that both `machine.I2C` and `SoftI2C` compatible objects are usable.
- On initialization, bring the chip into a known state via reset or explicit configuration confirmation.
- For I2C, scan and verify the expected address if possible.
- For I2C, `scan()` must only compare against the 7-bit expected address; if the datasheet evidence is `0x3C/0x3D`, the debug driver should still check for `0x1E`.
- For SPI, verify CS handling and read a known register or perform a safe read-back.
- For UART, send a known command like `AT` where applicable and verify the response.
- If an ID register exists, read and compare against the expected value.
- If no ID register exists, use a safe register read/write sanity check as a substitute.
- Mark write-only registers as `write-only` and skip read-back.
- When the datasheet provides a ready signal, prefer ready/status-bit polling with timeout over fixed sleeps.
- Only use fixed sleeps when there is no ready signal; delays should include conversion time plus margin.
- Must verify CRC/checksum when provided by the chip.
- On failure, print expected vs actual values.
- On failure, print wiring/power/protocol hints.
- Catch underlying `OSError` and raise or print descriptive context including address, register, and operation.
- Every wait/poll loop must be bounded by `ticks_ms()`/`ticks_diff()` or a fixed iteration count.
- Pre-allocate bytearrays for repeated bus I/O where possible.
- End with `SELF_TEST_PASS` on successful self-test; otherwise print `SELF_TEST_FAIL: <reason>`.

## Hardware Verification Gate

Hardware verification is the normal gate before generating a production driver.

Plugin-mode behavior:

- Before scanning ports or devices, request `permission_request(device_scan)`.
- Before running `run_on_device.py` or `mpremote`, request `permission_request(device_run)`.
- If the host lacks `serial_port_scan`, `device_command`, or `mpremote_run` capability, return a `HOST_CAPABILITY_MISSING` partial, do not fake a device scan or write `DEVICE_NOT_FOUND`. If device scan has been authorized and executed but the target device is not found, return `DEVICE_NOT_FOUND`.
- Use `scripts/run_on_device.py --com <port> --file <debug.py> --capture --timeout-ms <ms> --json-summary`.
- Repair verification is limited to 10 rounds.
- Each round's run log is saved as `gen_driver/logs/driver_verify_round<N>.log`.
- On `SELF_TEST_PASS`, checkpoint to `hardware_verify_passed`.
- On no device, timeout, permission denial, or user cancellation, output a partial with a resumable checkpoint.
- When there is no device or verification has not passed, do not mark `{chip}.py` as `production_driver` in `file_manifest.files[]`, unless the user explicitly skips verification and the output includes a warning and skip metadata.
- A partial result with no device, timeout, permission denial, or user cancellation must write `hardware_verified=false`, `verification_mode="none"`, `next_phase=null`, and point `resume_step` to the next executable verification step.
- If a UI/CLI table displays an unverified `{chip}.py`, the Role must show `Driver artifact (unverified)` or `Unverified driver artifact`; do not show `Production driver (unverified)`.

Production driver generation is only allowed when one of the following conditions is met:

| Condition | Required output |
|---|---|
| Real hardware verification passed | `hardware_verified=true`, no warning required. |
| User explicitly skipped hardware verification | `hardware_verified=false`, must include a warning artifact and structured note. |
| Local mock returned `SELF_TEST_PASS` | Marked as mock verification; do not claim real hardware proof. |

Do not silently jump from debug generation to production success just because a device is unavailable.

## Production Driver Requirements

Only generate `project/firmware/drivers/<chip>_driver/<chip>.py` after passing the hardware gate above.

Production driver rules inherited from `upy-gen-driver`:

- Remove debug banners and step-by-step prints.
- Concise diagnostic methods like `_self_test()` or `scan()` may be retained, but should not run by default.
- Retain meaningful exception messages with register/address/action context.
- Code organization order: constants, class, `__init__`, public methods, private helpers, and `deinit()`.
- Maintain I2C/SPI/UART dependency injection.
- I2C address constants must be 7-bit addresses, e.g., `_I2C_ADDR = const(0x1E)`; do not use `_I2C_ADDR_WRITE = const(0x3C)` or `_I2C_ADDR_READ = const(0x3D)` as actual API call addresses.
- I2C drivers must accept `machine.I2C`, `SoftI2C`, or compatible objects via duck typing; do not reject `SoftI2C` with a strict `isinstance(i2c, I2C)` check.
- After generating `{chip}.py`, `{chip}_debug.py`, and `test_{chip}.py`, a static quality check must be performed: Python syntax, undefined constants/names, helper method call argument count, consistency between I2C capability check and actual I/O API usage.
- Do not let constant naming conventions drift between the debug driver and the production driver; if the debug driver uses `_ODR_10HZ` / `_MD_IDLE`, the production driver must either define the same constants or change them all to `ODR_10HZ` / `MODE_IDLE` and synchronize all references.
- Helper method signatures must cover all call forms; for example, if the code calls `_read_reg(reg, buf)`, the definition must be `def _read_reg(self, reg, buf=None)` or equivalent.
- The I2C constructor capability check must cover the methods actually used; if a helper calls `readfrom_mem_into`, do not only check for `readfrom_mem`.
- Validate argument types and ranges in `__init__`.
- Bring the chip into a known state in `__init__`.
- Track shadow state independently per setter; `set_gain()` must not modify `_vref`, and `set_vref()` must not modify `_gain`.
- When feasible, update shadow state only after a successful hardware write.
- Implement `deinit()` when the datasheet supports standby/powerdown.
- Device code must not depend on CPython-only modules.
- Avoid dynamic allocation in hot read loops where possible.
- Datasheet page/table comments are only for explaining constants, timing, formulas, or register behavior; do not write tutorials.

Then run the P0 normalization checklist from `references/norm_driver_p0_rules.md`, and validate real file content with `scripts/validate_phase_complete.py --artifact-root <session_root> --session-state <state_file>`. Do not output a result that can be integrated if the validator fails.

## Checkpoints

Use these stable checkpoint names:

`started`, `input_collected`, `source_preprocessed`, `understanding_written`, `debug_driver_written`, `hardware_verify_ready`, `hardware_verify_passed`, `production_driver_written`, `normalized`, `standalone_assets_written`, `standalone_test_passed`, `manifest_updated`, `phase_completed`, `cancelled`, `verification_exhausted`.

`phase_complete.payload.checkpoint.checkpoint_id` must use `upy-gen-driver-plugin:<session_id>:<checkpoint_name>`. For example, `upy-gen-driver-plugin:8234517f-d65a-4620-a391-3936c7c9eda4:hardware_verify_ready`; do not write only `upy-gen-driver-plugin:hardware_verify_ready`.

Maintain state using the following commands:

```bash
python scripts/update_session_state.py --session-dir <session_root> --session-id <session_id> --checkpoint <name> --step <step> --status running --idempotency-key <key>
python scripts/update_session_state.py --session-dir <session_root> --check
```

Resume rules:

- Require consistency of `session_id`, `phase`, `protocol_version`, and checkpoint name.
- The current checkpoint in `session_state.upy_gen_driver_plugin.json` must match the checkpoint part of `phase_complete.payload.checkpoint.checkpoint_id`; partial results especially must not have the state written as `phase_completed`.
- If a hash exists, verify that the last trusted artifact exists and its hash matches.
- If the manifest hash changes after the checkpoint, return `ARTIFACT_STALE`, or fall back to a previous safe checkpoint.
- Do not re-execute a completed write when the target file hash already matches.
- Keep `verify_round < max_verify_rounds`; on exhaustion, checkpoint to `verification_exhausted` and return partial.

## Retry, Cancellation, and Timeout

Use the following result forms:

| Event | State update | Final or next message |
|---|---|---|
| User retry | `status="retrying"`, same checkpoint, same action idempotency key, `retry_of=<msg_id>` | Re-issue the action request, or continue from checkpoint |
| LLM repair retry | Increment `verify_round`, keep session, write verify log | Continue until pass or `verification_exhausted` |
| User cancellation | `status="cancelled"`, checkpoint `cancelled` | `phase_complete.result="partial"`, with `CANCELLED_BY_USER` |
| Approval timeout | `status="partial"`, revert to previous safe checkpoint | `APPROVAL_TIMEOUT` or `DEVICE_RUN_TIMEOUT` structured error |
| Script timeout | Set `status="partial"` or `retrying` based on policy | `SOURCE_PREPROCESS_TIMEOUT` or `DEVICE_RUN_TIMEOUT` |
| Capability missing | `status="partial"` | `HOST_CAPABILITY_MISSING`, with missing capability in details |

Timeout defaults:

- approval/input card: `300000`
- PDF extraction: `30000`
- Arduino conversion: `15000`
- GitHub/datasheet fetch: `30000`
- device scan: `5000`
- device debug run: host `60000`, device script `30000`
- standalone test: host `30000`, device script `15000`

## Structured Errors

Use stable codes from `references/protocol_fields.md`. Important codes include `MISSING_INPUT_SOURCE`, `HOST_CAPABILITY_MISSING`, `PERMISSION_DENIED`, `SOURCE_PREPROCESS_FAILED`, `SOURCE_PREPROCESS_TIMEOUT`, `DEVICE_NOT_FOUND`, `DEVICE_RUN_TIMEOUT`, `HARDWARE_VERIFY_FAILED`, `HARDWARE_VERIFY_EXHAUSTED`, `STANDALONE_TEST_FAILED`, `MANIFEST_UPDATE_CONFLICT`, `ARTIFACT_STALE`, and `CANCELLED_BY_USER`.

Each error must include `code`, `severity`, `phase_step`, `retryable`, `message`, `details`, and `next_action`.

## Required Artifacts

On success, if the corresponding files have been generated, the following must be written into `payload.file_manifest.files[]`:

- `gen_driver/docs/driver_understanding.json`
- `project/firmware/drivers/<chip>_driver/<chip>_debug.py`
- `project/firmware/drivers/<chip>_driver/<chip>.py`
- `project/firmware/drivers/<chip>_driver/test_<chip>.py`
- `project/firmware/drivers/<chip>_driver/wiring_<chip>.md`
- `gen_driver/logs/driver_verify_round<N>.log` or an explicit skip-verification artifact
- `session_state.upy_gen_driver_plugin.json`
- `project/project-manifest.json` in `pipeline` mode

`phase_complete.upy_gen_driver_plugin.json` is the final protocol envelope and is not required to be placed in its own `payload.file_manifest.files[]`. If auditing is needed, it should be recorded by the host or an external sidecar manifest; do not force it into its own file manifest for self-referential hashing.

On partial, the last trusted artifact and a checkpoint from which to resume must be included.

Every existing or generated file in `file_manifest.files[]` must include a real `sha256` and `bytes`. Do not use `"hash": "unverified"` or other placeholder fields instead of `sha256`.
`payload.artifacts[]` must contain a non-empty `file_list` whose `files[]` or `items[]` lists at least the trusted files produced or retained in this run; do not provide only `title`, `label`, or an empty array.

## Local Mock Testing

Local tests can write files, but must also write protocol artifacts under `sessions/<session_id>/gen_driver/`. Use:

```bash
python test/smoke_tests.py
python test/run_local_mock_session.py --mode standalone --scenario no_device
python test/run_local_mock_session.py --mode standalone --scenario cancelled
python test/run_local_mock_session.py --mode standalone --scenario timeout
python test/run_local_mock_session.py --mode standalone --scenario retry_success
python scripts/validate_phase_complete.py --input sample/phase_complete.upy_gen_driver_plugin.partial.no_device.json
```

Do not treat mock outputs as real hardware verification proof. `no_device`, `cancelled`, and `timeout` must all use `verification_mode="none"`; only success paths like `retry_success` where a local mock self-test actually passed may use `verification_mode="mock"`, and must include a `MOCK_VERIFICATION_ONLY` warning.

Minimum local coverage:

- `no_device`: partial checkpoint at `hardware_verify_ready`, `DEVICE_NOT_FOUND`.
- `missing_device_capability`: partial checkpoint at `hardware_verify_ready`, `HOST_CAPABILITY_MISSING`, `details.missing_capability` pointing to the missing capability.
- `cancelled`: partial checkpoint `cancelled`, `CANCELLED_BY_USER`.
- `timeout`: partial checkpoint at `hardware_verify_ready`, `DEVICE_RUN_TIMEOUT`.
- `retry_success`: first device run times out, retry keeps the same session, and produces success with `retry_of`.
- idempotency: re-running the same action must not duplicate file manifest entries or overwrite files whose hash already matches.
