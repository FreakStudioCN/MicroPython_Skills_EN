# Parallel Development Strategy

## Core Principle

**Define the protocol first, both sides develop against the protocol, mock independently for testing, and finally integrate.**

Skill maintainers and plugin engineers do not need to wait for each other. After defining the message format, each works independently.

---

## Three-Line Parallel Model

```
         ┌── Protocol Definition (this document set) ──┐
         │  02-protocol.md                             │
         │  skills/_template.md                        │
         └─────────────────────┬───────────────────────┘
                               │ Both sides adhere to
      ┌────────────────────────┼────────────────────────┐
      ▼                        │                        ▼
┌──────────────────┐           │                 ┌──────────────┐
│ Skill Side       │           │                 │ Plugin Side  │
│ (Embedded)       │           │                 │ (Frontend)   │
├──────────────────┤           │                 ├──────────────┤
│ Modify           │           │                 │ Implement    │
│ SKILL.md         │           │                 │ 7 message    │
│ Output           │           │                 │ send/receive │
│ protocol messages│           │                 │ + UI components│
│                  │           │                 │              │
│ Testing:         │           │                 │ Testing:     │
│ mock plugin      │           │                 │ mock server  │
│ (simple script)  │           │                 │ (send fake   │
│                  │           │                 │  messages)   │
└──────────────────┘           │                 └──────────────┘
                               │
                               ▼
                        ┌──────────────┐
                        │ Integration  │
                        │ (Real Docking)│
                        └──────────────┘
```

---

## Plugin Side Independent Development Guide

### What You Need to Do

1. Implement receiving and rendering of 7 S→P messages (full JSON Schema in `02-protocol.md`)
2. Implement sending of 5 P→S messages
3. Implement device passthrough (mpremote spawn)
4. Implement script execution (child_process.spawn)

### Testing Without a Server

Use a mock server to send fake messages. The simplest way: write a Node.js script that pushes JSON messages into the WebView in sequence:

```javascript
// mock-server.js — for plugin side independent testing
const messages = [
  { type: "status_update", payload: { level: "info", message: "Analyzing requirements...", progress: 0.1 } },
  { type: "status_update", payload: { level: "success", message: "✓ Extracted 3 components", progress: 0.3 } },
  { type: "approval_request", payload: { /* component confirmation card */ } },
  // ... simulate the complete flow
];

// Send one message every second to simulate server response
messages.forEach((msg, i) => {
  setTimeout(() => webview.postMessage(msg), i * 1000);
});
```

### Acceptance Criteria (Independent Testing)

- [ ] Can receive `status_update` and render it as a timeline
- [ ] Can receive `approval_request` and render it as an approval card; user action can send `approval_response`
- [ ] Can receive `device_command(action=exec)` and spawn mpremote, sending results back as `device_result`
- [ ] Can receive `phase_complete` and render the results panel (table/file tree/markdown)
- [ ] Can receive `stream` and append it in real-time to the terminal panel

---

## Skill Side Independent Development Guide

Skill side development has two phases: **first modify the logic (testable locally), then modify the communication (mechanical translation).** Do not mix the two.

### Phase A — Logic First (Run Directly with Local Claude Code)

**What to change:** The flow logic in SKILL.md — add steps, fix bugs, adjust order, add validation.

**What not to change:** The communication method. Keep local tools like `Read`/`Bash`/`AskUserQuestion` that Claude Code can execute directly.

**How to test:** Load SKILL.md with Claude Code and run it locally. You will immediately know if the logic and output are correct.

```
Example: After modifying upy-gen-driver Step 5, add Step 6 (generate independent test script)
  → SKILL.md writes "Write firmware/drivers/sht30_driver/test_sht30.py"
  → Claude Code writes the file ✓
  → Logic verification passes ✓
```

**Acceptance Criteria:**
- [ ] Local Claude Code can complete the skill end-to-end with correct output
- [ ] Number of flow steps, order, and branch logic meet expectations
- [ ] No plugin, no server, no mock required

### Phase B — Communication Translation (Execute After Logic is Confirmed)

After Phase A passes, mechanically translate each item according to the **"IV. SKILL.md Modification Points"** table in the corresponding interface document:

```
Local Tool                        Protocol Message
Read                          →   file_operation(read)
Write / Edit                  →   file_operation(write)
Bash(python validate.py ...)  →   script_run(validate.py ...)
Bash(mpremote ...)            →   device_command(...)
AskUserQuestion               →   approval_request(...)
```

The logic remains unchanged; only the I/O method is swapped. This step will not introduce new bugs.

**How to test:** Use `mock_plugin.py` to simulate plugin responses and verify the message format.

```python
# mock_plugin.py — for Phase B protocol verification
import json, sys

mock_device_state = {"i2c_scan": "[48, 60]"}
mock_user_choice = {"action": "confirm"}

for line in sys.stdin:
    msg = json.loads(line)
    t = msg["type"]
    
    if t == "approval_request":
        print(json.dumps({
            "type": "approval_response",
            "payload": {"approval_id": msg["payload"]["approval_id"], **mock_user_choice}
        }))
    elif t == "device_command":
        print(json.dumps({
            "type": "device_result",
            "payload": {"cmd_id": msg["payload"]["cmd_id"], "success": True,
                        "stdout": mock_device_state.get("i2c_scan", "")}
        }))
    elif t == "status_update":
        print(f"[UI] {msg['payload']['level']}: {msg['payload']['message']}")
    elif t == "phase_complete":
        print(f"[UI] Phase done: {msg['payload']['summary']}")
```

**Acceptance Criteria:**
- [ ] All I/O points have been translated to the corresponding protocol messages according to the modification points table
- [ ] mock_plugin.py can complete one phase, and the message sequence meets expectations
- [ ] Message JSON conforms to the Schema in `02-protocol.md`

---

## Integration Testing Checklist

After both sides pass independent testing, integrate phase by phase in the following order:

| Order | Phase | Integration Content | Estimated Time |
|------|-------|---------|---------|
| 1 | upy-analyze | Intent decomposition → Component confirmation card → Driver search progress → Results panel | 30min |
| 2 | upy-select-hw | MCU recommendation card / User-selected board → BOM + pin table | 20min |
| 3 | upy-scaffold | Scheduling mode selection → File tree preview | 20min |
| 4 | upy-generate | Code generation progress → File writing → Lint results display | 60min |
| 5 | upy-deploy | Device command passthrough → REPL real-time stream → Results panel | 60min |
| 6 | upy-simulate | Simulation script execution → Rich output stream | 30min |
| 7 | upy-autofix | Fix loop progress → Multiple device_command round trips | 45min |
| 8 | upy-wiring + upy-diagram | HTML preview rendering | 20min |

---

## Communication Mechanism Suggestions

- Skill maintainer notifies the plugin side after completing the interface document for each skill
- Plugin side implements the UI components for that phase according to the document (can work on other phases in parallel)
- Integrate and test after each phase is completed; do not wait until all 10 skills are written before integrating
