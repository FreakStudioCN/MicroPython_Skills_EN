# upy-deploy Interface Definition

> Status: ✅ Finalized
>
> Phase 5 — One-click flash and run. Compile .py→.mpy, upload firmware/, soft reset, persistent session to collect REPL output, fetch device logs, determine PASS/FAIL. On FAIL, construct error_context and pass to upy-autofix.

---

## I. Skill Overview

| Item | Content |
|------|---------|
| Phase | deploy |
| Upstream Skill | upy-generate or upy-simulate (user-triggered); upy-autofix (auto-triggered in incremental mode) |
| Downstream Skill | upy-autofix (on FAIL); upy-wiring / upy-diagram (parallel visualization on PASS) |
| One-line Responsibility | Compile upload → Soft reset → Run collect → Log fetch → LLM PASS/FAIL determination |

**Two Operation Modes:**

| Mode | Trigger | Purpose |
|------|---------|---------|
| `full` | User clicks [One-click Flash] button | Fresh compile + full upload + run |
| `incremental` | Auto-called after upy-autofix fix | Compile and upload only changed_files, then run for verification |

**Core Constraints:**
- All mpremote operations are executed by the plugin via `device_command` / `script_run`; the server does not touch the serial port
- Phase 6 determination is done by the server-side LLM (replacing the original grep rules)
- `flash_device.py` + `read_device_log.py` must support the `--json` flag to output structured data
- main.py is uploaded last and kept as .py (not compiled)

---

## II. Plugin Input → Skill (P→S)

### Full Mode (User-triggered)

```json
{
  "type": "start_phase",
  "phase": "deploy",
  "session_id": "uuid-xxx",
  "payload": {
    "mode": "full",
    "com_port": "COM3",
    "manifest": { /* Complete project-manifest.json */ },
    "previous_error": null
  }
}
```

### Incremental Mode (Autofix-triggered)

```json
{
  "type": "start_phase",
  "phase": "deploy",
  "session_id": "uuid-xxx",
  "payload": {
    "mode": "incremental",
    "com_port": "COM3",
    "manifest": { /* Complete project-manifest.json */ },
    "changed_files": [
      "firmware/tasks/sensor_task.py",
      "firmware/drivers/sht30/mock.py"
    ],
    "previous_error": {
      "error_type": "RuntimeError",
      "traceback": "Traceback (most recent call last):\n  File \"main.py\", line 42...",
      "failed_phase": "deploy",
      "failed_at": "Phase 4: REPL captured NameError"
    }
  }
}
```

| Field | Type | Required | Source | Description |
|-------|------|----------|--------|-------------|
| `mode` | string | Yes | Plugin | `"full"` / `"incremental"` |
| `com_port` | string | Yes | Plugin serial port selector | e.g., COM3, /dev/ttyACM0 |
| `manifest` | object | Yes | upy-generate output | Complete project-manifest.json |
| `changed_files` | string[] | Required when mode=incremental | upy-autofix | List of files modified by autofix; only these are compiled and uploaded |
| `previous_error` | object? | No | upy-autofix | Error context from the previous deploy failure; LLM can adjust strategy based on this |

---

## III. Skill Output → Plugin (S→P)

### Message Sequence

```
Phase 1: Compile + Upload + Verify
  → status_update "Compiling .py → .mpy... (N files)"
  → script_run(flash_device.py --port COM3 --compile --upload --verify --no-reset --json)
  → stream × N (script_stdout, one JSON line per file)
  → script_result
  → status_update "✓ Compile and upload complete: 15 .mpy + 5 .py, verification passed"

Phase 2: Soft Reset + Wait for Reconnection
  → status_update "Soft resetting device..."
  → device_command(action="soft_reset")
  → device_result (Plugin handles reconnection wait internally, includes reconnection time)
  → status_update "✓ Device ready (reconnection took 2.3s)"

Phase 3: Persistent Session Collection
  → status_update "Collecting device output... (60s timeout)"
  → device_command(action="stream", timeout_ms=60000)
  → stream × N (device_output, real-time REPL output lines)
  → device_result (Collection complete, includes full output text)
  → status_update "✓ Collection complete: 342 lines of output"

Phase 4: Fetch Device Logs
  → status_update "Reading device logs..."
  → script_run(read_device_log.py --port COM3 --log-dir /log --json)
  → script_result (stdout = JSON formatted log content)
  → status_update "✓ Read 2 log files"

Phase 5: LLM Determination
  → status_update "Analyzing run results..."
  → phase_complete (PASS/FAIL + error_context)
```

