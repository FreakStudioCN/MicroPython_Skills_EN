# upy-scaffold Interface Definition

> Status: ✅ Finalized
>
> Phase 3 — Project skeleton generation. Reads the project-manifest.json from the select-hw phase and generates a complete firmware/ directory skeleton according to the scheduling mode.

---

## I. Skill Overview

| Item | Content |
|------|---------|
| Phase | scaffold |
| Upstream Skill | upy-select-hw (automatic entry) or incremental trigger from any phase (user adds device) |
| Downstream Skill | upy-generate |
| One-line Responsibility | Determine scheduling mode → Render templates → Generate complete firmware/ skeleton (no business logic) |

**Core Constraint:** Do not write business tasks, fill driver code, or convert async drivers. Only build the skeleton; business logic is left for upy-generate.

**Two Operating Modes:**

| Mode | Trigger | Behavior |
|------|---------|----------|
| `full` | upy-select-hw completes | Generate a complete firmware/ skeleton from scratch |
| `incremental` | User adds a device in a subsequent phase | Generate only a `drivers/<name>_driver/__init__.py` stub for the new device |

---

## II. Plugin Input → Skill (P→S)

The plugin sends **1 message** to start this skill:

```json
{
  "type": "start_phase",
  "phase": "scaffold",
  "session_id": "uuid-xxx",
  "payload": {
    "mode": "full",
    "manifest": "{...complete project-manifest.json (phase: select-hw)...}",
    "new_devices": []
  }
}
```

| Field | Type | Required | Source | Description |
|-------|------|----------|--------|-------------|
| `mode` | string | Yes | Server determines | `"full"` for fresh generation / `"incremental"` for incremental stub |
| `manifest` | object | Yes | upy-select-hw's phase_complete | Complete project-manifest.json |
| `new_devices` | array | Required for incremental | User-added device list | `[{name, driver: {source, install_cmd}}]` |

**Mode Determination Logic (Server-Side):**
- `manifest.phase === "select-hw"` and first entry → `full`
- User clicks "Add Device" in a subsequent phase → after select-hw incrementally assigns pins, scaffold receives `incremental`

---

## III. Skill Output → Plugin (S→P)

### Message Sequence

```
full mode:
  Step 1 Approval for Configuration
    → approval_request #1: Combined card (scheduling mode + extra modules + custom files)

  Step 2 Documentation Reference
    → [Internal] WebFetch asyncio/_thread official docs (server has network, invisible to plugin)

  Step 3 Generate Skeleton
    → status_update "Rendering board.py..."
    → status_update "Rendering conf.py / boot.py..."
    → status_update "Rendering main.py (mode: timer)..."
    → status_update "Copying lib/ base libraries..."
    → status_update "Generating drivers/ stubs..."
    → status_update "Copying tools/ deployment tools..."
    → file_operation × N (one write message per generated file)

  Step 4 Validation
    → script_run: flake8

  Step 5 Output
    → phase_complete: Result panel

incremental mode:
    → status_update "Generating driver stub for new device..."
    → file_operation × 1 (only writes drivers/<name>_driver/__init__.py)
    → phase_complete
```

### Message Details

#### approval_request #1 — Combined Approval Card

Combines the three AskUserQuestions from the current SKILL.md (scheduling mode + extra modules + custom files) into one card with three sections.

