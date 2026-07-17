# System Architecture

## One-Sentence Summary

**The server-side LLM executes the complete SKILL.md to make decisions. The plugin is a mindless execution layer—rendering UI + transparently passing through local I/O.**

---

## Three-Component Responsibility Boundaries

```
┌─────────────────────────────────────────────────┐
│                   VS Code Plugin                  │
│  (TypeScript, Local Process)                      │
│                                                   │
│  Responsibilities:                                │
│  ✅ Render UI (Board Gallery / Approval Card /    │
│     Progress Timeline / Result Panel)             │
│  ✅ Pass through mpremote commands (scan/flash/   │
│     REPL)                                         │
│  ✅ Pass through file read/write (write firmware/ │
│     to workspace)                                 │
│  ✅ Pass through script execution (flake8 /       │
│     pylint / render scripts)                      │
│  ✅ Pass through device output stream (real-time  │
│     REPL → Server)                                │
│  ✅ Manage user preferences (mode/language/       │
│     existing hardware)                            │
│                                                   │
│  Not Responsible For:                             │
│  ❌ No business decisions                         │
│  ❌ No parsing of device output meaning           │
│  ❌ No code generation                            │
│  ❌ No knowledge of skill / SKILL.md existence    │
└─────────────┬───────────────────────────────────┘
              │ HTTP + SSE
              │ Protocol: JSON (7 message types)
              ▼
┌─────────────────────────────────────────────────┐
│                   Server-Side                     │
│  (Python / LLM, Remote)                           │
│                                                   │
│  Responsibilities:                                │
│  ✅ Load complete SKILL.md as LLM system           │
│     instruction                                   │
│  ✅ Execute pipeline decisions (intent             │
│     decomposition → selection → generation → ...) │
│  ✅ Call upypi / GitHub API to search for drivers │
│  ✅ Generate code / tests / wiring.json /         │
│     diagram.json                                  │
│  ✅ Analyze device output, determine PASS/FAIL    │
│  ✅ Error classification decisions + delegate to  │
│     upstream skill                                │
│  ✅ Maintain board database + skill version        │
│                                                   │
│  Not Responsible For:                             │
│  ❌ No direct mpremote operations (no serial      │
│     access)                                       │
│  ❌ No writing to user's local files              │
│  ❌ No UI rendering                               │
└─────────────────────────────────────────────────┘
```

## Key Design Decisions

| Decision | Reason |
|------|------|
| Plugin makes no decisions | Plugin engineers don't need to understand embedded/MicroPython/drivers. Only implement sending/receiving 7 message types |
| SKILL.md kept entirely on server | Avoids `mpy-hardware-extension` phase_profile sanitisation issues |
| All local I/O transparently passed through | Plugin doesn't know what mpremote is doing—server says exec, it execs |
| Board database can be local or remote | During local testing, plugin reads JSON file directly; in production, goes through API |
| SKILL.md does not hardcode communication method | SKILL.md describes business logic, not bound to I/O mechanism. LLM adapts based on environment: locally uses Read/Bash/AskUserQuestion, server uses file_operation/device_command/approval_request. Same SKILL.md runs in both environments |

## Message Flow

```
Plugin → Server (7 types):
  start_phase          — Start skill
  approval_response    — Result of user clicking approval card
  device_result        — Result of mpremote command execution
  script_result        — Result of local script execution
  file_result          — Result of file operation
  user_intervention    — User intervention during troubleshooting (autofix only)
  error_lib_update     — Error library CRUD (autofix only)
  stream_ack           — Stream data acknowledgement/termination

Server → Plugin (7 types):
  approval_request     — Requires user approval (device confirmation/alternative selection/mode selection)
  status_update        — Progress update (searching/generating/compiling)
  device_command       — Pass through mpremote command
  file_operation       — Read/write workspace files
  script_run           — Execute local script
  phase_complete       — Phase complete, includes result data
  stream               — Real-time data stream (device REPL output)
```

## Example of a Complete Phase Interaction

```
User input "Make a temperature and humidity monitor"
  → Plugin: Package as message → Server

Server (upy-analyze):
  1. → status_update: "Analyzing requirements..."
  2. → approval_request: Device confirmation card
  3. ← Plugin: approval_response {confirmed: true, devices: [...]}
  4. → status_update: "Searching for drivers... (1/3)"
  5. → status_update: "✓ SSD1306 → upypi"
  6. → phase_complete: {result: success, device_table: [...], ...}

Server (upy-select-hw):
  7. → approval_request: MCU recommendation card
  8. ← Plugin: approval_response {selected_board: "esp32-devkit-v1"}
  9. → phase_complete: {result: success, bom: [...], pinout: [...]}

... (subsequent phases similar)
```

## Session Management

- Each project has one session_id (UUID, generated by plugin)
- Plugin sends session_id in the first message
- Server maintains session context (current phase + manifest snapshot + LLM conversation history)
- Plugin restart → new session_id → server starts over
