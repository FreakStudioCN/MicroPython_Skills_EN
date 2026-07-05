# upy-simulate Interface Definition

> Status: ✅ Finalized
>
> Phase 4.5 — Full-process business simulation on PC. Reads all code from firmware/ as context, LLM autonomously designs Mock assembly + scheduling + visualization + data scenarios, generates test/pc/sim_main.py and runs verification.

---

## I. Skill Overview

| Item | Content |
|------|---------|
| Phase | simulate |
| Upstream Skill | upy-generate (manual trigger) or upy-autofix (auto-triggered in verify mode) |
| Downstream Skill | upy-deploy (manual) or back to upy-autofix (on FAIL) |
| One-line Responsibility | Full-process PC simulation — LLM reads code → autonomously designs Mock/scheduling/scenarios → generates sim_main.py → validates → runs → outputs coverage report |

**Two Operation Modes:**

| Mode | Trigger | Purpose |
|------|---------|---------|
| `full` | User clicks [PC Simulation] button | Fresh generation of sim_main.py, full read of firmware/ |
| `verify` | Auto-called after upy-autofix fix | Only re-reads files in changed_files, updates sim_main.py then runs |

**Core Constraints:**
- Do not modify any files under firmware/
- All new code written to test/pc/
- sim_main.py must support `--plain` flag (disables rich formatting, outputs plain text for streaming)
- Default generation is CLI mode (rich), no GUI mode

---

## II. Plugin Input → Skill (P→S)

Plugin sends **1 message** to server to start this skill:

### Full Mode (User Manual Trigger)

```json
{
  "type": "start_phase",
  "phase": "simulate",
  "session_id": "uuid-xxx",
  "payload": {
    "mode": "full",
    "manifest": { /* complete project-manifest.json */ },
    "user_scenario": null,
    "skip_approval": false
  }
}
```

| Field | Type | Required | Source | Description |
|-------|------|----------|--------|-------------|
| `mode` | string | Yes | Plugin | `"full"` / `"verify"` |
| `manifest` | object | Yes | upy-generate output | Complete project-manifest.json |
| `user_scenario` | string? | No | User input | Custom scenario description entered by user in plugin (e.g., "WiFi disconnects for 5 seconds then auto-reconnects"). null means no custom scenario |
| `skip_approval` | boolean | No | Plugin | Default false. When true, skips Step 5 scenario selection card and directly runs the default recommended scenario |

### Verify Mode (Auto-called by upy-autofix)

```json
{
  "type": "start_phase",
  "phase": "simulate",
  "session_id": "uuid-xxx",
  "payload": {
    "mode": "verify",
    "manifest": { /* ... */ },
    "changed_files": [
      "firmware/tasks/sensor_task.py",
      "firmware/drivers/sht30/mock.py"
    ],
    "skip_approval": true
  }
}
```

| Field | Type | Required | Source | Description |
|-------|------|----------|--------|-------------|
| `changed_files` | string[] | Required when mode=verify | upy-autofix | List of files modified by autofix. simulate only needs to re-read these files + sim_main.py, check if signature matches |

---

## III. Skill Output → Plugin (S→P)

### Message Sequence

```
Step 1 Full Context Read
  → file_operation(read) × 15+ (full mode)
  → file_operation(read) × 1~5 (verify mode, only changed_files + sim_main.py)
  → status_update "Reading firmware/ code..."

Step 1B Project Classification
  → status_update "Project type: sensor monitoring + alarm, no network"

Step 2 LLM Autonomous Design
  → status_update "Designing Mock assembly plan..."
  → status_update "Designing data scenarios..." × N

Step 3 Code Generation
  → file_operation(write) × 1~2: sim_main.py + sim_scheduler.py (timer mode)
  → status_update "Generated test/pc/sim_main.py"

Step 4 Code Validation
  → script_run(flake8) → script_result
  → script_run(pylint) → script_result
  → status_update "✓ flake8 passed" / "✗ flake8: 3 errors"

Step 5 Scenario Selection (Conditional)
  → approval_request: Scenario selection card (when skip_approval=false)
  → Or skip directly (when skip_approval=true, use recommended scenario)

Step 6 Run
  → script_run(sim_main.py --plain --ticks N --scenario X)
  → stream multiple (script_stdout, one line per tick)
  → script_result (run complete)

Step 7 Output
  → phase_complete: Coverage report panel
```

