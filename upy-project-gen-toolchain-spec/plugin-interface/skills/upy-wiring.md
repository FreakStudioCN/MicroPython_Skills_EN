# upy-wiring Interface Definition

> Status: ✅ Finalized
>
> Phase 7a — Wiring diagram generation. Read all firmware/ .py source files to extract actual pins/buses/addresses, cross-validate with manifest, LLM generates wiring.json, script renders Mermaid .md + SVG + PNG + HTML.

---

## I. Skill Overview

| Item | Content |
|------|---------|
| Phase | wiring |
| Upstream Skill | upy-scaffold or upy-generate (manual/auto trigger) |
| Downstream Skill | upy-diagram (parallel, can be generated simultaneously) |
| One-line Responsibility | firmware source as authoritative data source → LLM extracts hardware connection facts → generates wiring.json → validates → renders Mermaid wiring diagram + SVG + PNG + HTML + pin cross-reference table |

**Core Constraints:**
- firmware > manifest > LLM inference (data priority)
- LLM generates JSON, script only validates + renders
- Schema is the sole contract: wiring.json must pass validate_json.py validation
- SVG + PNG + HTML are mandatory outputs

---

## II. Plugin Input → Skill (P→S)

```json
{
  "type": "start_phase",
  "phase": "wiring",
  "session_id": "uuid-xxx",
  "payload": {
    "manifest": { /* complete project-manifest.json */ },
    "complexity": "full"
  }
}
```

| Field | Type | Required | Source | Description |
|-------|------|----------|--------|-------------|
| `manifest` | object | Yes | upy-generate output | Complete manifest, including mcu/devices/pinout/bom |
| `complexity` | string | No | Plugin setting | `"simple"` — only .md; `"full"` — .md + .svg + .png + .html + _pins.md. Default `"full"` |

**Note:** The manifest is passed in start_phase, so the LLM does not need to file_operation(read) it again. However, firmware/ source files are **not** in the payload and must be read one by one via file_operation(read).

---

## III. Skill Output → Plugin (S→P)

### Message Sequence

```
Step 1-3: Read source + extract hardware facts
  → status_update "Reading firmware/ source files..."
  → file_operation(read) × N (all .py files under firmware/)
  → status_update "✓ Read N files, extracted X pins, Y buses, Z warnings"

Step 4: Generate wiring.json
  → status_update "Generating wiring intermediate JSON..."
  → file_operation(write) → docs/wiring.json
  → status_update "✓ wiring.json generated"

Step 5: Validate
  → script_run(validate_json.py --schema .upy/schemas/wiring.schema.json --json docs/wiring.json)
  → script_result
  → (Validation fails → LLM fixes wiring.json → file_operation(write) → re-validate, loop until pass)

Step 6: Render
  → status_update "Rendering wiring diagram (mermaid.ink)..."
  → script_run(render_wiring_local.py --input docs/wiring.json --output docs/ --format all)
  → status_update "✓ Generated wiring.md + .svg + .png + .html + _pins.md"

Step 7: Update manifest
  → file_operation(read) → project-manifest.json
  → (Server modifies wiring field)
  → file_operation(write) → project-manifest.json

Output
  → phase_complete(file_list)
```

### status_update List

| step_id | level | message | Trigger |
|---------|-------|---------|---------|
| read_src | info | Reading firmware/ source files... | Step 2 start |
| read_file | info | Reading: firmware/tasks/sensor_task.py (N/M) | Per file |
| read_done | success | ✓ Read N files, extracted X pins, Y buses, Z warnings | Step 2+3 complete |
| gen_json | info | Generating wiring intermediate JSON... | Step 4 |
| gen_done | success | ✓ wiring.json generated | Step 4 complete |
| validate | info | Validating wiring.json... | Step 5 |
| validate_pass | success | ✓ wiring.json validation passed | Validation passed |
| validate_fail | warn | ✗ wiring.json: N errors → fixing (round M) | Validation failed, entering fix loop |
| render | info | Rendering wiring diagram (mermaid.ink API)... | Step 6 |
| render_svg | info | Rendering SVG... | render_wiring_local sub-step |
| render_png | info | Rendering PNG... | render_wiring_local sub-step |
| render_done | success | ✓ Generated 5 files | Step 6 complete |
| update_manifest | info | Updating manifest... | Step 7 |
| done | success | ✓ Wiring diagram generation complete | All complete |