### Message Details

#### script_run — flash_device.py (Phase 1)

```json
{
  "type": "script_run",
  "payload": {
    "script_id": "deploy_flash",
    "interpreter": "python",
    "script": "tools/flash_device.py",
    "args": ["--port", "COM3", "--compile", "--upload", "--verify", "--no-reset", "--json"],
    "cwd": "{project_dir}",
    "timeout_ms": 120000
  }
}
```

**`--json` Output Format (stdout, one JSON per line):**

```
{"step": "scan", "total": 20, "entry_files": ["main.py", "boot.py"]}
{"step": "compile", "file": "tasks/sensor_task.py", "status": "ok", "size": 1536}
{"step": "compile", "file": "tasks/display_task.py", "status": "ok", "size": 2048}
{"step": "compile", "file": "drivers/sht30/sht30.py", "status": "skip", "reason": "entry_file"}
{"step": "upload", "file": "lib/scheduler.mpy", "status": "ok", "size_kb": 2.1, "progress": "1/18"}
{"step": "upload", "file": "main.py", "status": "ok", "size_kb": 1.5, "progress": "18/18", "note": "entry_last"}
{"step": "verify", "remote_total": 20, "local_total": 20, "missing": [], "status": "ok"}
{"step": "done", "status": "success", "compiled": 15, "uploaded": 20, "errors": []}
```

The server-side LLM parses each JSON line and converts compile/upload progress into `status_update`.

#### device_command — soft_reset (Phase 2)

```json
{
  "type": "device_command",
  "payload": {
    "cmd_id": "deploy_reset",
    "action": "soft_reset",
    "timeout_ms": 60000
  }
}
```

**Plugin-side Behavior (not a simple passthrough):**
1. Execute `mpremote connect <com> soft-reset`
2. Enter reconnection wait loop (poll every 2s with `resume exec "print(1)"`)
3. If COM port changes (Windows), auto-run `mpremote connect list` to rescan
4. Receive `"1"` → Device ready → Return success
5. 60s timeout → Return failure

```json
{
  "type": "device_result",
  "payload": {
    "cmd_id": "deploy_reset",
    "success": true,
    "stdout": "Device ready (COM3, reconnection took 2.3s)",
    "stderr": "",
    "exit_code": 0
  }
}
```

#### device_command — stream (Phase 3)

```json
{
  "type": "device_command",
  "payload": {
    "cmd_id": "deploy_repl",
    "action": "stream",
    "timeout_ms": 60000,
    "expect_output": true
  }
}
```

**Plugin-side Behavior:**
1. Open a persistent `mpremote connect <com> repl` session
2. For each line of output received → send a `stream` message (`stream_type: "device_output"`)
3. Detect `"starting scheduler"` or equivalent flag → terminate early (early_exit=true)
4. 60s timeout → Close session
5. Send `device_result` (includes full output text for LLM analysis)

```
stream message example:
  chunk_index=0:  "MPY: soft-reboot\n"
  chunk_index=1:  "I2C scan: [0x3C, 0x44]\n"
  chunk_index=2:  "[INFO] starting scheduler\n"
  ...
  chunk_index=341: "[INFO] tick 60 complete\n"
  → device_result(success=true, stdout=<full text>)
```

#### script_run — read_device_log.py (Phase 4)

```json
{
  "type": "script_run",
  "payload": {
    "script_id": "deploy_read_log",
    "interpreter": "python",
    "script": "tools/read_device_log.py",
    "args": ["--port", "COM3", "--log-dir", "/log", "--json"],
    "cwd": "{project_dir}",
    "timeout_ms": 30000
  }
}
```

**`--json` Output Format (stdout):**

```json
{
  "status": "ok",
  "log_dir": "/log",
  "logs": [
    {
      "name": "run_0.log",
      "size_bytes": 2048,
      "content": "2026-06-17T10:30:00Z [INFO] boot complete\n..."
    },
    {
      "name": "run_1.log",
      "size_bytes": 512,
      "content": "..."
    }
  ],
  "errors": []
}
```

When there are no log files on the device: `{"status": "empty", "log_dir": "/log", "logs": [], "errors": []}`

#### phase_complete (Phase 5 — LLM Determination)

