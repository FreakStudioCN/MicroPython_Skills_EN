# upy-autofix Interface Definition

> Status: ✅ Finalized
>
> Phase 6 — Error-library-driven interactive single-point troubleshooting. Match error_lib → adapt debug_steps → step-by-step automatic/manual execution → locate root cause → delegate upstream skill for fix → feed back into error library.

---

## I. Skill Overview

| Item | Content |
|------|---------|
| Phase | autofix |
| Upstream Skill | upy-deploy (auto-triggered on FAIL) or user clicks [Debug] button |
| Downstream Skills | upy-generate / upy-select-hw / upy-analyze (delegated fix); upy-deploy (verification); upy-simulate (optional PC verification) |
| One-line Responsibility | Query error library → generate troubleshooting plan → step-by-step guidance (auto-detect + manual cooperation) → locate root cause → fix → verify → feed back into knowledge base |

**Core Change:** From "auto-fix 3 times then give up" to "error-library-driven + structured debug_steps + step-by-step automatic/manual troubleshooting". The LLM only performs **matching + parameter adaptation + step scheduling**, not step generation.

---

## II. Plugin Input → Skill (P→S)

### Start autofix

```json
{
  "type": "start_phase",
  "phase": "autofix",
  "session_id": "uuid-xxx",
  "payload": {
    "mode": "auto",
    "error_context": { /* error_context carried by deploy phase_complete */ },
    "user_symptom": null,
    "user_suspect": null,
    "max_attempts": 3
  }
}
```

| Field | Type | Required | Source | Description |
|-------|------|----------|--------|-------------|
| `mode` | string | Yes | Trigger | `"auto"` — auto-triggered on deploy FAIL; `"manual"` — user clicks [Debug] button |
| `error_context` | object | Yes | deploy phase_complete | Contains traceback / file_path / line_number / repl_output / log_report |
| `user_symptom` | string? | No | User input | User's supplementary observation ("SHT30 module LED not lit") |
| `user_suspect` | string? | No | User input | User's suspected cause ("maybe breadboard contact is poor") |
| `max_attempts` | number | No | Plugin setting | Default 3 |

### Mid-process User Intervention

```json
{
  "type": "user_intervention",
  "payload": {
    "action": "pause",
    "note": "I just noticed the SDA Dupont wire was loose, I've reconnected it",
    "resume_action": "retry_current_step"
  }
}
```

| action | Description |
|--------|-------------|
| `pause` | Pause, continue after user provides information |
| `skip_step` | Skip current step |
| `abort` | Terminate troubleshooting, generate diagnostic bundle, enter manual mode |
| `add_note` | Do not pause, append observation record |

### User Manages Error Library

```json
{
  "type": "error_lib_update",
  "payload": {
    "action": "add",
    "entry": { /* complete error_lib entry structure */ }
  }
}
```

---

## III. Skill Output → Plugin (S→P)

### Message Sequence

```
Phase 0: User Input (when mode="manual")
  → approval_request #1: Error symptom input card (debug_symptom_input)

Phase 1: Query Library + Generate Troubleshooting Plan
  → status_update "Searching error library..."
  → file_operation(read) → error_lib.json (project-level)
  → file_operation(read) → error_lib.json (global-level)
  → status_update "✓ Matched 2 similar cases, highest score 95"
  → status_update "Adapting troubleshooting steps..."
  → phase_complete (debug_plan artifact)

Phase 2: Step-by-step Troubleshooting Execution (Loop)
  For each step:
    ├── auto_verify / auto_detect:
    │     → status_update "Step N/M: {title}"
    │     → device_command(action="exec", code=...)
    │     → device_result
    │     → approval_request: Result confirmation card (debug_step_result)
    │
    ├── user_measure / user_observe:
    │     → status_update "Step N/M: {title}"
    │     → approval_request: Guided operation card (debug_user_measure / debug_user_observe)
    │
    └── user_action:
          → status_update "Step N/M: {title}"
          → approval_request: Guided operation card (debug_user_action)
          → User completes action → device_command(action="exec") auto re-test
          → approval_request: Result confirmation card

  Branch based on step.on_pass / step.on_fail:
    → continue → next step
    → goto_step N → jump to step N
    → abort → exit loop, display troubleshooting guidance
    → resolve → exit loop, enter Phase 3

Phase 3: Fix + Verify + Record
  → status_update "Root cause located: {root_cause}"
  → (If code change needed) Delegate upy-generate / upy-select-hw
  → deploy(mode="incremental")
  → status_update "✓ Fix verification passed"
  → file_operation(write) → error_lib.json (update statistics)
  → phase_complete (result="success")

Phase 4: 3 Unsuccessful Troubleshooting Attempts
  → status_update "Automatic troubleshooting could not locate root cause, generating diagnostic bundle..."
  → file_operation(write) → diagnostic_bundle.json
  → phase_complete (result="failed", artifact=diagnostic_bundle)
  → approval_request: Diagnostic bundle export card (debug_bundle_export)
```

