# Communication Protocol

## Protocol Basics

- Transport: HTTP POST (plugin→server), SSE (server→plugin)
- Format: JSON
- Encoding: UTF-8
- Message Direction: `S→P` = server to plugin, `P→S` = plugin to server

All messages share a common outer envelope. The `type` field determines the message category, and the `phase` field identifies the current pipeline stage.

## Message Envelope

```json
{
  "msg_id": "uuid",
  "session_id": "uuid",
  "phase": "analyze",
  "timestamp": "2026-06-16T10:30:00Z",
  "type": "approval_request",
  "payload": { ... }
}
```

| Field | Type | Direction | Description |
|-------|------|-----------|-------------|
| `msg_id` | string | Bidirectional | Unique message ID, used to correlate request/response |
| `session_id` | string | Bidirectional | Project session ID, generated on first plugin send |
| `phase` | string | Bidirectional | Current pipeline stage (analyze/select-hw/scaffold/generate/simulate/deploy/autofix/wiring/diagram/cold-driver) |
| `timestamp` | string | Bidirectional | ISO 8601 timestamp |
| `type` | string | Bidirectional | Message type, see table below |
| `payload` | object | Bidirectional | Message payload, structure determined by type |

---

## I. S→P Messages (Server to Plugin, 7 types)

### 1. approval_request — Requires User Approval

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "device_confirm_001",
    "header": "Confirm Project Plan",
    "question": "Are the following devices correct?",
    "summary": {
      "project_name": "Temperature and Humidity Monitoring Alarm",
      "board": { "display_name": "ESP32 DevKit V1", "mcu": "ESP32-WROOM-32" }
    },
    "items": [
      {
        "id": "d1",
        "name": "SHT30",
        "subtitle": "I2C Temperature and Humidity Sensor",
        "meta": "User Specified",
        "selectable": true,
        "selected": true
      },
      {
        "id": "d2",
        "name": "SSD1306 OLED",
        "subtitle": "I2C Display (0x3C)",
        "meta": "System Recommended",
        "selectable": true,
        "selected": true
      }
    ],
    "allow_add": true,
    "allow_remove": true,
    "actions": [
      { "label": "Confirm, Start Searching for Drivers", "value": "confirm", "primary": true },
      { "label": "Modify Device List", "value": "modify" }
    ],
    "multi_select": true
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `approval_id` | string | Approval ID, returned as-is in the response |
| `header` | string | Card title |
| `question` | string | Main question |
| `summary` | object | Optional, top summary area. Can contain `project_name`, `board`, `description` |
| `items` | array | Optional, list of options. Each item contains `id`/`name`/`subtitle`/`meta`/`selectable`/`selected` |
| `allow_add` | boolean | Whether to allow users to add new items |
| `allow_remove` | boolean | Whether to allow users to delete items |
| `actions` | array | Action buttons. `label`=display text, `value`=return value, `primary`=highlight |
| `multi_select` | boolean | Whether multiple selection is allowed |
| `item_groups` | array | Optional, grouped single/multiple selection. Each group contains `group_id`/`group_header`/`multi_select`/`items`, items structure same as top-level items |
| `file_upload` | object | Optional, enable file upload. Sub-fields: `enabled`(bool), `accept`(string[]), `max_files`(int), `max_size_mb`(int), `generate_thumbnails`(bool), `thumbnail_size`([int,int]), `preprocess`(object, extension→preprocessing script) |
| `text_inputs` | array | Optional, text input fields. Each item contains `id`/`label`/`placeholder`/`type`("text"\|"url"\|"number") |
| `guidance` | object | Optional, debugging guidance. Contains `tool`(required tool name), `steps`(string[]), `normal_range`(normal range), `diagram_ref`(reference diagram) |

### 2. status_update — Progress Update