### script_run — validate_json.py (Step 5)

```json
{
  "type": "script_run",
  "payload": {
    "script_id": "wiring_validate",
    "interpreter": "python",
    "script": ".upy/scripts/validate_json.py",
    "args": ["--schema", ".upy/schemas/wiring.schema.json", "--json", "docs/wiring.json"],
    "cwd": "{project_dir}",
    "timeout_ms": 15000
  }
}
```

**Note:** `validate_json.py` and `wiring.schema.json` are both copied to the project `.upy/` directory by upy-scaffold and are accessible locally by the plugin.

### script_run — render_wiring_local.py (Step 6)

```json
{
  "type": "script_run",
  "payload": {
    "script_id": "wiring_render",
    "interpreter": "python",
    "script": ".upy/scripts/render_wiring_local.py",
    "args": ["--input", "docs/wiring.json", "--output", "docs/", "--format", "all"],
    "cwd": "{project_dir}",
    "timeout_ms": 60000
  }
}
```

**Note:** The render script requires network access (mermaid.ink API) to generate SVG/PNG and must be executed on the plugin side. Timeout of 60s covers network latency.

### phase_complete

```json
{
  "type": "phase_complete",
  "payload": {
    "phase": "wiring",
    "result": "success",
    "summary": "Wiring diagram generation complete: 2 buses (I2C×1, GPIO×3), 12 pins, 3 warnings",
    "next_phase": "diagram",
    "artifacts": [
      {
        "type": "file_list",
        "title": "Generated Files",
        "files": [
          { "path": "docs/wiring.json", "size": 4096, "status": "new", "description": "Wiring intermediate JSON" },
          { "path": "docs/wiring.md", "size": 2048, "status": "new", "description": "Mermaid wiring diagram" },
          { "path": "docs/wiring.svg", "size": 32768, "status": "new", "description": "SVG vector wiring diagram" },
          { "path": "docs/wiring.png", "size": 65536, "status": "new", "description": "PNG wiring diagram" },
          { "path": "docs/wiring.html", "size": 8192, "status": "new", "description": "Self-contained HTML (browser view)" },
          { "path": "docs/wiring_pins.md", "size": 1024, "status": "new", "description": "Pin cross-reference table" }
        ]
      }
    ],
    "warnings": [
      "I2C pull-up resistors not declared: Please confirm SDA/SCL are pulled up to 3.3V with 4.7kΩ resistors"
    ],
    "errors": []
  }
}
```

**warnings examples (LLM auto-generates by rules):**

| Condition | level | Example msg |
|-----------|-------|-------------|
| I2C address conflict | `danger` | "SHT30 and BMP280 both at 0x76 — address conflict" |
| No pull-up resistor declaration | `warning` | "Verify I2C pull-up resistors on SDA/SCL (4.7kΩ to 3.3V)" |
| 5V device on 3.3V | `danger` | "LCD1602: 5V device on 3.3V pin — level shifter needed" |
| Buzzer without current-limiting resistor | `info` | "Add 220Ω resistor in series with buzzer" |
| firmware vs manifest mismatch | `danger` | "SHT30: firmware uses 0x44, manifest says 0x45" |

---

## IV. SKILL.md Modification Points

Total 6 changes:

| No. | Location | Current Behavior | Change To | Reason |
|-----|----------|-----------------|-----------|--------|
| 1 | Pre-check | `python --version` + `python -c "import jsonschema"` | Remove. Dependencies guaranteed by plugin environment + scaffold preset files | Server does not perceive runtime environment |
| 2 | Step 1 Read schema | LLM Read `wiring.schema.json` (spec directory) | Schema preset by scaffold to `{project}/.upy/schemas/wiring.schema.json`. Server-side LLM has built-in schema knowledge and generates JSON directly, no file read needed | Spec files not in project directory, server cannot access |
| 3 | Step 2-3 Read source files | LLM directly Read firmware/**/*.py + manifest | `file_operation(read)` reads all .py files under firmware/ one by one. Manifest already in start_phase.payload, no need to re-read | Server reads files through plugin |
| 4 | Step 4 Write wiring.json | LLM writes local file | `file_operation(write)` → docs/wiring.json | Unified file operations |
| 5 | Step 5 Validate | `python validate_json.py --schema <spec path> --json ...` | `script_run(validate_json.py --schema .upy/schemas/wiring.schema.json --json docs/wiring.json)`. Script preset by scaffold to `.upy/scripts/` | Schema and script must be in project directory, executable locally by plugin |
| 6 | Step 6+7 Render | `python render_wiring_local.py --input ... --output ...` | `script_run(render_wiring_local.py --input docs/wiring.json --output docs/ --format all)`. Script preset by scaffold | Rendering requires network (mermaid.ink) + write local files → plugin execution |
| 7 | Step 8 Update manifest | `python -c "..."` inline script | `file_operation(read)` manifest → server modifies wiring field → `file_operation(write)` | Unified file operations |