### Key approval_request Cards

#### Card #1 — Error Symptom Input (debug_symptom_input)

```
┌──────────────────────────────────────────┐
│  🔍 Debug Assistant                       │
│                                          │
│  Deployment failure detected:             │
│  ┌────────────────────────────────────┐  │
│  │ OSError: [Errno 19] ENODEV         │  │
│  │ I2C scan: []                       │  │
│  │ at firmware/tasks/sensor_task.py:23 │  │
│  └────────────────────────────────────┘  │
│                                          │
│  Please supplement the symptoms you observed: │
│  ┌────────────────────────────────────┐  │
│  │ (placeholder: Is module LED on? Checked wiring?)│
│  └────────────────────────────────────┘  │
│                                          │
│  Your suspected cause (optional):         │
│  ┌────────────────────────────────────┐  │
│  │ (placeholder: Power/Wiring/Module damage...) │
│  └────────────────────────────────────┘  │
│                                          │
│  [Start Troubleshooting]  [Skip, Auto-fix]│
└──────────────────────────────────────────┘
```

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "debug_symptom_input",
    "header": "Debug Assistant",
    "question": "Please supplement the observed error symptoms to help quickly locate the problem",
    "summary": {
      "error_preview": "OSError: [Errno 19] ENODEV\nI2C scan: []\nat firmware/tasks/sensor_task.py:23",
      "error_type": "OSError_19",
      "affected_device": "SHT30"
    },
    "items": [],
    "allow_add": false,
    "allow_remove": false,
    "multi_select": false,
    "actions": [
      { "label": "Start Troubleshooting", "value": "start_debug", "primary": true },
      { "label": "Skip, Auto-fix", "value": "skip_to_autofix" }
    ]
  }
}
```

#### Card #2 — Guided Operation (debug_user_measure)

```
┌──────────────────────────────────────────┐
│  Step 3/6: Measure I2C Bus Voltage        │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │  📐 Operation Guide                 │  │
│  │                                    │  │
│  │  Tool: Multimeter (DC Voltage DCV) │  │
│  │                                    │  │
│  │  1. Red probe to SDA pin (GPIO21)  │  │
│  │  2. Black probe to GND             │  │
│  │  3. Record voltage value           │  │
│  │  4. Same method for SCL pin (GPIO22)│  │
│  │                                    │  │
│  │  Normal value: 3.3V ± 0.3V         │  │
│  │  Acceptable: 3.0V ~ 3.6V           │  │
│  │  Abnormal: < 3.0V or > 3.6V        │  │
│  │                                    │  │
│  │  [I2C Bus Voltage Measurement Diagram]│
│  └────────────────────────────────────┘  │
│                                          │
│  SDA Pin Voltage (GPIO21):                │
│  [~3.3V Normal]  [< 3.0V Low]  [~0V No Voltage] │
│                                          │
│  SCL Pin Voltage (GPIO22):                │
│  [~3.3V Normal]  [< 3.0V Low]  [~0V No Voltage] │
│                                          │
│  [Cannot measure, skip]                   │
└──────────────────────────────────────────┘
```

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "debug_user_measure",
    "header": "Step 3/6: Measure I2C Bus Voltage",
    "question": "Please measure the I2C bus voltage with a multimeter and select the result",
    "summary": {
      "step_id": 3,
      "step_type": "user_measure",
      "expected_normal": "SDA and SCL both around 3.3V (with pull-up resistors)",
      "expected_abnormal": "Any pin voltage < 3.0V (missing pull-up) or ~0V (no pull-up)"
    },
    "guidance": {
      "tool": "Multimeter (DC Voltage DCV)",
      "steps": [
        "Red probe to SDA pin (GPIO21), black probe to GND, record voltage",
        "Red probe to SCL pin (GPIO22), black probe to GND, record voltage"
      ],
      "normal_range": { "min": 3.0, "max": 3.6, "unit": "V" },
      "diagram_ref": "i2c_bus_voltage_measure"
    },
    "items": [
      {
        "id": "sda_normal",
        "name": "SDA: ~3.3V Normal",
        "subtitle": "GPIO21 voltage in normal range",
        "meta": "",
        "selected": false
      },
      {
        "id": "sda_low",
        "name": "SDA: < 3.0V Low",
        "subtitle": "GPIO21 voltage abnormal",
        "meta": "",
        "selected": false
      },
      {
        "id": "sda_zero",
        "name": "SDA: ~0V No Voltage",
        "subtitle": "GPIO21 may be floating or shorted",
        "meta": "",
        "selected": false
      }
    ],
    "allow_add": false,
    "allow_remove": false,
    "multi_select": true,
    "actions": [
      { "label": "Confirm", "value": "confirm", "primary": true },
      { "label": "Cannot measure, skip", "value": "skip" }
    ]
  }
}
```