```
┌──────────────────────────────────────────┐
│  Project Skeleton Configuration          │
│                                          │
│  ▸ Scheduling Mode (Single Select)       │
│  ◉ Timer tick (Recommended)              │
│     ISR counting + main loop polling, suitable for pure sensor acquisition │
│  ○ asyncio                               │
│     uasyncio coroutines, suitable for WiFi / LCD projects │
│  ○ _thread                               │
│     Multi-threading, suitable for blocking operations │
│                                          │
│  ▸ Extra Modules (Multi-Select)          │
│  ☑ Logging System (lib/logger/*)         │
│  ☑ Deployment Tools (tools/flash_device.py) │
│  ☐ Performance Timer (lib/time_helper.py) │
│  ☐ Maintenance Tasks (tasks/maintenance.py) │
│  ☐ PC Log Reader (tools/read_device_log.py) │
│                                          │
│  ▸ Custom Files (Optional)               │
│  [+ Add Custom Directory/File]           │
│                                          │
│  [Confirm, Start Skeleton Generation]  [Modify Configuration] │
└──────────────────────────────────────────┘
```

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "scaffold_config",
    "header": "Project Skeleton Configuration",
    "question": "Select scheduling mode and modules to inject",
    "summary": {
      "project_name": "Temperature and Humidity Monitoring Alarm",
      "mcu": "ESP32 DevKit V1"
    },
    "items": [
      {
        "id": "mode_timer",
        "name": "Timer tick (Recommended)",
        "subtitle": "ISR counting + main loop polling, suitable for pure sensor acquisition",
        "meta": "★ Recommended",
        "selected": true,
        "group": "scheduler_mode"
      },
      {
        "id": "mode_async",
        "name": "asyncio",
        "subtitle": "uasyncio coroutines, suitable for WiFi / LCD / LVGL projects",
        "meta": "",
        "selected": false,
        "group": "scheduler_mode"
      },
      {
        "id": "mode_thread",
        "name": "_thread",
        "subtitle": "Multi-threading, suitable for blocking operations",
        "meta": "",
        "selected": false,
        "group": "scheduler_mode"
      },
      {
        "id": "module_logger",
        "name": "Logging System",
        "subtitle": "lib/logger/* — logging + rotating_logger, device-side log recording and rotation",
        "meta": "Recommended",
        "selected": true,
        "group": "extra_modules"
      },
      {
        "id": "module_flash",
        "name": "Deployment Tools",
        "subtitle": "tools/flash_device.py — mpy compilation + firmware flashing + file upload",
        "meta": "Recommended",
        "selected": true,
        "group": "extra_modules"
      },
      {
        "id": "module_time_helper",
        "name": "Performance Timer",
        "subtitle": "lib/time_helper.py — timed_function / timed_coro decorators",
        "meta": "",
        "selected": false,
        "group": "extra_modules"
      },
      {
        "id": "module_maintenance",
        "name": "Maintenance Tasks",
        "subtitle": "tasks/maintenance.py — GC check + idle callback",
        "meta": "",
        "selected": false,
        "group": "extra_modules"
      },
      {
        "id": "module_log_tools",
        "name": "PC Log Tools",
        "subtitle": "tools/read_device_log.py + log_report.py — PC-side log reading and JSON report",
        "meta": "",
        "selected": false,
        "group": "extra_modules"
      }
    ],
    "allow_add": true,
    "allow_remove": false,
    "multi_select": true,
    "item_groups": {
      "scheduler_mode": {"multi_select": false, "label": "Scheduling Mode"},
      "extra_modules": {"multi_select": true, "label": "Extra Modules"}
    },
    "actions": [
      { "label": "Confirm, Start Skeleton Generation", "value": "confirm", "primary": true },
      { "label": "Modify Configuration", "value": "modify" }
    ]
  }
}
```

**item_groups Field Description (New):**

Groups items by `group` for rendering; different groups can have different selection modes:
- `scheduler_mode`: `multi_select: false` → single select (radio button)
- `extra_modules`: `multi_select: true` → multi-select (checkbox)

**Scheduling Mode Recommendation Rules (LLM determines server-side based on manifest, only marks ★ Recommended):**

| Condition | Recommendation |
|-----------|----------------|
| devices includes display with LVGL | async |
| requirements.network = wifi | async |
| Default | timer |

#### status_update List

| step_id | message | level | Trigger |
|---------|---------|-------|---------|
| scaffold_start | Generating project skeleton... | info | Step 3 start |
| render_board | Rendering board.py (pin mapping + query functions)... | info | Generating board.py |
| render_conf | Rendering conf.py / boot.py... | info | Generating config files |
| render_main | Rendering main.py (mode: timer)... | info | Generating entry file |
| copy_lib | Copying lib/ base libraries (logger + scheduler)... | info | Copying template files |
| gen_drivers | Generating drivers/ stubs (3 devices)... | info | Generating driver stubs |
| copy_tools | Copying tools/ deployment tools... | info | Copying PC tools |
| scaffold_lint | Running flake8 validation... | info | Validation start |
| scaffold_lint_ok | ✓ flake8 validation passed | success | Validation passed |
| scaffold_lint_warn | ⚠ flake8 found N issues, auto-fixed | warn | Issues found but fixable |
| scaffold_done | ✓ Skeleton generation complete: 18 files, 8 directories | success | All complete |
| incremental_stub | Generating driver stub for new device DHT22... | info | incremental mode |
| incremental_done | ✓ DHT22 driver stub generated | success | incremental complete |

#### file_operation Sequence

The server first runs `init_scaffold.py` (stdin reads manifest, stdout outputs JSON), then sends each file one by one:

```json
{
  "type": "file_operation",
  "payload": {
    "op_id": "scaffold_fo_001",
    "op": "write",
    "path": "firmware/board.py",
    "content": "# -*- coding: utf-8 -*-\n# @Generated : upy-scaffold\n...",
    "encoding": "utf-8"
  }
}
```

**Complete File List for full Mode (timer mode + all modules):**

| # | File Path | Generation Method | Description |
|---|-----------|-------------------|-------------|
| 1 | `firmware/board.py` | Template rendering | Pin mapping BOARDS dictionary + query functions |
| 2 | `firmware/conf.py` | Template rendering | Sampling rate / logging / watchdog constants |
| 3 | `firmware/boot.py` | Template rendering | WDT + emergency_exception_buf |
| 4 | `firmware/main.py` | Template rendering | Hardware instantiation + scheduler framework by mode |
| 5 | `firmware/lib/logger/logging.py` | Pure copy | Core logging |
| 6 | `firmware/lib/logger/rotating_logger.py` | Pure copy | Rotating log |
| 7 | `firmware/lib/logger/__init__.py` | Pure copy | Logger package export |
| 8 | `firmware/lib/scheduler/timer_sched.py` | Pure copy | Timer scheduler (timer mode only) |
| 9 | `firmware/lib/scheduler/__init__.py` | Generated | `from .timer_sched import Scheduler` |
| 10 | `firmware/lib/time_helper.py` | Pure copy | Performance timing decorators (optional) |
| 11 | `firmware/tasks/maintenance.py` | Pure copy | GC check + idle callback (optional) |
| 12 | `firmware/tasks/__init__.py` | Generated | `# Tasks package` |
| 13~N | `firmware/drivers/<name>_driver/__init__.py` | Generated | One stub per device, with TODO comments |
| N+1 | `tools/flash_device.py` | Pure copy | mpy compilation + flashing + upload |
| N+2 | `tools/read_device_log.py` | Pure copy | PC-side device log reading |
| N+3 | `tools/log_report.py` | Pure copy | Log → JSON report |
| N+4 | `README.md` | Template rendering | Project name + BOM table + pin table |
| N+5 | `LICENSE` | Generated | MIT |
| N+6 | `.flake8` | Generated | F821/F401 exemptions + max-line=120 |
| N+7~14 | `host/.gitkeep` etc. | Generated | Placeholder files (8 directories) |
| — | `.upy/schemas/project-manifest.schema.json` | Pure copy | Manifest validation schema |
| — | `.upy/schemas/wiring.schema.json` | Pure copy | wiring.json validation schema |
| — | `.upy/schemas/diagram.schema.json` | Pure copy | diagram.json validation schema |
| — | `.upy/schemas/diagnostic_bundle.schema.json` | Pure copy | Diagnostic bundle validation schema |
| — | `.upy/scripts/validate_json.py` | Pure copy | Generic JSON Schema validator (shared by wiring + diagram + autofix) |
| — | `.upy/scripts/init_scaffold.py` | Pure copy | Skeleton generation script (used by this skill) |
| — | `.upy/scripts/download_drivers.py` | Pure copy | Driver download (used by generate) |
| — | `.upy/scripts/render_wiring_local.py` | Pure copy | Wiring diagram rendering (used by wiring) |
| — | `.upy/scripts/render_diagram_local.py` | Pure copy | Architecture diagram rendering (used by diagram) |
| — | `.upy/scripts/extract_pdf.py` | Pure copy | PDF text extraction (used by gen-driver) |
| — | `.upy/scripts/convert_arduino.py` | Pure copy | Arduino API mapping (used by gen-driver) |
| — | `.upy/scripts/flash_device.py` | Pure copy | Flashing + verification (used by deploy) |
| — | `.upy/scripts/read_device_log.py` | Pure copy | Device log reading (used by deploy) |
| — | `.upy/scripts/run_on_device.py` | Pure copy | REPL execution + capture (shared by gen-driver + deploy) |
| — | `.upy/scripts/hardware_sanity.py` | Pure copy | Hardware signal validation (used by autofix) |
| — | `.upy/scripts/triage.py` | Pure copy | Automatic troubleshooting (used by autofix) |
| — | `.upy/error_lib.json` | Pure copy | Error library template (used by autofix) |

