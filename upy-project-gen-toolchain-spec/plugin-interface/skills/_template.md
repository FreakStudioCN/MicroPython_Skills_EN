# [Skill Name] Interface Definition

> Status: ⚠ Pending
>
> Filling Instructions: Fill in according to the 6 sections of this template. Change the status to ✅ Finalized after completion.
> This template is referenced by `skills/README.md`. Both plugin-side and server-side engineers should develop according to this document.

---

## I. Skill Overview

| Item | Content |
|------|---------|
| Phase | [analyze / select-hw / scaffold / generate / simulate / deploy / autofix / wiring / diagram / cold-driver] |
| Upstream Skill | [Which skill automatically enters after completion, or triggered manually by the user] |
| Downstream Skill | [Which skill to enter after completion] |
| One-line Responsibility | [Describe what this skill does in one sentence] |

---

## II. Plugin Input → Skill (P→S)

This is the data the plugin needs to provide to this skill.

| Input Item | Type | Required | Source | Description |
|------------|------|----------|--------|-------------|
| user_description | string | Yes | User input field | User's project description |
| pre_selected_board | object? | No | Board selector | Board pre-selected by the user in the plugin |
| preferences.mode | string | No | Plugin settings | "beginner" / "custom" |
| ... | ... | ... | ... | ... |

**pre_selected_board structure (when the user has pre-selected a board):**
```json
{
  "id": "esp32-devkit-v1",
  "display_name": "ESP32 DevKit V1",
  "mcu": "ESP32-WROOM-32",
  "chip_family": "esp32",
  "firmware_url": "https://micropython.org/download/ESP32_GENERIC/"
}
```

---

## III. Skill Output → Plugin (S→P)

This is the messages this skill will send to the plugin.

List them by execution step, annotating the message type and key content for each step.

### Step Message Table

| Step | Message Type | Trigger Condition | Key Content |
|------|--------------|-------------------|-------------|
| Step X: xxx | status_update | When execution starts | message: "Currently xxx..." |
| Step Y: xxx | approval_request | When user confirmation is needed | header: "xxx", items: [...] |
| ... | ... | ... | ... |
| Final | phase_complete | When phase completes | result: "success"/"failed", artifacts: [...] |

### approval_request Card Design

For each approval card, draw the plugin's rendering effect (ASCII diagram or description):

```
┌─────────────────────────────────────────┐
│  Card Title                             │
│                                         │
│  [Summary Information Area]             │
│                                         │
│  ☑ Option 1 — Description              │
│  ☑ Option 2 — Description              │
│                                         │
│  [+ Add]                                │
│                                         │
│  [Confirm Button]  [Cancel Button]      │
└─────────────────────────────────────────┘
```

### status_update List

List all progress message texts that this skill will emit:

| step_id | message | level | Trigger Timing |
|---------|---------|-------|----------------|
| ... | ... | ... | ... |

### phase_complete Artifacts

| Artifact Type | Title | Content Description |
|---------------|-------|---------------------|
| table | xxx | headers: [...], rows: [...] |
| file_tree | xxx | ... |
| ... | ... | ... |

---

## IV. SKILL.md Modification Points

> **This section is used for Phase B communication translation.** Phase A logic changes (process additions/deletions, step adjustments, branch corrections) should first be run through locally in Claude Code. After confirming the logic is correct, mechanically translate the communication method point by point according to this section. Do not mix logic changes and communication translation.

List the specific locations and content that need modification. Format:

| Modification Location | Current Behavior | Change To | Reason |
|-----------------------|------------------|-----------|--------|
| Step 2A Branching | AskUserQuestion(...) | approval_request(...) | Approval card replaces command-line Q&A |
| Step X: xxx | Bash(...) | device_command(...) | Pass through to plugin for execution |
| ... | ... | ... | ... |

---

## V. UI Components the Plugin Needs to Implement

| Component | Purpose | Reusable Protocol Message |
|-----------|---------|---------------------------|
| Progress Timeline | Display the status_update sequence of this skill | status_update |
| XXX Approval Card | xxx | approval_request |
| ... | ... | ... |

---

## VI. Independent Test Scenarios

### Plugin-side Testing (Without Server)

1. Manually construct the phase_complete message of this skill, confirm the result panel renders correctly
2. Manually construct the approval_request message of this skill, confirm the card interaction is correct

### Skill-side Testing (Without Plugin)

1. Use mock_plugin.py to simulate plugin responses, run through the complete skill flow
2. Confirm all emitted message JSON conforms to the 02-protocol.md Schema