#### Card #3 — Result Confirmation (debug_step_result)

```
┌──────────────────────────────────────────┐
│  Step 2/6: I2C Bus Scan (Auto-executed)   │
│                                          │
│  Expected Behavior:                       │
│  ✓ I2C.scan() should return [0x3C, 0x44] │
│    (SSD1306 @ 0x3C, SHT30 @ 0x44)        │
│                                          │
│  Actual Output:                           │
│  ┌────────────────────────────────────┐  │
│  │ []                                 │  │
│  └────────────────────────────────────┘  │
│                                          │
│  ❌ No I2C devices detected               │
│                                          │
│  Do you see the output above?             │
│  [Yes, output is []]  [Not sure]          │
│                                          │
│  Supplementary observation (optional):    │
│  ┌────────────────────────────────────┐  │
│  │ (placeholder)                     │  │
│  └────────────────────────────────────┘  │
│                                          │
│  → Next step: Measure I2C bus voltage     │
└──────────────────────────────────────────┘
```

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "debug_step_result",
    "header": "Step 2/6: I2C Bus Scan",
    "question": "Do you see the expected output?",
    "summary": {
      "step_id": 2,
      "step_type": "auto_detect",
      "expected_normal": "I2C.scan() should return [0x3C, 0x44]",
      "actual_output": "[]",
      "verdict": "fail",
      "next_step": "Measure I2C bus voltage"
    },
    "items": [],
    "allow_add": false,
    "allow_remove": false,
    "multi_select": false,
    "actions": [
      { "label": "Yes, output as above", "value": "confirm_output", "primary": true },
      { "label": "Not sure", "value": "unsure" },
      { "label": "Add observation...", "value": "add_note" }
    ]
  }
}
```

### status_update List

| step_id | level | message | Trigger |
|---------|-------|---------|---------|
| search_lib | info | Searching error library... | Phase 1 start |
| lib_match | success | ✓ Matched N cases, highest score XX | Match success |
| lib_no_match | warn | ⚠ No match in error library, will analyze from scratch | No match |
| adapt_steps | info | Adapting troubleshooting steps... | Filling template parameters |
| plan_ready | success | ✓ Troubleshooting plan: M steps, estimated X minutes | Before Phase 2 |
| step_start | info | Step N/M: {title} | Each step start |
| step_auto_exec | info | Executing auto-detection... | auto_verify/auto_detect |
| step_auto_pass | success | ✓ {title} — Normal | Auto step pass |
| step_auto_fail | warn | ⚠ {title} — Abnormal | Auto step fail |
| step_user_wait | info | ⏳ Waiting for user action... | user_measure/action |
| step_user_done | success | ✓ User feedback received | User action complete |
| step_skip | info | ⛔ Skipping step N | User skip |
| root_cause_found | success | ✓ Root cause located: {description} | Phase 3 start |
| fix_delegate | info | Delegating fix to {skill}... | Delegate upstream skill |
| fix_verify | info | Verifying fix result... | deploy phase |
| fix_done | success | ✓ Fix verification passed | Verification passed |
| lib_update | info | Updating error library... | Writing error_lib |
| bundle_gen | info | Generating diagnostic bundle... | Phase 4 |
| bundle_done | success | ✓ Diagnostic bundle generated | Bundle complete |

### phase_complete

**PASS:**

```json
{
  "type": "phase_complete",
  "payload": {
    "phase": "autofix",
    "result": "success",
    "summary": "Root cause located: SDA/SCL missing pull-up resistors. Added 4.7kΩ pull-ups, I2C scan normal.",
    "next_phase": null,
    "artifacts": [
      {
        "type": "table",
        "title": "Troubleshooting Process",
        "headers": ["Step", "Type", "Title", "Result"],
        "rows": [
          ["1/6", "Auto", "Confirm MCU basic function", "✓ MCU_OK"],
          ["2/6", "Auto", "I2C bus scan", "✗ []"],
          ["3/6", "User Measure", "Measure I2C bus voltage", "✓ SDA:0.2V SCL:0.1V — Abnormal"],
          ["4/6", "User Action", "Add 4.7kΩ pull-up resistors", "✓ Done → Re-test SCAN:[0x3C,0x44]"]
        ]
      },
      {
        "type": "markdown",
        "title": "Root Cause Analysis",
        "content": "### Root Cause\n\nSDA/SCL pins missing external pull-up resistors. ESP32 internal pull-up ~45kΩ is too weak; I2C requires 4.7kΩ external pull-up to 3.3V.\n\nSymptom: I2C.scan() returns empty.\n\nFix: SDA(GPIO21)→4.7kΩ→3.3V, SCL(GPIO22)→4.7kΩ→3.3V.\n\nSource: Error library err_sht30_i2c_19 (verified, 13 successes)"
      }
    ],
    "warnings": [],
    "errors": [],
    "error_lib_updated": true,
    "matched_entry_id": "err_sht30_i2c_19"
  }
}
```

**FAIL (3 unsuccessful troubleshooting attempts):**

```json
{
  "type": "phase_complete",
  "payload": {
    "phase": "autofix",
    "result": "failed",
    "summary": "3 rounds of troubleshooting could not locate root cause. Diagnostic bundle generated for manual analysis.",
    "next_phase": null,
    "artifacts": [
      {
        "type": "table",
        "title": "Troubleshooting Process",
        "headers": ["Round", "Step", "Type", "Title", "Result"],
        "rows": [
          ["1", "1/6", "Auto", "MCU basic function", "✓"],
          ["1", "2/6", "Auto", "I2C scan", "✗ []"],
          ["1", "3/6", "User Measure", "Bus voltage", "✓ Normal 3.3V"],
          ["1", "4/6", "User Action", "Add pull-up", "✗ Still empty"],
          ["1", "5/6", "User Measure", "SHT30 power", "✓ Normal 3.3V"],
          ["1", "6/6", "User Action", "Replace sensor", "✗ Still empty"],
          ["2", "1/1", "Auto", "Lower I2C frequency", "✗ Still empty"],
          ["3", "1/2", "Auto", "Try SoftI2C", "✗ Still empty"],
          ["3", "2/2", "Auto", "Onboard LED", "✓"]
        ]
      },
      {
        "type": "markdown",
        "title": "LLM Analysis",
        "content": "### Ruled Out\n\n- I2C frequency (100kHz/400kHz both failed)\n- Pin configuration (SoftI2C also failed)\n- MCU power/reset (onboard LED normal)\n- I2C bus voltage (normal 3.3V)\n- Pull-up resistors (added)\n- Sensor power (normal 3.3V)\n- Sensor module (still failed after replacement)\n\n### Still Suspected\n\n- MCU I2C peripheral hardware damage (low probability)\n- Breadboard internal short (needs multimeter continuity check pin by pin)\n\n### Knowledge Gap\n\nCannot remotely distinguish between 'I2C peripheral damage' and 'breadboard hidden short'. Suggest cross-validating with a different MCU module."
      }
    ],
    "warnings": ["Troubleshooting steps exhausted, root cause not located"],
    "errors": ["Automatic troubleshooting unresolved"],
    "diagnostic_bundle_path": ".upy/diagnostic_bundles/bundle_20260617T103000.json",
    "error_lib_updated": true
  }
}
```

---

## IV. SKILL.md Modification Points

14 changes total:

| No. | Location | Current Behavior | Change To | Reason |
|-----|----------|-----------------|-----------|--------|
| 1 | Role Definition | Orchestration coordination layer, triage.py collects + LLM analyzes + delegates fix | **Error-library-driven interactive single-point troubleshooting**. LLM matches error_lib → adapts debug_steps → step-by-step execution → user cooperation | From "auto-fix 3 times" to "guided troubleshooting" |
| 2 | Prerequisites | triage.py + deploy_logs/ | Added `error_lib.json` (project-level `.upy/` + global-level `~/.upy/`) | Querying library is the first step |
| 3 | Step 1 | `python triage.py --log-dir ... --port COM3` | **Deleted**. Log parsing done by LLM directly from error_context. I2C scan moved into debug_steps auto_detect steps | Server does not run local scripts |
| 4 | Step 2 Analysis | 7 error categories → branch to different skills | **Phase 1: Query library for match**. Extract signature (error_type+keywords+device) → file_operation(read) error_lib.json → match scoring → take highest score entry → fill template parameters `{I2C_SCL}` `{I2C_SDA}` etc. → generate debug_plan | Error library driven, LLM does not generate steps |
| 5 | Step 2.5 Hardware Signal Verification | LLM generates sanity_config.json → `python hardware_sanity.py` → ask_user | **Merged into Phase 2 debug_steps execution loop**. auto_verify/auto_detect → device_command. user_measure/user_observe → approval_request | Unified into 6 step types |
| 6 | Step 2.5 User Feedback | hardware_sanity.py sets `_pending_question` → LLM calls AskUserQuestion | **Changed to approval_request cards**. user_feedback mode results directly mapped to debug_step_result / debug_user_measure cards | Plugin side renders approval cards uniformly |
| 7 | Step 3 Delegation | `Skill("upy-generate")` / `Skill("upy-select-hw")` | **Server-side internal call** + phase switch. LLM starts generate with mode="fix" + error_context, phase field switch notifies plugin | Same-process call, phase lets plugin perceive progress |
| 8 | Step 4 Verification | `Skill("upy-simulate")` → `Skill("upy-deploy")` | deploy(mode="incremental", changed_files). Before each verification, send approval_request showing "expected normal behavior", after execution send result confirmation card | User participates in verification |
| 9 | Step 5 Hardware Troubleshooting Guidance | Plain text Chinese guidance | **Integrated into debug_steps fail_guidance field**. On abort, display structured guidance card (including wiring diagram references) | Not plain text, includes diagrams |
| 10 | Step 6 3 Failures | git rollback + plain text bottleneck report | **Generate diagnostic_bundle.json** (structured JSON, containing all attempts + code snapshots + LLM analysis + knowledge gaps). **No auto-rollback**, preserve scene for manual analysis | For human + LLM joint analysis, not simple abandonment |
| 11 | Step 7 Error Recording | Append to `logs/error_report.json` | **Write to error_lib.json**. Success → update success_count + avg_steps_to_resolve. Failure → append suspected entry. After manual resolution → write verified entry | Unified as error_lib |
| 12 | **New** Phase 0 | None | `approval_request` error symptom input card: REPL output pre-filled + user supplements observations + suspected cause. Displayed when mode="manual" | User participation entry point |
| 13 | **New** Error Library Self-Evolution | None | After troubleshooting ends → update entry statistics. New steps added by LLM → append to entry debug_steps. After 3 total failures, user inputs root cause → create verified entry | Knowledge accumulation |
| 14 | **New** Adaptation Parameter Extraction | None | LLM extracts actual pin numbers/addresses from board.py + manifest, fills variables like `{I2C_SCL}`, `{SHT30_ADDR}` in debug_steps templates | Generic template → project-specific |

---

## V. Validation Scripts

### validate_error_lib.py (New)

**Path:** `G:\MicroPython_Skills\upy-autofix\scripts\validate_error_lib.py`

Validates error_lib.json structural integrity. Runs automatically after LLM writes to error_lib.

| Check Item | Description |
|------------|-------------|
| JSON Syntax | Parsable |
| Required Fields | Each entry contains id / signature / classification / debug_steps / metadata |
| debug_steps Completeness | Each step contains step_id / type / title / expected_normal / on_pass / on_fail |
| step.type Enumeration | `auto_verify` / `auto_detect` / `user_measure` / `user_observe` / `user_action` / `info_only` |
| on_pass/on_fail Enumeration | `continue` / `goto_step` / `abort` / `retry_step` / `resolve` |
| goto_step/retry_step Target Exists | `on_pass_target` / `on_fail_target` / `retry_step_id` points to an existing step_id |
| step_id Unique | No duplicates within the same entry |
| certainty Enumeration | `verified` / `suspected` / `speculative` |
| source Enumeration | `manual` / `auto` |

stdout:
```json
{ "status": "pass", "errors": [], "warnings": [] }
```

### validate_diagnostic_bundle.py (New)

**Path:** `G:\MicroPython_Skills\upy-autofix\scripts\validate_diagnostic_bundle.py`

Validates diagnostic bundle structural integrity.

| Check Item | Description |
|------------|-------------|
| JSON Syntax | Parsable |
| Required Top-level Fields | bundle_id / project / error_summary / attempts / llm_analysis |
| attempts Sequential | attempt_number increments from 1 |
| error_summary.across_attempts | Matches number of attempts |
| code_snapshot File Exists | Referenced file paths exist in the project directory |

### triage.py (Refactored)

**Path:** `G:\MicroPython_Skills\upy-autofix\scripts\triage.py`

| Change | Content |
|--------|---------|
| Removed `--port` / `--sda` / `--scl` | No longer calls mpremote |
| Added `--input` | Reads log text from stdin (server reads via file_operation then pipes), replaces `--log-dir` |
| Keep `parse_errors()` | Regex matching logic unchanged |
| Keep `--snapshot` / `--rollback` | Git operations unchanged |
| Added `--validate-lib` | Calls validate_error_lib.py to validate error_lib.json |

### hardware_sanity.py (Minor Modification)

**Path:** `G:\MicroPython_Skills\upy-autofix\scripts\hardware_sanity.py`

| Change | Content |
|--------|---------|
| `_pending_question` field extended | Added `expected_behavior` + `abnormal_options[]` fields for LLM to generate approval_request |
| Added `--stdin-config` | Reads config JSON from stdin, avoids server writing files to disk |

---

## VI. Template Files

### error_lib.json

**Path:** `G:\MicroPython_Skills\upy-autofix\templates\error_lib.json`

Empty template, copied to `{project}/.upy/error_lib.json` by `init_scaffold.py`.

```json
{
  "$schema": "https://upy-toolchain/error_lib/v1",
  "version": "1.0",
  "updated_at": "",
  "entries": []
}
```

**Database Construction Mechanism** — See [error_lib.json Construction Specification](#viii-error_libjson-construction-specification).

### diagnostic_bundle_schema.json

**Path:** `G:\MicroPython_Skills\upy-autofix\templates\diagnostic_bundle_schema.json`

Diagnostic bundle schema, referenced by validate_diagnostic_bundle.py. Dynamically generated by LLM when autofix fails after 3 attempts.

---

## VII. Plugin-side UI Components

| Component | Corresponding Message | Description |
|-----------|-----------------------|-------------|
| [Debug] Button | Triggers start_phase(mode="manual") | Appears in result panel after deploy FAIL |
| Error Symptom Input Card | approval_request `debug_symptom_input` | REPL output pre-filled + user supplements |
| Troubleshooting Plan Panel | phase_complete artifact `debug_plan` | Step list + match source + estimated time |
| Guided Operation Card | approval_request `debug_user_measure` / `debug_user_action` | Contains operation steps/wiring diagram/normal range |
| Result Confirmation Card | approval_request `debug_step_result` | "Do you see the expected behavior?" |
| Diagnostic Bundle Export Card | approval_request `debug_bundle_export` | After 3 unsuccessful troubleshooting attempts |
| Error Library Management Panel | error_lib_update | Browse/add/edit/delete entries |

---

## VIII. error_lib.json Construction Specification

### File Hierarchy

```
Project-level: {project_dir}/.upy/error_lib.json    ← Specific to current project, created by init_scaffold from template
Global-level: ~/.upy/error_lib.json                 ← Shared across projects, managed by plugin sync
```

### Library Query Priority

```
1. Query project-level first → match found (score ≥ 40) → use directly
2. No match in project-level → query global-level → match found → use
3. Both levels match → merge results, sort by score, deduplicate (same id, take higher score)
4. No match in either level → LLM generates debug_steps from scratch (fallback)
```

### Data Sources

| Source | Write Location | Trigger Condition | certainty |
|--------|----------------|-------------------|-----------|
| User manual upload (plugin [Upload Case] button) | Global-level | User action | `verified` |
| autofix troubleshooting success (reusing existing entry) | Project-level | success_count++ / avg_steps update | Unchanged |
| autofix troubleshooting success (LLM custom new steps) | Project-level | New steps appended to entry debug_steps end | Unchanged |
| autofix troubleshooting success (brand new problem) | Project-level | Create new entry | `suspected` |
| autofix 3 total failures + manual resolution | Global-level | Created after user inputs root cause | `verified` |

### Promotion to Global Level

When a project-level entry simultaneously meets the following conditions, LLM suggests promotion:

```
success_count ≥ 3  AND  certainty = "verified"  AND  source = "auto"
```

→ `approval_request`: "This case has been verified 3 times. Promote to global error library?" → User confirms → `error_lib_update(action="promote", entry_id="...")`.

### Entry Lifecycle

```
User upload (manual, verified)
  → Matched and used by autofix → success_count++

