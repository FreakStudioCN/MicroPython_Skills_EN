---
name: upy-analyze-plugin
description: Plugin-based V0 analyze phase. Reads a one-sentence hardware project requirement and plugin context, completes requirement parsing, device confirmation, driver search, alternative recommendations or cold-driver marking, and outputs a complete envelope with phase_complete + manifest_content. Triggered by: plugin start_phase(analyze), user description "make a / I want to / help me write a" MicroPython hardware project, or when a project manifest needs to be generated.
---

# upy-analyze

## Responsibilities

Converts a user's one-sentence hardware requirement into an analyze manifest that can be handed over to `upy-select-hw`.

Only does:

- Parse requirements and implementation families.
- Generate and confirm a device list.
- Search for built-in runtime capabilities and specific device drivers.
- Mark alternative recommendations or cold-driver paths.
- Output `phase_complete`, where `payload.manifest_content` is the single primary handover item for downstream.

Does not:

- Select MCU or board.
- Assign pins.
- Generate business code.
- Flash devices.
- Write plugin-side UI or device log parsing logic into the plugin.

## Operation Modes

## Protocol Field Description

First, follow the execution flow of this file. When constructing or troubleshooting specific message fields, read `references/v0-protocol.md`; it defines the field meanings and enums for envelope, `start_phase`, `approval_request`, `status_update`, `script_run`, manifest, `phase_complete`, checkpoint, structured errors, and artifacts.

When outputting JSON, prioritize the shapes in `templates/*.json` and `mock-messages/analyze/*.json`; do not invent field names.

### Formal Plugin Mode

The plugin starts via `start_phase`:

```json
{
  "protocol_version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440000",
  "session_id": "4f6d9d72-9c4a-4f11-90df-3f2ad6e726cc",
  "phase": "analyze",
  "timestamp": "2026-06-21T00:00:00Z",
  "type": "start_phase",
  "payload": {
    "user_description": "Make a temperature and humidity monitor, buzzer alarm when threshold exceeded",
    "pre_selected_board": null,
    "preferences": { "mode": "beginner", "locale": "zh" },
    "existing_hardware": []
  }
}
```

In formal mode:

- `session_id` must be created and passed by the plugin.
- The skill/server must inherit the same `session_id`; do not create a separate formal session.
- All S->P messages must carry a complete envelope.
- Local files, scripts, and device actions can only be expressed through protocol tools.

`start_phase` field quick reference:

| Field | Required | Source | Meaning |
|-------|----------|--------|---------|
| `protocol_version` | Yes | Plugin | Fixed `"1.0"` |
| `msg_id` | Yes | Plugin | UUID for the current message |
| `session_id` | Yes | Plugin | UUID for the current workflow session, remains unchanged throughout the process |
| `phase` | Yes | Plugin | Fixed `"analyze"` |
| `timestamp` | Yes | Plugin | ISO 8601 timestamp |
| `type` | Yes | Plugin | Fixed `"start_phase"` |
| `payload.user_description` | Yes | User input | One-sentence hardware requirement |
| `payload.pre_selected_board` | No | Plugin UI | Pre-selected board; analyze only records, does not verify |
| `payload.preferences.mode` | No | Plugin settings | `beginner` or `custom`, default `beginner` |
| `payload.preferences.locale` | No | Plugin settings | Default `zh` |
| `payload.existing_hardware` | No | User profile | Array of existing hardware, default `[]` |

### Claude Code Direct Test Mode

When there is no real plugin host, debug artifacts can be written, but these files do not replace `phase_complete.payload.manifest_content`.

If the input lacks `session_id`, the direct test mode must generate a UUID and force the use of a session-isolated directory:

```text
{test_root}/sessions/{session_id}/
  manifest_draft.json
  manifest_validated.json
  phase_complete.analyze.json
  driver_search_log.md
  analyze_phase_log.md
```

Direct test mode must call the validation script before finishing:

```bash
python {skill_dir}/scripts/init_manifest.py --input {session_dir}/manifest_draft.json --write-path {session_dir}/manifest_validated.json
python {skill_dir}/scripts/init_manifest.py --validate-phase-complete --input {session_dir}/phase_complete.analyze.json --compare-manifest {session_dir}/manifest_validated.json
```

If either validation fails, analyze must not be declared successful.