**PASS Example:**

```json
{
  "type": "phase_complete",
  "payload": {
    "phase": "deploy",
    "result": "success",
    "summary": "Deployment successful: Device ran for 60s without anomalies, scheduler scheduled 3 tasks normally",
    "next_phase": null,
    "artifacts": [
      {
        "type": "markdown",
        "title": "Run Summary",
        "content": "### Device Run Report\n\n- **Device**: ESP32-WROOM-32 @ COM3\n- **Sampling**: 60 ticks × 1000ms = 60s\n- **I2C Scan**: [0x3C (SSD1306), 0x44 (SHT30)]\n- **Tasks**: sensor_task (every tick), display_task (every 5 ticks), alarm_task (every tick)\n- **Logs**: run_0.log (2048 bytes)\n- **Anomalies**: None"
      },
      {
        "type": "table",
        "title": "Deployment Phase",
        "headers": ["Phase", "Status", "Description"],
        "rows": [
          ["Compile", "✓", "15 .py → .mpy"],
          ["Upload", "✓", "20 files, 0 failures"],
          ["Verify", "✓", "20/20 files consistent"],
          ["Reset", "✓", "COM3 reconnection 2.3s"],
          ["Run", "✓", "60s no anomalies"]
        ]
      }
    ],
    "warnings": [],
    "errors": [],
    "error_context": null
  }
}
```

**FAIL Example:**

```json
{
  "type": "phase_complete",
  "payload": {
    "phase": "deploy",
    "result": "failed",
    "summary": "Deployment failed: SHT30 raised OSError at tick 12 during device run",
    "next_phase": "autofix",
    "artifacts": [
      {
        "type": "markdown",
        "title": "Error Summary",
        "content": "### Error Information\n\n- **Type**: OSError\n- **Location**: firmware/tasks/sensor_task.py:23 — sht30.measure()\n- **Occurrence Time**: tick 12 (~15s after device start)\n- **Logs**: run_0.log contains full Traceback"
      }
    ],
    "warnings": [],
    "errors": ["OSError: I2C read failed at tick 12"],
    "error_context": {
      "phase": "deploy",
      "error_type": "OSError",
      "file_path": "firmware/tasks/sensor_task.py",
      "line_number": 23,
      "traceback": "Traceback (most recent call last):\n  File \"main.py\", line 42, in sensor_cb\n  File \"tasks/sensor_task.py\", line 23, in sensor_read\nOSError: I2C read failed",
      "repl_output": "<full REPL output text>",
      "log_files": {
        "run_0.log": "<log file content>"
      },
      "log_report": {
        "error_count": 1,
        "errors": [
          { "level": "P0_TRACEBACK", "message": "OSError: I2C read failed", "line": 42 }
        ]
      },
      "device_info": {
        "com_port": "COM3",
        "i2c_scan": "[0x3C, 0x44]",
        "firmware": "MicroPython v1.24.1"
      }
    }
  }
}
```

**error_context passed to upy-autofix:** autofix receives `error_context`, extracts `traceback` + `file_path` + `line_number` as entry points, and uses `repl_output` + `log_report` as auxiliary context.

#### status_update List

| step_id | level | message | Trigger Time |
|---------|-------|---------|-------------|
| compile_start | info | Compiling .py → .mpy... | Phase 1 start |
| compile_progress | info | Compiling: sensor_task.py → sensor_task.mpy (15/15) | Per file compilation complete |
| compile_done | success | ✓ Compilation complete: 15 .mpy, 0 failures | Compilation phase end |
| upload_start | info | Uploading firmware/ → device... | Upload phase start |
| upload_progress | info | Uploading: lib/scheduler.mpy (1/20) | Per file upload complete |
| upload_done | success | ✓ Upload complete: 20 files | Upload phase end |
| verify_start | info | Verifying file integrity... | Verify phase start |
| verify_done | success | ✓ Verification passed: 20/20 files consistent | Verification passed |
| verify_fail | warn | ⚠ 2 files missing, retransmitting... | Verification failed |
| reset_start | info | Soft resetting device... | Phase 2 start |
| reset_done | success | ✓ Device ready (reconnection took X.Xs) | Device ready |
| reset_timeout | error | ✗ Device reconnection timeout (60s) | Reconnection timeout |
| stream_start | info | Collecting device output (60s)... | Phase 3 start |
| stream_done | success | ✓ Collection complete: XXX lines of output | Collection end |
| stream_early | success | ✓ Scheduler started, ending collection early | Detected starting scheduler |
| log_start | info | Reading device logs... | Phase 4 start |
| log_done | success | ✓ Read N log files | Log reading complete |
| log_empty | info | No log files on device | No logs |
| judge_start | info | Analyzing run results... | Phase 5 start |
| judge_pass | success | ✓ Deployment successful: Device running normally | PASS |
| judge_fail | error | ✗ Deployment failed: [Error Type] | FAIL |