autofix auto-create (auto, suspected)
  → Verified successful by subsequent troubleshooting 1 time → suspected → success_count=1
  → Successful 3 times → LLM suggests promotion → User confirms → verified + moved to global level

verified entry fails 1 time
  → fail_count++ → still verified (sporadic)
  → Fails 3 consecutive times → downgraded to suspected → LLM reviews debug_steps
```

### Match Scoring Algorithm

```
score = (regex_match ? 50 : 0)
      + (keyword_intersection / total_keywords * 30)
      + (match_device_models intersection non-empty ? 10 : 0)
      + (match_bus_types intersection non-empty ? 10 : 0)
      + certainty_bonus (verified=20, suspected=10, speculative=0)
      + min(success_count, 20)
```

- score ≥ 80 → High confidence, execute directly, do not show troubleshooting plan card (reduce interaction)
- score 40~79 → Medium confidence, show troubleshooting plan card for user confirmation
- score < 40 → Low confidence/no match, LLM generates debug_steps from scratch + show plan card

### Template Parameter Filling

Placeholders in the form `{VAR}` in debug_steps are automatically filled by the LLM from the project context:

| Template Variable | Source | Example |
|-------------------|--------|---------|
| `{I2C_SCL}` | firmware/board.py → I2C_SCL | `22` |
| `{I2C_SDA}` | firmware/board.py → I2C_SDA | `21` |
| `{I2C_BUS_ID}` | firmware/board.py | `0` |
| `{SHT30_ADDR}` | project-manifest.json → devices[].address | `0x44` |
| `{MCU_LED_PIN}` | boards/*.json → onboard_led | `2` |
| `{ALL_I2C_ADDRS}` | project-manifest.json → devices[].address concatenation | `[0x3C, 0x44]` |
| `{DRIVER_MODULE}` | project-manifest.json → devices[].driver.module | `sht30` |
| `{DRIVER_CLASS}` | project-manifest.json → devices[].driver.class | `SHT30` |

---

## IX. Independent Test Scenarios

### Plugin-side Tests (No Server)

1. Manually send approval_request `debug_symptom_input` → Verify error symptom input card interaction
2. Manually send phase_complete (debug_plan artifact) → Verify troubleshooting plan panel rendering
3. Manually send approval_request `debug_user_measure` → Verify guided operation card (wiring diagram/normal range/options)
4. Manually send approval_request `debug_step_result` → Verify result confirmation card
5. Simulate complete troubleshooting sequence: device_command → device_result → approval_request confirm → next step
6. Manually send phase_complete (FAIL + diagnostic_bundle_path) → Verify diagnostic bundle export card

### Skill-side Tests (No Plugin)

1. Prepare error_lib.json (containing 3 test entries), construct start_phase(mode="auto", error_context=OSError_19)
2. Verify match scoring: confirm err_sht30_i2c_19 gets the highest score
3. Verify parameter filling: check that `{I2C_SDA}` in debug_steps is replaced with actual value
4. Mock user responses to each approval_request, verify on_pass/on_fail branching logic
5. Verify PASS path: troubleshoot to root_cause → delegate generate → deploy → error_lib update
6. Verify FAIL path: steps exhausted → generate diagnostic_bundle → phase_complete(failed)
7. Run validate_error_lib.py to confirm LLM-written entry structure is valid