---

## V. Validation Script Changes

### validate_json.py

**Path:** `G:\MicroPython_Skills\upy-project-gen-toolchain-spec\scripts\validate_json.py`

**No changes needed.** It is already a generic JSON Schema validator, takes `--schema` + `--json`, outputs `[OK]` / `[FAIL]` + error list, exit code 0=pass / 1=fail / 2=error. script_run uses it directly.

### render_wiring_local.py

**Path:** `G:\MicroPython_Skills\upy-wiring\scripts\render_wiring_local.py`

| Change | Content |
|--------|---------|
| Added `--json-summary` | Outputs a single JSON summary line to stdout upon render completion: `{"status":"ok","files":[{"path":"docs/wiring.md","size":2048},...],"errors":[]}`. For server to confirm output file list and sizes |

**No other changes needed.** The render script already uses defensive reading (`safe_get`) and will not crash due to missing wiring.json fields.

### Impact on upy-scaffold

| Source File | Target Location | Purpose |
|-------------|-----------------|---------|
| `G:/.../upy-project-gen-toolchain-spec/scripts/validate_json.py` | `{project}/.upy/scripts/validate_json.py` | Shared validation for wiring + diagram |
| `G:/.../upy-project-gen-toolchain-spec/wiring.schema.json` | `{project}/.upy/schemas/wiring.schema.json` | Validate wiring.json |
| `G:/.../upy-wiring/scripts/render_wiring_local.py` | `{project}/.upy/scripts/render_wiring_local.py` | Render wiring diagram |

---

## VI. Plugin-side UI Components

| Component | Corresponding Message | Description |
|-----------|----------------------|-------------|
| Progress timeline | status_update × ~10 | Read→Extract→Generate→Validate→Render→Output |
| Wiring diagram preview | Click wiring.html in file_list | WebView embedded preview (Tab switching: Wiring Diagram/Source Code/Pin Table) |
| [Generate Wiring Diagram] button | Triggers start_phase | Enabled after scaffold/generate completes |
| [Regenerate] button | Replace button | Can regenerate after initial generation |

### Wiring Diagram Preview Description

After plugin receives phase_complete(file_list), it renders as a file list. User clicks `wiring.html` → WebView preview:

```
┌──────────────────────────────────────────┐
│ [Wiring] [Mermaid Source] [Pin Table]     │
├──────────────────────────────────────────┤
│                                          │
│   ┌─────────────────────┐                │
│   │    ESP32 DevKit     │                │
│   │  ┌──────┬──────┐    │    ┌────────┐  │
│   │  │ GPIO21│ SDA  │────┼────│ SHT30  │  │
│   │  │ GPIO22│ SCL  │────┼────│ 0x44   │  │
│   │  └──────┴──────┘    │    └────────┘  │
│   └─────────────────────┘                │
│                                          │
└──────────────────────────────────────────┘
```

---

## VII. Independent Test Scenarios

### Plugin-side Testing (No Server)

1. Manually send `status_update` sequence → Verify read→generate→validate→render→output timeline
2. Manually send `file_operation(read)` request → Return simulated firmware/ file content
3. Manually send `file_operation(write)` → docs/wiring.json → Confirm file written
4. Manually send `phase_complete` (file_list) → Verify file list rendering + wiring.html preview entry

### Skill-side Testing (No Plugin)

1. Prepare complete firmware/ directory + manifest, mock file_operation(read) to return file content
2. LLM generates wiring.json → Run validate_json.py validation passes
3. Run render_wiring_local.py → Confirm 5 output files generated
4. Verify cross-validation rules: Construct case where firmware and manifest mismatch → Confirm firmware takes precedence + warning generated
5. Check all sent message JSON conforms to 02-protocol.md Schema