#### script_run — flake8 Validation

```json
{
  "type": "script_run",
  "payload": {
    "script_id": "scaffold_lint_001",
    "interpreter": "python",
    "script": "flake8",
    "args": ["firmware/", "tools/", "--max-line-length=120"],
    "cwd": "{project_dir}",
    "timeout_ms": 15000
  }
}
```

#### phase_complete

```json
{
  "type": "phase_complete",
  "payload": {
    "phase": "scaffold",
    "result": "success",
    "summary": "Project skeleton generation complete: timer mode, 18 files, 8 directories",
    "next_phase": "generate",
    "artifacts": [
      {
        "type": "file_tree",
        "title": "Project Structure",
        "tree": {
          "firmware": {
            "board.py": "file",
            "conf.py": "file",
            "boot.py": "file",
            "main.py": "file",
            "lib": {
              "logger": {
                "logging.py": "file",
                "rotating_logger.py": "file",
                "__init__.py": "file"
              },
              "scheduler": {
                "timer_sched.py": "file",
                "__init__.py": "file"
              },
              "time_helper.py": "file"
            },
            "tasks": {
              "maintenance.py": "file",
              "__init__.py": "file"
            },
            "drivers": {
              "sht30_driver": { "__init__.py": "file" },
              "ssd1306_driver": { "__init__.py": "file" },
              "buzzer_driver": { "__init__.py": "file" }
            },
            "assets": {}
          },
          "tools": {
            "flash_device.py": "file",
            "read_device_log.py": "file",
            "log_report.py": "file"
          },
          "test": { "device": {}, "pc": {} },
          "host": {},
          "build": { "firmware": {}, "mpy": {} }
        }
      }
    ],
    "warnings": [],
    "errors": [],
    "manifest_content": "{Complete updated project-manifest.json JSON text}"
  }
}
```