### Message Details

#### approval_request — Scenario Selection Card (Conditional)

**Trigger Condition:** `skip_approval` = false
**No Trigger:** `skip_approval` = true (directly run recommended scenario)

```
┌──────────────────────────────────────────┐
│  Simulation Run                           │
│                                          │
│  Project Type: Sensors ×2, Alarm, OLED Display │
│  5 scenarios generated:                   │
│    normal, temp_rising, temp_dropping,    │
│    intermittent_failure, sensor_death     │
│                                          │
│  Recommended: temp_rising --ticks 60      │
│  (Covers complete alarm cycle: trigger→cooldown→recovery) │
│                                          │
│  ⚠ normal scenario only validates data flow, │
│     does not trigger any business branch. │
│                                          │
│  [Run Recommended]  [Run normal]         │
│  [Switch Scenario...]   [Custom Scenario...] │
│  [Skip for Now]                           │
└──────────────────────────────────────────┘
```

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "sim_scenario_select",
    "header": "Simulation Run",
    "question": "PC simulation script has passed syntax validation. Start running?",
    "summary": {
      "project_types": ["sensor_monitoring", "alarm_monitoring", "gui_display"],
      "scenarios": [
        { "name": "temp_rising", "description": "Temperature continuously rises → crosses high threshold → alarm triggers", "recommended": true, "min_ticks": 60 },
        { "name": "temp_dropping", "description": "Temperature continuously drops → crosses low threshold", "recommended": false, "min_ticks": 30 },
        { "name": "normal", "description": "Data fluctuates within normal range, no thresholds triggered", "recommended": false, "min_ticks": 30 },
        { "name": "intermittent_failure", "description": "SHT30 intermittent failure → validates independent fault tolerance", "recommended": false, "min_ticks": 30 },
        { "name": "sensor_death", "description": "SHT30 permanent failure → validates degraded behavior", "recommended": false, "min_ticks": 30 }
      ],
      "warning": "normal scenario only validates data flow, does not trigger any business branch."
    },
    "items": [],
    "allow_add": false,
    "allow_remove": false,
    "multi_select": false,
    "actions": [
      { "label": "Run temp_rising (Recommended)", "value": "run_recommended", "primary": true },
      { "label": "Run normal scenario", "value": "run_normal" },
      { "label": "Switch scenario to run", "value": "custom_scenario" },
      { "label": "Custom scenario", "value": "custom_user" },
      { "label": "Skip for now", "value": "skip" }
    ]
  }
}
```

**approval_response Handling:**

| action value | Server Behavior |
|--------------|-----------------|
| `run_recommended` | Run recommended scenario, `--ticks` uses recommended value |
| `run_normal` | Run normal scenario, `--ticks` default 30 |
| `custom_scenario` | User writes scenario name in `notes` (e.g., `--scenario sensor_death`) |
| `custom_user` | User writes natural language description in `notes` → LLM maps to mock API → generates new scenario → runs |
| `skip` | Do not run, directly phase_complete (result="partial"), keep sim_main.py |

#### status_update List

| step_id | level | message | Trigger Time |
|---------|-------|---------|--------------|
| read_context | info | Reading firmware/ code... (N/15) | Step 1 start |
| read_context_done | success | Read 15 files, total XXXX lines | Step 1 complete |
| classify | info | Analyzing project type... | Step 1B start |
| classify_done | success | Project type: sensor monitoring + alarm, no network | Step 1B complete |
| design_mock | info | Designing Mock assembly plan... | Step 2 start |
| design_scenario | info | Designing data scenarios... (N/M) | Step 2D |
| scenario_done | success | Generated 5 scenarios, covering 3/5 dimensions | Step 2D complete |
| generate_code | info | Generating sim_main.py... | Step 3 |
| write_sim | success | ✓ Generated test/pc/sim_main.py (XXX lines) | sim_main.py write complete |
| lint_flake8 | info | Validating flake8... | Step 4 start |
| lint_flake8_pass | success | ✓ flake8 passed | flake8 no errors |
| lint_flake8_fail | warn | ✗ flake8: N errors → fixing | flake8 has errors |
| lint_pylint | info | Validating pylint... | pylint start |
| lint_pylint_pass | success | ✓ pylint passed | pylint no errors |
| lint_pylint_fail | warn | ✗ pylint: N issues → fixing | pylint has warnings/errors |
| lint_retry | info | Round N fix... | Re-validate after fix |
| lint_max_retry | error | Failed after 5 rounds, please check manually | Exceeds 5 rounds |
| sim_running | info | Running sim_main.py --scenario temp_rising --ticks 60 | Step 6 start |
| sim_tick | info | (Real-time output via stream) | Output per tick |
| sim_done_pass | success | ✓ Simulation passed — all @Coverage events triggered at expected ticks | PASS |
| sim_done_weak | warn | ⚠ Weak pass — 3/5 dimensions covered, 2 uncovered | WEAK_PASS |
| sim_done_fail | error | ✗ Simulation failed — Python Traceback | FAIL |

#### phase_complete

```json
{
  "type": "phase_complete",
  "payload": {
    "phase": "simulate",
    "result": "success",
    "summary": "Simulation complete: temp_rising scenario PASS, 5/5 @Coverage events triggered within expected range",
    "next_phase": "deploy",
    "artifacts": [
      {
        "type": "markdown",
        "title": "Coverage Report",
        "content": "### Simulation Coverage Report\n\n| Dimension | Status | Description |\n|-----------|--------|-------------|\n| [sensor] Sensor Read | ✅ | 60/60 ticks normal |\n| [sensor] Sensor Fault Tolerance | ✅ | Triggered → OSError at ticks 3,6,9... |\n| [alarm] High Temp Alarm Trigger | ✅ | Triggered → temp ≥ 35.0 at tick 21 |\n| [alarm] Actuator Activation | ✅ | Buzzer ON + LED ON at tick 21 |\n| [alarm] Alarm Cooldown | ✅ | cooling active ticks 21-50 |\n| [alarm] Low Temp Alarm | ⚠ Not Covered | Current scenario does not cover |\n| [alarm] Humidity Alarm | ⚠ Not Covered | No scenario covers 80%/20% humidity thresholds |\n\n**Result: WEAK_PASS**\n\nSuggestion: `python test/pc/sim_main.py --scenario temp_dropping --ticks 30`\nSuggestion: Add humidity_high scenario to cover 80% humidity threshold"
      }
    ],
    "warnings": [
      "Low temperature alarm threshold not covered, suggest running temp_dropping scenario",
      "Humidity alarm 80%/20% thresholds not covered by any scenario"
    ],
    "errors": []
  }
}
```

**result Values:**

| result | Condition |
|--------|-----------|
| `success` | Run passed (PASS), all @Coverage events triggered |
| `partial` | Weak pass (WEAK_PASS), some dimensions not covered; or user chose "Skip for now" |
| `failed` | Run failed (FAIL), has Traceback or exceeded 5 fix rounds |

---

## IV. SKILL.md Modification Points

Total 6 changes:

| No. | Location | Current Behavior | Changed To | Reason |
|-----|----------|-----------------|------------|--------|
| 1 | Step 1 Full Read | LLM directly calls Read tool to read files | Read files one by one via `file_operation(read)` (server reads local files through plugin) | Server has no local filesystem access |
| 2 | Step 4 Validation | `Bash: python -m flake8 ...` + `python -m pylint ...` | `script_run(flake8)` + `script_run(pylint)`, parse script_result.stdout/stderr | Script execution unified through plugin |
| 3 | Step 5 Ask User | `AskUserQuestion(...)` (CLI Q&A) | `approval_request` scenario selection card | Plugin renders approval card, not command-line interaction |
| 4 | Step 6 Run | `Bash: python test/pc/sim_main.py --ticks N ...` | `script_run(sim_main.py --plain --ticks N ...)`, output per tick pushed in real-time via `stream` | `--plain` disables rich formatting to make output pipeable; stream enables real-time display on plugin |
| 5 | sim_main.py Generation Constraint | No `--plain` requirement | LLM must support `--plain` flag in sim_main.py: when `--plain`, disable rich Live/Table/Panel, use `print()` for line-by-line JSON output (`{"tick": N, ...}`) | No TTY when running via script_run, rich Live produces ANSI garbage |
| 6 | New verify mode | No such mode | Add `mode=verify` entry: only read changed_files + sim_main.py, detect signature changes → update sim_main.py → directly run recommended scenario (skip_approval=true) | autofix→simulate fast verification loop |

### sim_main.py `--plain` Output Format Specification

When `--plain`, output one JSON line per tick:

```json
{"tick": 1, "temp": 25.1, "hum": 61.2, "alarm": false, "buzzer": false, "led": false, "display": "T:25.1 H:61.2\nOK"}
{"tick": 2, "temp": 25.3, "hum": 60.8, "alarm": false, "buzzer": false, "led": false, "display": "T:25.3 H:60.8\nOK"}
...
{"tick": 21, "temp": 35.2, "hum": 58.1, "alarm": true, "buzzer": true, "led": true, "display": "T:35.2 H:58.1\nALARM!"}
```

Fields are determined by LLM based on the project, but must include `tick`. `display` field is optional (when display device exists).

---

## V. Validation Script Changes

No new validation scripts needed. sim_main.py itself goes through flake8 + pylint validation, which is equivalent to validation.

### Impact on upy-generate Templates

In upy-generate's scaffold templates, `sim_main.py` is not a template file (dynamically generated by upy-simulate), no impact. However, the scheduling callback in `firmware/main.py` template should reserve an interface, not involved in this modification.

---

## VI. UI Components Needed on Plugin Side

| Component | Corresponding Message | Key Functionality |
|-----------|----------------------|-------------------|
| Progress Timeline | status_update × N | Six-stage progress: file read → classification → design → generation → validation → run |
| Scenario Selection Card | approval_request | Scenario list + recommended marker + "Run Recommended"/"Custom"/"Skip for Now" buttons |
| Terminal Output Panel | stream (script_stdout) | Real-time display of JSON lines per tick, switchable to table view |
| Coverage Report Panel | phase_complete (markdown artifact) | Render coverage table, PASS/WEAK_PASS/FAIL status markers |
| [PC Simulation] Button | Trigger start_phase(mode="full") | Enable after upy-generate completes |
| [Re-simulate] Button | Trigger start_phase(mode="full") | Replace "PC Simulation" button after simulation completes, can re-run |

### Terminal Output Panel Description

Plugin renders in real-time upon receiving `stream` message (`stream_type: "script_stdout"`). When `--plain`, each line is JSON, plugin can parse into structured data:
- **Default View**: Raw text scrolling (like terminal)
- **Table View**: Parse JSON lines, render as dynamic table (one row per tick, columns = JSON keys)
- User can switch between the two views

---

## VII. Independent Test Scenarios

### Plugin-Side Testing (No Server)

1. Manually send `status_update` sequence → verify six-stage timeline rendering
2. Manually send `approval_request` (scenario selection card) → verify:
   - Recommended scenario highlighted
   - Clicking "Run Recommended" sends `approval_response(action="run_recommended")`
   - Clicking "Skip for Now" sends `approval_response(action="skip")`
3. Manually send `stream` sequence (simulate `--plain` per-tick JSON lines) → verify terminal panel real-time update + table view switching
4. Manually send `phase_complete` (markdown coverage report) → verify coverage table + PASS/WEAK_PASS/FAIL marker rendering

### Skill-Side Testing (No Plugin)

1. Use mock_plugin.py to simulate plugin responses:
   - Manually construct start_phase (mode="full", manifest, skip_approval=true)
   - Auto-return file content for file_operation(read) (need a complete firmware/ directory prepared)
   - Auto-return `{"exit_code": 0, "stdout": ""}` for script_run(flake8/pylint)
   - Auto-return simulated output for script_run(sim_main.py)
2. Check sim_main.py header contains `@ProjectTypes` + `@CoverageReport` comments
3. Check sim_main.py handles `--plain` flag correctly
4. Check all sent message JSON conforms to 02-protocol.md Schema
5. Verify mode: Construct start_phase (mode="verify", changed_files=["firmware/tasks/sensor.py"]) → verify only reads changed_files + sim_main.py
