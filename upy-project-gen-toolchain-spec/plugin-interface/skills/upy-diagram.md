# upy-diagram Interface Definition

> Status: ✅ Finalized
>
> Phase 7b — Software architecture diagram generation. Read all .py files + manifest in firmware/, LLM analyzes code structure/execution flow/data flow, generates diagram.json, script renders 3 types of Mermaid diagrams (architecture diagram + flowchart + data flow diagram) × 4 formats (.md + .svg + .png + .html).

---

## I. Skill Overview

| Item | Content |
|------|---------|
| Phase | diagram |
| Upstream Skill | upy-generate (manual/auto trigger) |
| Downstream Skill | None |
| One-line Responsibility | Read code → LLM analyzes layered architecture/execution flow/data flow → generate diagram.json → validate → render 3 Mermaid diagram types × 4 formats = 13 files |

**Same pattern as upy-wiring, differences:**
- Has user interaction (complexity selection)
- Outputs 3 diagram types (architecture graph TB + flowchart sequenceDiagram + data flow graph LR)
- Outputs 13 files (vs wiring's 6)

---

## II. Plugin Input → Skill (P→S)

```json
{
  "type": "start_phase",
  "phase": "diagram",
  "session_id": "uuid-xxx",
  "payload": {
    "manifest": { /* complete project-manifest.json */ },
    "complexity": null
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `manifest` | object | Yes | Complete manifest |
| `complexity` | string? | No | `"simple"` / `"medium"` / `"full"`. null → triggers approval_request to ask. If present (from user preference) → skip |

**Complexity level constraints:**

| Parameter | simple | medium (default) | full |
|-----------|:------:|:----------------:|:----:|
| Total modules | ≤6 | ≤10 | ≤16 |
| Modules per layer | ≤2 | ≤4 | ≤6 |
| Cross-layer dependency edges | ≤6 | ≤12 | ≤20 |
| Total flow steps | ≤5 | ≤8 | ≤14 |
| Total data_flow edges | ≤2 | ≤4 | ≤8 |

---

## III. Skill Output → Plugin (S→P)

### Message Sequence

```
(Optional) Step 0: Complexity selection
  → approval_request: complexity selection card (diagram_complexity)
  → (skipped if complexity is preset)

Step 1-2: Read source code
  → file_operation(read) × N (all .py in firmware/)
  → status_update "✓ Read N files"

Step 3: Analyze + generate diagram.json
  → status_update "Analyzing code structure..."
  → status_update "Architecture: 5 layers / N modules / K dependencies"
  → status_update "Flow: M steps (boot→init→scan→create→assembly→run)"
  → status_update "Data flow: P channels"
  → file_operation(write) → docs/diagram.json
  → status_update "✓ diagram.json generated"

Step 4: Validate
  → script_run(validate_json.py --schema .upy/schemas/diagram.schema.json --json docs/diagram.json)
  → (validation fails → LLM fixes → file_operation(write) → re-run script_run, loop until pass)

Step 5: Render
  → status_update "Rendering architecture diagram (mermaid.ink)..."
  → status_update "Rendering flowchart..."
  → status_update "Rendering data flow diagram..."
  → script_run(render_diagram_local.py --input docs/diagram.json --output docs/ --format all)
  → status_update "✓ Generated 13 files"

Step 6: Update manifest
  → file_operation(read) → project-manifest.json
  → (server modifies diagrams field)
  → file_operation(write) → project-manifest.json

Output
  → phase_complete(file_list + diagnostics table)
```

### approval_request — Complexity Selection (diagram_complexity)

Conditional trigger: shown when `complexity` is null.

```
┌──────────────────────────────────────────┐
│  Architecture Diagram Complexity          │
│                                          │
│  Select the level of detail:              │
│                                          │
│  ○ Simple                                │
│    Highly condensed, ≤6 modules, ≤5 steps │
│    Suitable for quick overview            │
│                                          │
│  ● Medium (Recommended)                   │
│    Balanced information, ≤10 modules, ≤8 steps │
│    Suitable for daily development & communication │
│                                          │
│  ○ Detailed                              │
│    Full expansion, ≤16 modules, ≤14 steps │
│    Suitable for complex projects or documentation │
│                                          │
│  [Confirm]                               │
└──────────────────────────────────────────┘
```

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "diagram_complexity",
    "header": "Architecture Diagram Complexity",
    "question": "Select the level of detail for the architecture diagram",
    "summary": {},
    "items": [
      {
        "id": "simple",
        "name": "Simple",
        "subtitle": "≤6 modules, ≤5 steps, highly condensed",
        "meta": "Suitable for quick overview",
        "selected": false
      },
      {
        "id": "medium",
        "name": "Medium (Recommended)",
        "subtitle": "≤10 modules, ≤8 steps, balanced information",
        "meta": "Suitable for daily development",
        "selected": true
      },
      {
        "id": "full",
        "name": "Detailed",
        "subtitle": "≤16 modules, ≤14 steps, full expansion",
        "meta": "Suitable for documentation",
        "selected": false
      }
    ],
    "allow_add": false,
    "allow_remove": false,
    "multi_select": false,
    "actions": [
      { "label": "Confirm", "value": "confirm", "primary": true }
    ]
  }
}
```

### status_update List

| step_id | level | message | Trigger |
|---------|-------|---------|---------|
| read_src | info | Reading firmware/ source code... | Step 2 start |
| read_done | success | ✓ Read N files | Step 2 complete |
| analyze | info | Analyzing code structure... | Step 3 start |
| arch_summary | success | Architecture: 5 layers / N modules / K cross-layer dependencies | Architecture analysis complete |
| flow_summary | success | Flow: M steps (boot→init→...→run) | Flow analysis complete |
| dataflow_summary | success | Data flow: P channels | Data flow analysis complete |
| gen_json | info | Generating diagram.json... | Writing JSON |
| gen_done | success | ✓ diagram.json generated | Step 3 complete |
| validate | info | Validating diagram.json... | Step 4 |
| validate_pass | success | ✓ Validation passed | Passed |
| validate_fail | warn | ✗ N errors → Fixing (round M) | Failed |
| render_arch | info | Rendering architecture diagram (mermaid.ink)... | Architecture diagram rendering |
| render_flow | info | Rendering flowchart... | Flowchart rendering |
| render_data | info | Rendering data flow diagram... | Data flow diagram rendering |
| render_done | success | ✓ Generated 13 files | Step 5 complete |
| update_manifest | info | Updating manifest... | Step 6 |
| done | success | ✓ Architecture diagram generation complete | All complete |

### script_run — render_diagram_local.py (Step 5)

```json
{
  "type": "script_run",
  "payload": {
    "script_id": "diagram_render",
    "interpreter": "python",
    "script": ".upy/scripts/render_diagram_local.py",
    "args": ["--input", "docs/diagram.json", "--output", "docs/", "--format", "all"],
    "cwd": "{project_dir}",
    "timeout_ms": 90000
  }
}
```

**timeout 90s** — Renders 3 diagram types × 3 formats (.md generated locally, .svg/.png each requires 1 HTTP request to mermaid.ink API), total 6 network requests.

### phase_complete

```json
{
  "type": "phase_complete",
  "payload": {
    "phase": "diagram",
    "result": "success",
    "summary": "Architecture diagram generation complete: 5 layers / 8 modules / 12 dependencies / 6 flow steps / 3 data flows (medium complexity)",
    "next_phase": null,
    "artifacts": [
      {
        "type": "file_list",
        "title": "Generated Files (13)",
        "files": [
          { "path": "docs/diagram.json", "size": 8192, "status": "new", "description": "Architecture intermediate JSON" },
          { "path": "docs/architecture.md", "size": 3072, "status": "new", "description": "Layered architecture diagram Mermaid" },
          { "path": "docs/architecture.svg", "size": 45056, "status": "new", "description": "Layered architecture diagram SVG" },
          { "path": "docs/architecture.png", "size": 87040, "status": "new", "description": "Layered architecture diagram PNG" },
          { "path": "docs/architecture.html", "size": 10240, "status": "new", "description": "Layered architecture diagram HTML" },
          { "path": "docs/flowchart.md", "size": 2048, "status": "new", "description": "Execution flowchart Mermaid" },
          { "path": "docs/flowchart.svg", "size": 32768, "status": "new", "description": "Execution flowchart SVG" },
          { "path": "docs/flowchart.png", "size": 56320, "status": "new", "description": "Execution flowchart PNG" },
          { "path": "docs/flowchart.html", "size": 9216, "status": "new", "description": "Execution flowchart HTML" },
          { "path": "docs/data_flow.md", "size": 1536, "status": "new", "description": "Data flow diagram Mermaid" },
          { "path": "docs/data_flow.svg", "size": 24576, "status": "new", "description": "Data flow diagram SVG" },
          { "path": "docs/data_flow.png", "size": 39936, "status": "new", "description": "Data flow diagram PNG" },
          { "path": "docs/data_flow.html", "size": 7168, "status": "new", "description": "Data flow diagram HTML" }
        ]
      },
      {
        "type": "table",
        "title": "Diagnostics Information",
        "headers": ["Metric", "Value"],
        "rows": [
          ["Total modules", "8"],
          ["Total dependency edges", "12"],
          ["Maximum depth", "4 layers"],
          ["Circular dependencies", "None"],
          ["Orphan modules", "None"],
          ["Direct import machine", "main.py (entry layer only, normal)"]
        ]
      }
    ],
    "warnings": [],
    "errors": []
  }
}
```

---

## IV. SKILL.md Modification Points

Total 6 changes:

| # | Location | Current Behavior | Change To | Reason |
|---|----------|-----------------|-----------|--------|
| 1 | Pre-checks | `python --version` + `python -c "import jsonschema"` | Remove | Server doesn't perceive environment |
| 2 | Step 0 Complexity | `AskUserQuestion(...)` | `approval_request` complexity selection card. Skip if `complexity` is preset in start_phase | Plugin-side interaction |
| 3 | Step 1 Read schema | LLM Read `diagram.schema.json` | Schema pre-placed by scaffold in `.upy/schemas/`. Server LLM has built-in schema knowledge to generate JSON directly | Spec not in project directory |
| 4 | Step 2 Read source | LLM directly Read firmware/**/*.py + manifest | `file_operation(read)` reads files one by one. Manifest already in start_phase.payload | Server reads files via plugin |
| 5 | Step 4 Validate | `validate_json.py --schema <spec path> --json ...` | `script_run(validate_json.py --schema .upy/schemas/diagram.schema.json --json docs/diagram.json)` | Schema+script pre-placed by scaffold in `.upy/` |
| 6 | Step 5+6 Render | `render_diagram_local.py --input ... --output ...` | `script_run(render_diagram_local.py --input docs/diagram.json --output docs/ --format all)`. Script pre-placed by scaffold | Requires network (mermaid.ink) + write files → plugin executes |
| 7 | Step 7 Update manifest | `python -c "..."` inline script | `file_operation(read)` → server modifies diagrams field → `file_operation(write)` | Unified file operations |

---

## V. Validation Scripts

### validate_json.py

Shared with wiring, **no changes needed**. Already a generic JSON Schema validator.

### render_diagram_local.py

**Path:** `G:\MicroPython_Skills\upy-diagram\scripts\render_diagram_local.py`

| Change | Content |
|--------|---------|
| Added `--json-summary` | Outputs one line of JSON when rendering completes: `{"status":"ok","files":[{"path":"docs/architecture.md","size":3072},...],"errors":[]}` |

**No other changes needed.** Defensive reading already verified in wiring script.

### Impact on upy-scaffold

| Source File | Target Location | Description |
|-------------|-----------------|-------------|
| `diagram.schema.json` | `{project}/.upy/schemas/diagram.schema.json` | Validates diagram.json |
| `render_diagram_local.py` | `{project}/.upy/scripts/render_diagram_local.py` | Renders 3 diagram types |

**Shares validate_json.py with wiring, no duplicate copy needed.**

---

## VI. Plugin-Side UI Components

| Component | Corresponding Message | Description |
|-----------|----------------------|-------------|
| Complexity selection card | approval_request `diagram_complexity` | Simple/Medium/Detailed three-choice, only shown when complexity is empty |
| Progress timeline | status_update × ~12 | Read→Analyze (Architecture/Flow/Data Flow)→Generate→Validate→Render×3 |
| Architecture diagram preview | file_list → click architecture.html | WebView embedded preview |
| Flowchart preview | file_list → click flowchart.html | WebView embedded preview |
| Data flow diagram preview | file_list → click data_flow.html | WebView embedded preview |
| Diagnostics information panel | phase_complete table artifact | Total modules/dependencies/depth/circular dependencies/orphan modules |
| [Generate Architecture Diagram] button | Triggers start_phase | Enabled after generate completes |

---

## VII. Independent Test Scenarios

### Plugin-Side Testing (No Server)

1. Manually send approval_request `diagram_complexity` → Verify three-choice + confirm
2. Manually send `status_update` sequence → Verify three-phase (Architecture/Flow/Data Flow) progress
3. Manually send `phase_complete` (file_list 13 files + diagnostics table) → Verify file list + diagnostics information panel
4. Click architecture.html / flowchart.html / data_flow.html → Verify WebView preview

### Skill-Side Testing (No Plugin)

1. Prepare complete firmware/ directory + manifest, mock file_operation(read) to return file contents
2. Verify complexity="simple": total modules ≤6, flow steps ≤5
3. Verify complexity="medium": total modules ≤10, flow steps ≤8
4. Verify complexity="full": total modules ≤16, flow steps ≤14
5. LLM generates diagram.json → validate_json.py validation passes
6. render_diagram_local.py → Confirm 13 output files + --json-summary output is correct
7. Check all sent message JSON conforms to 02-protocol.md Schema