### No approval_request Required

No human-interaction cards are needed throughout the deploy process. The user only needs to click the [One-click Flash] button to start; everything else runs automatically.

---

## IV. SKILL.md Modification Points

A total of 8 changes:

| No. | Location | Current Behavior | Change To | Reason |
|-----|----------|-----------------|-----------|--------|
| 1 | Pre-checks | `python --version` + `mpremote --version` | Remove. Dependencies are guaranteed by the plugin environment | Server does not perceive the runtime environment |
| 2 | Phase 1 Upload | Directly call `mpremote fs cp/mkdir` or `python flash_device.py` | `script_run(flash_device.py --json --verify)` | Script executed locally by the plugin; JSON output parsed by the server |
| 3 | Phase 2 Verify | Separate step: `mpremote fs tree/ls` + Bash comparison | Merge into Phase 1: `flash_device.py --verify` auto-verifies after upload, auto-retransmits missing files | Reduces one script_run round trip |
| 4 | Phase 3 Soft Reset | `mpremote soft-reset` + Python polling `exec("print(1)")` | `device_command(action="soft_reset")`, plugin handles the entire reconnection wait loop internally, returns success/timeout | Plugin encapsulates reconnection logic; server only cares about the result |
| 5 | Phase 4 Persistent Session | subprocess.Popen + threading code (pipe/PTY) | `device_command(action="stream", timeout_ms=60000)` → `stream` messages pushed line by line | Reuses stream messages; plugin establishes REPL session |
| 6 | Phase 5 Fetch Logs | `mpremote fs cat/cp` + `log_report.py` | `script_run(read_device_log.py --json)` returns all log content at once (JSON); server can then call `script_run(log_report.py)` for structured parsing | Reduces multiple mpremote calls, fetches everything in one go |
| 7 | Phase 6 Initial Judgment | Local grep rules (Traceback/rst cause/MemoryError) | Server-side LLM comprehensively analyzes REPL output + log_report JSON → determines PASS/FAIL | LLM has full project context, more accurate judgment; can distinguish expected warnings from real errors |
| 8 | New incremental mode | Only full mode | Add `mode=incremental`: read `changed_files` → compile and upload only those files → flash_device.py uses `--files` parameter | Fast verification loop for autofix→deploy, no need for full retransmission |

---

## V. Verification Script Changes

### flash_device.py

**Path:** `G:\MicroPython_Skills\upy-scaffold\templates\pc\flash_device.py`

| Change | Content |
|--------|---------|
| Add `--json` flag | Output one JSON line per step to stdout (step/status/file/size, etc.), replacing the current `print("[compile] ...")` human-readable format. Keep original output without `--json` |
| Add `--verify` flag | After upload, auto-run `fs ls` to traverse remote directories, compare with local file list, output `{"step":"verify", "missing":[...], "status":"ok"/"fail"}`. Auto-retransmit missing files |
| Add `--files` parameter | Value `"tasks/sensor.py,drivers/sht30/sht30.py"` (comma-separated relative paths). When specified, only compile/upload these files (incremental mode). **Entry files are always kept as .py and not compiled** |
| Remove `select_com_port()` | Interactive COM port selection → replaced by `--port` parameter (provided by the plugin in start_phase). Exit with error if no `--port` |
| Remove `--flash` firmware flashing | Firmware flashing is a one-time operation, not part of the deploy flow. Parameter kept but not used by deploy |
| `_upload_dir` sort guarantee | main.py is always uploaded last (existing logic, ensure no regression) |

**`--json` Output Line Type Specification:**

| step | Fields | Description |
|------|--------|-------------|
| `scan` | `total`, `entry_files` | Scan result: total file count + list of entry files (not compiled) |
| `compile` | `file`, `status`, `size?`, `error?` | status: ok / skip(entry file) / fail |
| `upload` | `file`, `status`, `size_kb`, `progress`, `note?` | progress: "N/M"; note: entry_last indicates main.py uploaded last |
| `verify` | `remote_total`, `local_total`, `missing`, `status` | Verification result |
| `done` | `status`, `compiled`, `uploaded`, `errors` | Final summary |