## V0 Protocol Hard Rules

### Complete Envelope

All formal protocol messages must include:

```json
{
  "protocol_version": "1.0",
  "msg_id": "uuid",
  "session_id": "uuid",
  "phase": "analyze",
  "timestamp": "2026-06-21T00:00:00Z",
  "type": "phase_complete",
  "payload": {}
}
```

Requirements:

- `protocol_version` is fixed to `"1.0"`.
- `msg_id` uses a UUID string.
- `session_id` uses a UUID string.
- Both the top-level `phase` and `payload.phase` are retained and must be consistent.

Envelope field quick reference:

| Field | Required | Generated By | Rule |
|-------|----------|--------------|------|
| `protocol_version` | Yes | Sender | Fixed `"1.0"` |
| `msg_id` | Yes | Sender | A new UUID for each message |
| `session_id` | Yes | Plugin | Unchanged for the same workflow |
| `phase` | Yes | Sender | Fixed to `"analyze"` during the analyze phase |
| `timestamp` | Yes | Sender | ISO 8601, UTC preferred |
| `type` | Yes | Sender | Message type |
| `payload` | Yes | Sender | Type-specific object |

### result Enum

`phase_complete.payload.result` only allows:

| result | Meaning | next_phase | checkpoint |
|--------|---------|------------|------------|
| `success` | Analyze fully successful, can proceed downstream | `select-hw` | Not required |
| `partial` | User cancelled, interrupted, timed out, missing input, or only partially completed search | `null` | Required |
| `failed` | Cannot produce a usable manifest, or protocol/format validation failed | `null` | Optional |

`partial` must include:

```json
{
  "checkpoint_id": "uuid",
  "resume_phase": "analyze",
  "resume_step": "driver_search",
  "resume_label": "Continue analyze driver search",
  "reason": "user_cancelled"
}
```

V0 only defines the checkpoint/resume structure, does not implement a full resume runtime.

### errors and structured_errors

Keep `errors: string[]` for human reading, while also outputting `structured_errors: object[]` for the plugin UI and orchestration:

```json
{
  "code": "manifest_validation_failed",
  "message": "devices[0].driver.source invalid",
  "severity": "error",
  "recoverable": true,
  "retryable": true,
  "source": "init_manifest.py"
}
```

`severity` only allows `info / warning / error / fatal`.

### Unified Artifact Model

`artifacts` must be an array. Debug file paths use the `file_list` artifact; do not write them as an object mapping.

`artifact.files[].status` only allows:

```text
created / updated / unchanged / skipped / error
```

Recommended file item:

```json
{
  "path": "manifest_validated.json",
  "status": "created",
  "kind": "manifest",
  "mime_type": "application/json",
  "description": "Validated and normalized analyze manifest"
}
```

`artifact_id` is not mandatory. `kind` and `description` are recommended to fill; the validation script may give a warning if missing.

## Permission Strategy

Adopt a long-flow strategy where "the first session prompts for overall permission once, and subsequent sessions reuse it."

After authorization in the analyze phase, the following are allowed:

- Write project analysis artifacts.
- Run the whitelisted script `scripts/init_manifest.py`.
- Access driver search sources, such as upypi, awesome-micropython, GitHub.

High-risk actions that still require separate confirmation:

- Deleting files.
- Flashing devices.
- Executing arbitrary shell commands.
- Uploading or publishing to upypi.

## Cancellation, Retry, Timeout

V0 writes these into the protocol and skill description first, but does not implement a full runtime:

- User cancels approval: output `result="partial"`, `next_phase=null`, write checkpoint.
- Driver search timeout: first degrade to warning; only fail if core information cannot be determined.
- Manifest validation failure: allow correction and retry; retry uses the same `session_id`.
- Retry behavior is recorded in logs or payload metadata.

## Execution Steps

### Step 1: Read Input Context

Read `start_phase.payload`:

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `user_description` | Yes | None | User's one-sentence requirement |
| `pre_selected_board` | No | `null` | Plugin pre-selected board; analyze only records, does not verify |
| `preferences.mode` | No | `beginner` | `beginner` or `custom` |
| `preferences.locale` | No | `zh` | Default Chinese |
| `existing_hardware` | No | `[]` | User's existing hardware |