```json
{
  "type": "status_update",
  "payload": {
    "level": "info",
    "message": "Searching for SSD1306 driver...",
    "progress": 0.25,
    "progress_label": "1/4",
    "step_id": "search_drivers",
    "step_status": "running",
    "detail": "Searched upypi, found 1 matching package"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `level` | string | `info` / `warn` / `error` / `success` |
| `message` | string | Required, brief description |
| `progress` | number | Optional, 0.0~1.0 |
| `progress_label` | string | Optional, progress text like "2/5" |
| `step_id` | string | Optional, step identifier, shared across multiple updates for the same step |
| `step_status` | string | Optional, `pending` / `running` / `done` / `failed` |
| `detail` | string | Optional, supplementary explanation |

Rendered by the plugin as a timeline list: completed (✓) / in progress (spinning) / failed (✗) + details.

### 3. device_command — Transparent mpremote

```json
{
  "type": "device_command",
  "payload": {
    "cmd_id": "dc_042",
    "action": "exec",
    "code": "import machine; i2c=machine.I2C(0); print(i2c.scan())",
    "timeout_ms": 5000,
    "expect_output": true
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `cmd_id` | string | Command ID, returned as-is in the response |
| `action` | string | `devs`(list available serial ports) / `scan`(scan I2C etc. buses) / `exec`(execute code) / `cp`(upload file) / `cp_from`(download file) / `mkdir`(create directory) / `ls`(list files) / `rm`(delete file) / `soft_reset`(soft reset) / `stream`(persistent session) / `run`(send .py file to REPL for execution) |
| `code` | string | Required when action=exec, Python code to execute |
| `src` | string | Required when action=cp, local source path (relative to project directory) |
| `dst` | string | Required when action=cp/mkdir/rm/ls, remote path |
| `timeout_ms` | number | Timeout in milliseconds, default 30000 |
| `expect_output` | boolean | Whether to wait for output, default true |

### 4. file_operation — File Read/Write

```json
{
  "type": "file_operation",
  "payload": {
    "op_id": "fo_015",
    "op": "write",
    "path": "firmware/tasks/sensor_task.py",
    "content": "# MicroPython sensor task\n...",
    "encoding": "utf-8"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `op_id` | string | Operation ID |
| `op` | string | `write`(overwrite, auto-create parent directories) / `read` / `list` / `delete` / `mkdir`(ensure directory exists) / `append`(append write) |
| `path` | string | File path relative to the project directory |
| `content` | string | Required when op=write, file content |
| `encoding` | string | Encoding, default utf-8 |

### 5. script_run — Execute Local Script

```json
{
  "type": "script_run",
  "payload": {
    "script_id": "sr_008",
    "interpreter": "python",
    "script": "flake8",
    "args": ["--max-line-length=100", "firmware/tasks/"],
    "cwd": "{project_dir}",
    "timeout_ms": 30000
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `script_id` | string | Script ID |
| `interpreter` | string | `python` / `node` / `shell` |
| `script` | string | Script name or command |
| `args` | string[] | Command line arguments |
| `cwd` | string | Working directory. `{project_dir}` = project root; `{skill_dir}` = skill script directory. default = `{project_dir}` |
| `timeout_ms` | number | Timeout in milliseconds |

### 6. phase_complete — Phase Complete

```json
{
  "type": "phase_complete",
  "payload": {
    "result": "success",
    "summary": "Device analysis complete, found drivers for 2 out of 3 devices",
    "next_phase": "select-hw",
    "artifacts": [
      {
        "type": "table",
        "title": "Device List",
        "headers": ["Device", "Type", "Interface", "Driver Source", "Status"],
        "rows": [
          ["SSD1306", "OLED", "I2C", "upypi", "✓"],
          ["SHT30", "Temp/Humidity", "I2C", "none", "⚠ Cold Hardware Path"]
        ]
      },
      {
        "type": "file_tree",
        "title": "Project Structure",
        "tree": { "firmware": { "main.py": "file", "tasks": { "sensor.py": "file" } } }
      }
    ],
    "warnings": ["SHT30 driver not found, will use cold hardware path"],
    "errors": []
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `result` | string | `success` / `failed` / `partial` |
| `summary` | string | Required, human-readable summary |
| `next_phase` | string | Optional, name of the next phase. Upon receipt, the plugin automatically triggers the corresponding skill's `start_phase`; `manifest` is taken from this message's `manifest_content` |
| `manifest_content` | object | Optional, complete snapshot of the current manifest. The downstream skill's `start_phase.payload.manifest` is automatically populated from this field |
| `artifacts` | array | List of artifacts, types in table below |
| `warnings` | string[] | Warning messages |
| `errors` | string[] | Error messages |

**Artifact types:**

| type | Rendering Method | Additional Fields |
|------|------------------|-------------------|
| `table` | Table | `headers` + `rows` |
| `file_tree` | Tree directory | `tree` (nested object) |
| `markdown` | Markdown rendering | `content` |
| `html` | iframe preview | `content` / `url` |
| `code_diff` | diff view | `file_path` + `changes[{line_start, line_end, old_text, new_text}]` |
| `file_list` | File list | `files: [{path, size, status}]` |

### 7. stream — Real-time Data Stream

```json
{
  "type": "stream",
  "payload": {
    "stream_id": "device_repl_001",
    "stream_type": "device_output",
    "chunk": "[0.5s] I2C scan result: [48, 60]\n",
    "chunk_index": 12,
    "done": false
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `stream_id` | string | Stream ID |
| `stream_type` | string | `device_output` / `script_stdout` / `script_stderr` |
| `chunk` | string | Data chunk |
| `chunk_index` | number | Sequence number (starting from 0) |
| `done` | boolean | Whether the stream is finished |

---

## II. P→S Messages (Plugin to Server)

7 types total: 5 basic messages + 2 autofix-specific messages.

### 0. start_phase — Start Skill

```json
{
  "type": "start_phase",
  "phase": "analyze",
  "payload": {
    "manifest": { },
    "source": null
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `phase` | string | Target skill name (analyze / select-hw / scaffold / generate / simulate / deploy / autofix / wiring / diagram / gen-driver / publish) |
| `payload` | object | Input data defined by each skill. Common fields: `manifest` (project list passed from upstream), `session_id` (carried when resuming a session). Other fields are defined by each skill's interface documentation |

`start_phase` is the first message from plugin to server, triggering the server to load the corresponding SKILL.md and begin execution. `manifest` is passed via `phase_complete.manifest_content` from the upstream skill; `session_id` is used to resume a previously interrupted session (e.g., gen-driver's "continue later" scenario).

### 1. approval_response

```json
{
  "type": "approval_response",
  "payload": {
    "approval_id": "device_confirm_001",
    "action": "confirm",
    "selected_ids": ["d1", "d2"],
    "added_items": [],
    "notes": ""
  }
}
```

### 2. device_result

```json
{
  "type": "device_result",
  "payload": {
    "cmd_id": "dc_042",
    "success": true,
    "stdout": "[48, 60]\n",
    "stderr": "",
    "exit_code": 0
  }
}
```

### 3. script_result

```json
{
  "type": "script_result",
  "payload": {
    "script_id": "sr_008",
    "success": false,
    "stdout": "firmware/tasks/sensor.py:15:1: E302 expected 2 blank lines\n",
    "stderr": "",
    "exit_code": 1
  }
}
```

### 4. file_result

```json
{
  "type": "file_result",
  "payload": {
    "op_id": "fo_015",
    "success": true,
    "error": null
  }
}
```

### 5. user_intervention — User Intervention During Troubleshooting (autofix specific)

```json
{
  "type": "user_intervention",
  "payload": {
    "approval_id": "debug_step_result_003",
    "action": "pause",
    "notes": "Measure SDA/SCL voltage first before continuing"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `approval_id` | string | Corresponding approval card ID |
| `action` | string | `pause` (pause troubleshooting) / `skip` (skip current step) / `abort` (terminate troubleshooting, generate diagnostic package) / `resume` (continue) |
| `notes` | string | User notes, optional |

### 6. error_lib_update — Error Library CRUD (autofix specific)

```json
{
  "type": "error_lib_update",
  "payload": {
    "action": "add",
    "entry": {
      "error_signature": "I2C readback mismatch 0xFF",
      "primary_error": "communication",
      "device_type": "I2C",
      "root_cause": "SDA stuck LOW, check pull-up",
      "fix_strategy": "Verify 4.7kΩ pull-up on SDA/SCL to 3.3V"
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `action` | string | `add` / `update` / `delete` / `query` |
| `entry` | object | error_lib.json entry. Required for `add`; provide fields to modify for `update`; only `error_signature` needed for `delete`; provide matching criteria for `query` |

### 7. stream_ack — Stream Data Acknowledgment/Termination

```json
{
  "type": "stream_ack",
  "payload": {
    "stream_id": "device_repl_001",
    "action": "stop",
    "chunk_index": 42,
    "reason": "marker_detected"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `stream_id` | string | Corresponding stream ID |
| `action` | string | `continue` (continue receiving) / `stop` (terminate stream) |
| `chunk_index` | number | Sequence number of the last received chunk |
| `reason` | string | Termination reason. `marker_detected` (expected marker detected) / `timeout` (timeout) / `user_request` (manual stop) |

When the plugin detects an expected marker (e.g., "starting scheduler") during a stream session, it can terminate early via `stream_ack(action=stop, reason=marker_detected)` and wrap the captured output as a `device_result` to return to the server.

---

## III. Error Handling

The server can send `phase_complete` with `result: "failed"` at any stage, carrying an `errors` array explaining the reason. Upon receipt, the plugin stops the current phase's timeline animation and displays an error panel.

When a plugin fails to execute a device_command / script_run / file_operation, it carries `success: false` + `error` fields in the corresponding result message. The server LLM decides the next step (retry/skip/degrade).

### Session Resumption

The plugin can carry a `session_id` in `start_phase` to resume a previously interrupted session (e.g., gen-driver's "continue later" scenario). Upon receiving an existing `session_id`, the server restores the previous context and continues from the breakpoint.

### Automatic Phase Transition

When `phase_complete.next_phase` is not null, the plugin automatically sends `start_phase` to the corresponding phase, with `payload.manifest` coming from `phase_complete.manifest_content`. The pipeline stops when next_phase is null.