**manifest_content New/Updated Fields:**
- `phase`: `"scaffold"`
- `scaffold_mode`: `"timer"` / `"async"` / `"thread"`
- `scaffold_modules`: `["logger", "flash_device", ...]`

---

## IV. SKILL.md Modification Points

7 changes total, arranged by execution step:

| No. | Location | Current Behavior | Change To | Reason |
|-----|----------|------------------|-----------|--------|
| 1 | Pre-checks | `python --version` + `python -c "import flake8"` | Remove. Dependency checks guaranteed by server environment | Plugin user cannot see server environment |
| 2 | Step 1A | AskUserQuestion single-select scheduling mode | Merge into `scheduler_mode` group of approval_request #1 (single select, with recommendation markers) | 3 questions merged into 1 card |
| 3 | Step 1B | AskUserQuestion multi-select extra modules | Merge into `extra_modules` group of the same approval_request (multi-select) | Same as above |
| 4 | Step 1C | AskUserQuestion custom files | Merge into the same approval_request, implemented via `allow_add: true` + input field | Same as above |
| 5 | Step 2 | WebFetch asyncio/_thread official docs | Unchanged. Server has network, invisible to plugin | No change needed |
| 6 | Step 3 | `python init_scaffold.py --project-dir {dir} --mode {mode}` writes to local disk | `python init_scaffold.py --mode {mode} --manifest - < manifest.json` (stdin in, stdout out JSON). Server parses JSON and sends `file_operation` sequence to plugin | Server does not write to local disk |
| 7 | New incremental | No such mode | `--mode incremental --new-devices '[{...}]'` only generates `drivers/<name>_driver/__init__.py` stub for new device | Support adding devices during deploy phase |