If a field is missing, fill in with the default value; if `user_description` is missing or empty, output `phase_complete(result="failed")` and do not continue guessing the requirement.

Send:

```json
{
  "type": "status_update",
  "payload": {
    "level": "info",
    "message": "Analyzing requirements, first breaking down implementation families and device list.",
    "step_id": "intent_extraction",
    "step_status": "running"
  }
}
```

### Step 2: Intent Decomposition and Device Confirmation

Extract from natural language:

- Project name.
- Functional chain.
- Implementation families.
- Device list.
- Interface types.
- User-specified devices vs. system-recommended devices.

Major device categories must first break down implementation families. For example, soil-type devices must distinguish between `ADC / RS485 Modbus / I2C/SPI / Combined solution`.

Only one mandatory confirmation point is retained: `approval_request(device_confirm)`.

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "device_confirm",
    "header": "Confirm Project Plan",
    "question": "Please confirm the device plan; for soil-type devices, you can change to ADC / RS485 Modbus / I2C plan here.",
    "summary": {
      "project_name": "Temperature and Humidity Monitoring Alarm",
      "description": "Periodically collect temperature and humidity, buzzer alarm when threshold exceeded",
      "board": { "status": "none" }
    },
    "items": [],
    "allow_add": true,
    "allow_remove": true,
    "multi_select": true,
    "actions": [
      { "label": "Confirm, start driver search", "value": "confirm", "primary": true },
      { "label": "Modify device list", "value": "modify" }
    ]
  }
}
```

After sending `approval_request`, you must wait for the user's response; do not continue pretending it has been confirmed.

### Step 3: Supplement Requirements

`beginner` mode fills in requirements by default. For `custom` mode or when information is clearly insufficient, send at most one `approval_request(requirement_supplement)`.

Default values:

| Field | Default |
|-------|---------|
| `scene` | `indoor` |
| `power` | `usb` |
| `network` | `none` |
| `sample_rate` | `normal_1hz` |
| `precision` | `normal` |
| `response_time` | `1s` |
| `temp_range` | `normal_0_40` |
| `size_constraint` | `none` |
| `budget_yuan` | `medium_50` |
| `experience` | `beginner` |
| `output` | `["serial"]` |
| `existing_hardware` | `[]` |
| `special_requirements` | `["none"]` |
| `mcu_specified` | `null` |

Content that cannot be fully expressed by schemas like voice, cloud, audio output, etc., should be recorded in `description`, `special_requirements`, device notes, and warnings. Do not fail directly due to insufficient output enum values.

### Step 4: Driver Search

For each confirmed device, judge on two levels:

1. Underlying runtime capabilities:
   - `machine.ADC`
   - `machine.Pin`
   - `machine.I2C`
   - `machine.SPI`
   - `machine.UART`
   - `machine.I2S`
   - `network`
   - `bluetooth`

2. Specific device drivers:
   - `upypi`
   - `awesome-micropython`
   - `github`
   - Other trusted MicroPython sources

Note:

- `builtin_runtime` only indicates that the underlying API is available; it does not mean a specific I2C/SPI/UART device driver has been found.
- For specific I2C/SPI/UART devices, `upypi` should still be checked first.
- `micropython_lib` is only used for official ecosystem general-purpose libraries/middleware, not as a default source for common sensor drivers.
- `driver.source="none"` is only used when it is clearly not a built-in runtime capability and all driver sources yield no results.

Send a `status_update` for each device search process.

If a system-recommended device has no driver, recommend at most 2 similar alternative devices using `approval_request(alternative_device)`. If a user-specified device has no driver, or the user rejects the alternative, mark it as `driver.source="cold-driver"` and let the subsequent `upy-gen-driver` handle it.

### Step 5: Build manifest_draft

Generate a manifest draft, which must include:

- `project_name`
- `requirements`
- `devices`

Each device must include:

- `name`
- `type`
- `interface`
- `source`: `user_specified` or `system_recommended`
- `quantity`
- `driver.source`

Valid `driver.source` values:

```text
builtin_runtime / micropython_lib / upypi / awesome-micropython / github / local / cold-driver / none
```

### Step 6: Mandatory Manifest Validation

Must call:

```bash
python {skill_dir}/scripts/init_manifest.py --input manifest_draft.json --write-path manifest_validated.json
```

Validation failure:

- Can retry after correcting the draft.
- If it still fails, output `phase_complete(result="failed")`.
- Do not continue to output `success`.

### Step 7: Output phase_complete

On success, output a complete envelope:

```json
{
  "protocol_version": "1.0",
  "msg_id": "550e8400-e29b-41d4-a716-446655440001",
  "session_id": "4f6d9d72-9c4a-4f11-90df-3f2ad6e726cc",
  "phase": "analyze",
  "timestamp": "2026-06-21T00:00:00Z",
  "type": "phase_complete",
  "payload": {
    "phase": "analyze",
    "result": "success",
    "summary": "Device analysis complete, manifest has passed validation.",
    "next_phase": "select-hw",
    "manifest_content": {},
    "artifacts": [
      {
        "type": "file_list",
        "title": "Claude Code Direct Test Artifacts",
        "files": [
          {
            "path": "manifest_draft.json",
            "status": "created",
            "kind": "manifest_draft",
            "mime_type": "application/json",
            "description": "Manifest draft before validation"
          },
          {
            "path": "manifest_validated.json",
            "status": "created",
            "kind": "manifest",
            "mime_type": "application/json",
            "description": "Validated and normalized analyze manifest"
          },
          {
            "path": "phase_complete.analyze.json",
            "status": "created",
            "kind": "phase_complete",
            "mime_type": "application/json",
            "description": "Complete analyze phase completion message"
          },
          {
            "path": "driver_search_log.md",
            "status": "created",
            "kind": "log",
            "mime_type": "text/markdown",
            "description": "Driver search record"
          }
        ]
      }
    ],
    "warnings": [],
    "errors": [],
    "structured_errors": []
  }
}
```

`phase_complete.payload` field quick reference:

| Field | Required | success | partial | failed |
|-------|----------|---------|---------|--------|
| `phase` | Yes | `"analyze"` | `"analyze"` | `"analyze"` |
| `result` | Yes | `"success"` | `"partial"` | `"failed"` |
| `summary` | Yes | Success summary | Interruption summary | Failure summary |
| `next_phase` | Yes | `"select-hw"` | `null` | `null` |
| `manifest_content` | Yes | Validated manifest | Best current manifest snapshot | Provide current snapshot if possible |
| `checkpoint` | Conditional | Not required | Required | Optional |
| `artifacts` | Yes | Array | Array | Array |
| `warnings` | Yes | String array | String array | String array |
| `errors` | Yes | Empty array or error summary | Empty array or error summary | Error summary |
| `structured_errors` | Yes | Empty array | Optional structured errors | Must describe primary failure |

Direct test mode recommends additionally writing `analyze_phase_log.md`, but it is not a mandatory deliverable of the formal protocol; it can be declared in the `file_list`.

After writing `phase_complete.analyze.json`, must call:

```bash
python {skill_dir}/scripts/init_manifest.py --validate-phase-complete --input phase_complete.analyze.json --compare-manifest manifest_validated.json
```

If validation fails, do not declare completion.

## Deliverable Files

Formal plugin mode relies on messages. Claude Code direct test mode writes the following in the session directory:

- `manifest_draft.json`
- `manifest_validated.json`
- `phase_complete.analyze.json`
- `driver_search_log.md`
- `analyze_phase_log.md` (recommended)

## Templates and Mocks

Use resources provided by this skill:

- `templates/envelope.phase_complete.json`
- `templates/checkpoint.json`
- `templates/structured_error.json`
- `templates/artifact.file_list.json`
- `mock-messages/analyze/*.json`
- `references/v0-protocol.md`

After modifying templates, enums, or output formats, the validation scripts and smoke tests must be updated.

## Hard Constraints

- Protocol format errors, missing required fields, invalid enums, and core manifest structure errors must be treated as errors.
- Business semantic issues should be warnings first, e.g., TouchPad board compatibility, incomplete voice output schema.
- `phase_complete.payload.manifest_content` is the single primary handover item for downstream.
- `manifest_validated.json` and `phase_complete.payload.manifest_content` must have consistent core fields; timestamp fields are not strictly compared.
- `phase_complete.artifacts` must be an array.
- `errors` must be a string array, `structured_errors` must be an object array.
- `partial` must have `next_phase=null` and include a checkpoint.
- `success` must have `next_phase="select-hw"` and a valid `manifest_content`.