### read_device_log.py

**Path:** `G:\MicroPython_Skills\upy-scaffold\templates\pc\read_device_log.py`

| Change | Content |
|--------|---------|
| Add `--json` flag | Output structured JSON to stdout (`{status, log_dir, logs[{name, size_bytes, content}], errors[]}`), replacing the current raw text output. Keep original output without `--json` |
| Remove `--tail` / `--clear` | These features are not used by the deploy flow. Parameters kept but not called by deploy |

### log_report.py

**Path:** `G:\MicroPython_Skills\upy-scaffold\templates\pc\log_report.py`

**Basically no changes needed.** It already outputs structured JSON, compatible with error_context. Just confirm that the error level enumeration in `parse_log()` is consistent with error_context.

### run_on_device.py (Shared with gen-driver)

**Path:** `G:\MicroPython_Skills\upy-deploy\scripts\run_on_device.py` (New)

**Purpose:** Use mpremote to send .py files to the device REPL for execution, capture stdout/stderr to log files. Used by gen-driver's Step 3 verification loop and Step 7 independent testing; can also be reused for deploy's Phase 3 REPL quick test.

| Parameter | Description |
|-----------|-------------|
| `--com` | COM port number |
| `--file` | Path to the .py file to execute (relative to project directory) |
| `--capture` | Enable output capture (write to logs/ directory) |
| `--timeout-ms` | Device execution timeout (ms), default 30000 |
| `--json-summary` | stdout outputs `{"status":"ok","output_file":"...","exit_code":0,"duration_ms":N}` |

Difference from `flash_device.py`: `flash_device.py` handles compile+upload+verify (full deployment), while `run_on_device.py` handles REPL send+execute+capture output (quick verification).

---

## VI. UI Components to be Implemented on the Plugin Side

| Component | Corresponding Message | Key Functionality |
|-----------|-----------------------|-------------------|
| Deployment Progress Timeline | status_update × 10~15 | Seven phases: Compile→Upload→Verify→Reset→Run→Logs→Judgment |
| Device Terminal Panel | stream (device_output) | **Reuse simulate's terminal panel**, display REPL output lines in real-time |
| Deployment Result Panel | phase_complete | PASS: Run summary markdown + phase table; FAIL: Error summary + error_context preview |
| [One-click Flash] Button | Trigger start_phase(mode="full") | Enabled after generate/simulate completes |
| [Re-flash] Button | Trigger start_phase(mode="full") | Replaces the "One-click Flash" button after FAIL |

### Device Terminal Panel Description

Shares the same UI component as simulate's terminal panel, with the difference being the data source:
- simulate: `stream` message `stream_type: "script_stdout"` (sim_main.py --plain output)
- deploy: `stream` message `stream_type: "device_output"` (REPL real-time output)

The plugin appends to the terminal panel in real-time upon receiving stream messages. Supports pause scrolling, copy text, and clear.

---

## VII. Independent Test Scenarios

### Plugin-side Testing (No Server)

1. Manually send `status_update` sequence → Verify seven-phase timeline rendering
2. Manually send `stream` sequence (simulate REPL output lines) → Verify terminal panel real-time scrolling
3. Manually send `phase_complete` (PASS) → Verify run summary + phase table rendering
4. Manually send `phase_complete` (FAIL + error_context) → Verify error summary panel + autofix entry
5. Construct `start_phase` message → Verify correct JSON is emitted after clicking [One-click Flash] button

### Skill-side Testing (No Plugin)

1. Use mock_plugin.py to simulate:
   - `script_run(flash_device.py)` → Return simulated JSON line output
   - `device_command(action="soft_reset")` → Return success
   - `device_command(action="stream")` → Return simulated REPL output (with/without Traceback)
   - `script_run(read_device_log.py)` → Return simulated log JSON
2. Verify PASS path: Normal REPL output + no error logs → result="success"
3. Verify FAIL path: REPL contains Traceback → result="failed" + complete error_context
4. Verify incremental mode: changed_files passed → flash_device.py only processes specified files
5. Check all emitted message JSON conforms to 02-protocol.md Schema