---

## V. Template Files and Script Changes

### 5.1 init_scaffold.py Changes

**Path:** `G:\MicroPython_Skills\upy-scaffold\scripts\init_scaffold.py`

| Change | Content |
|--------|---------|
| Input method | `--project-dir` changed to `--manifest -` (reads manifest JSON from stdin) |
| Output method | No longer writes to disk. Outputs JSON to stdout: `{phase, mode, directories[], files[{path, content, encoding}], summary}` |
| Template engine | Introduce `string.Template` (Python standard library, zero extra dependencies), replacing `lines.append()` concatenation in 5 `generate_*` functions |
| Incremental mode | New `--mode incremental --new-devices '[{name, driver}]'` |
| flake8 removal | Remove `subprocess.run(flake8)` at end of script, replaced by Phase 4's `script_run` message trigger |

**string.Template Replacement Example:**

```python
from string import Template

def render_template(tmpl_name, variables):
    tmpl_path = os.path.join(TEMPLATES_DIR, tmpl_name + ".tmpl")
    with open(tmpl_path, "r", encoding="utf-8") as f:
        tmpl = Template(f.read())
    return tmpl.safe_substitute(variables)
```

**init_scaffold.py Core Flow (Pseudo-code):**

```python
def main():
    args = parse_args()
    manifest = json.load(sys.stdin)

    variables = extract_variables(manifest)  # Extract template variables from manifest
    files = []
    dirs = []

    # 1. Render template files
    for tmpl in ["firmware/board.py", "firmware/conf.py", "firmware/boot.py",
                 f"firmware/main_{mode}.py", "firmware/README.md"]:
        content = render_template(tmpl, variables)
        files.append({"path": tmpl, "content": content, "encoding": "utf-8"})

    # 2. Pure copy files (determined by mode and user selections)
    for src in COPY_FILES[mode]:
        content = read_raw(os.path.join(TEMPLATES_DIR, src))
        files.append({"path": "firmware/" + src, "content": content, "encoding": "utf-8"})

    # 3. Generate driver stubs
    for device in manifest["devices"]:
        stub = f"# {device['name']} driver stub\n# TODO: upy-generate fills this\n"
        name = safe_var_name(device["name"])
        files.append({"path": f"firmware/drivers/{name}_driver/__init__.py",
                       "content": stub, "encoding": "utf-8"})
        dirs.append(f"firmware/drivers/{name}_driver")

    # 4. Other generated files
    files.append({"path": ".flake8", "content": generate_flake8(), "encoding": "utf-8"})
    files.append({"path": "LICENSE", "content": generate_license(), "encoding": "utf-8"})

    # 5. Collect directories
    dirs += infer_directories(files)

    # 6. Output JSON to stdout
    output = {
        "phase": "scaffold",
        "mode": mode,
        "scaffold_mode": mode,
        "directories": sorted(set(dirs)),
        "files": files,
        "summary": f"Generated {len(files)} files, {len(dirs)} directories"
    }
    json.dump(output, sys.stdout, ensure_ascii=False, indent=2)
```

### 5.2 New Template Files (7 .py.tmpl)

**Path:** `G:\MicroPython_Skills\upy-scaffold\templates\firmware\`

| Template File | Purpose | Key Variables |
|---------------|---------|---------------|
| `board.py.tmpl` | Pin mapping constants | `${MCU_MODEL}` `${BOARD_ID}` `${I2C_PINS_BLOCK}` `${FIXED_PINS_BLOCK}` `${I2C_FREQ}` `${UART_BAUD}` `${BOOT_SENSITIVE_LIST}` `${FLASH_PINS_LIST}` `${INPUT_ONLY_LIST}` |
| `conf.py.tmpl` | Project configuration constants | `${PROJECT_NAME}` `${MCU_MODEL}` `${FW_VERSION}` `${SAMPLE_INTERVAL_MS}` `${LOG_DIR}` `${LOG_LEVEL}` |
| `boot.py.tmpl` | Boot sequence | `${GENERATED_AT}` (almost no variables, only emergency_exception_buf template) |
| `main_timer.py.tmpl` | Timer mode entry | `${PROJECT_NAME}` `${I2C_INIT_BLOCK}` `${GPIO_INIT_BLOCK}` |
| `main_async.py.tmpl` | asyncio mode entry | Same as above |
| `main_thread.py.tmpl` | _thread mode entry | Same as above |
| `README.md.tmpl` | Project README | `${PROJECT_NAME}` `${MODE}` `${MCU_MODEL}` `${MCU_BOARD}` `${FIRMWARE_URL}` `${BOM_TABLE_ROWS}` `${PINOUT_TABLE_ROWS}` `${TOTAL_PRICE}` |

Multi-line block variables (`${I2C_INIT_BLOCK}`, `${GPIO_INIT_BLOCK}`, `${BOM_TABLE_ROWS}`, `${PINOUT_TABLE_ROWS}`, `${FIXED_PINS_BLOCK}`, `${I2C_PINS_BLOCK}`) are pre-computed as strings by the Python script from manifest.pinout/manifest.bom. The template contains a single placeholder.

### 5.3 Template Directory Structure

```
upy-scaffold/templates/
├── firmware/                     ← 7 .py.tmpl templates (need rendering)
│   ├── board.py.tmpl
│   ├── conf.py.tmpl
│   ├── boot.py.tmpl
│   ├── main_timer.py.tmpl
│   ├── main_async.py.tmpl
│   ├── main_thread.py.tmpl
│   └── README.md.tmpl
├── lib/                          ← Pure copy (9 .py files, no variables)
│   ├── logger/
│   │   ├── logging.py
│   │   ├── rotating_logger.py
│   │   └── __init__.py
│   ├── scheduler/
│   │   └── timer_sched.py
│   └── time_helper.py
├── tasks/
│   └── maintenance.py            ← Pure copy
└── pc/                           ← Pure copy
    ├── flash_device.py
    ├── read_device_log.py
    └── log_report.py
```

The 9 pure-copy `.py` files are complete, runnable code. They contain no `${variable}` placeholders; the script reads and outputs them as-is.

### 5.4 init_scaffold.py stdout JSON Specification

```json
{
  "phase": "scaffold",
  "mode": "timer",
  "scaffold_mode": "timer",
  "directories": [
    "firmware/drivers/buzzer_driver",
    "firmware/drivers/sht30_driver",
    "firmware/drivers/ssd1306_driver",
    "firmware/lib/logger",
    "firmware/lib/scheduler",
    "firmware/tasks",
    "host",
    "test/device",
    "test/pc",
    "tools",
    ".upy",
    ".upy/schemas",
    ".upy/scripts"
  ],
  "files": [
    {
      "path": "firmware/board.py",
      "content": "# -*- coding: utf-8 -*-\n# @Generated : upy-scaffold\n...",
      "encoding": "utf-8"
    },
    {
      "path": "firmware/main.py",
      "content": "from machine import Pin, I2C\n...",
      "encoding": "utf-8"
    }
  ],
  "summary": "Generated 18 files, 10 directories"
}
```

The server iterates through the `files` array and sends one `file_operation` (op: "write") per file to the plugin side for local disk writing. The `directories` array is used by the plugin side to pre-create directories.

---

## VI. UI Components Required on Plugin Side

| Component | Corresponding Message | Key Functionality |
|-----------|-----------------------|-------------------|
| Progress Timeline | status_update × 5~8 messages | Reuse existing timeline component |
| Skeleton Configuration Card | approval_request #1 | Scheduling mode single select + extra modules multi-select + custom file input field. **New `item_groups` group rendering** (different groups have different selection modes) |
| File Tree Preview | phase_complete artifact[0] | Tree directory display of generated file structure |
| File Writing | file_operation × N | Write files to local disk one by one (needs new "Skeleton generation in progress, writing files..." progress indicator) |

### item_groups Group Rendering Specification

When approval_request contains the `item_groups` field, the plugin should render by group:

```
┌─ scheduler_mode: "Scheduling Mode" (radio) ─┐
│  ◉ Timer tick (Recommended)                 │
│  ○ asyncio                                  │
│  ○ _thread                                  │
└─────────────────────────────────────────────┘

┌─ extra_modules: "Extra Modules" (checkbox) ┐
│  ☑ Logging System                          │
│  ☑ Deployment Tools                        │
│  ☐ Performance Timer                       │
└────────────────────────────────────────────┘
```

Each group's `multi_select` in `item_groups` determines whether it renders as radio or checkbox.

---

## VII. Independent Test Scenarios

### Plugin-Side Testing (No Server)

1. Manually send `approval_request` #1 (skeleton configuration card) → Verify:
   - `item_groups` group rendering is correct (scheduler_mode single select, extra_modules multi-select)
   - Switching scheduling modes, recommendation markers follow
   - Checking/unchecking extra modules
   - Clicking "Add Custom File" opens input field
2. Manually send `phase_complete` (with file_tree artifact) → Verify file tree renders correctly
3. Manually send `file_operation` sequence (5 writes) → Verify files are written to local disk one by one + progress indicator

### Skill-Side Testing (No Plugin)

1. **full mode + timer:**
   - mock_plugin sends start_phase (mode=full, manifest=temperature/humidity project, phase: select-hw)
   - Auto-reply to approval_request #1 with `{"action": "confirm", "selected_ids": ["mode_timer", "module_logger", "module_flash"]}`
   - Verify init_scaffold.py stdout JSON contains 15+ files
   - Verify all file_operation paths are correct, encoding is utf-8
2. **full mode + async:**
   - mock_plugin reply selects mode_async
   - Verify main.py uses uasyncio framework
   - Verify scheduler/timer_sched.py is not injected
3. **incremental mode:**
   - mock_plugin sends start_phase (mode=incremental, new_devices=[{name: "DHT22", driver: {source: "upypi"}}])
   - Verify only 1 file is generated: `firmware/drivers/dht22_driver/__init__.py`
4. **init_scaffold.py template rendering:**
   - Given a standard manifest, run `python init_scaffold.py --mode timer --manifest - < test_manifest.json`
   - Verify board.py's I2C pins in stdout JSON match manifest.pinout
   - Verify main.py's GPIO initialization matches manifest.pinout
   - Verify README.md's BOM table matches manifest.bom
