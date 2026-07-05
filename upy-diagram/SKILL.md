---
name: upy-diagram
description: Step 7 — Software Architecture Diagram Generation. Reads firmware/ code and project-manifest.json, LLM generates intermediate JSON, script renders Mermaid text architecture diagram (.md code block, CLI natively readable) + SVG + PNG + HTML (double-click browser viewable). Trigger: after upy-generate completes.
---

# Software Architecture Diagram Generation Skill

## Role Definition

Given `project-manifest.json` (phase: generate) and all `.py` files under `firmware/`, the LLM understands `diagram.schema.json`, then analyzes code structure, execution flow, and data flow, populates the intermediate JSON, which is then validated by a script and used to generate Mermaid text diagrams (Markdown code blocks) + SVG + PNG + HTML. **Mermaid .md + SVG + PNG + HTML are all required outputs; the script defaults to --format all. The LLM is responsible for reading the code and filling in the JSON; the script only performs validation and rendering.**

---

## Pre-checks

```bash
python --version
python -c "import jsonschema; print('jsonschema OK')"
```

If missing, prompt to install: `pip install jsonschema`

SVG rendering requires network access (mermaid.ink API, zero local dependencies), see Step 6 for details.

---

## Execution Steps

### Step 0: Select Complexity Level

**Before performing any analysis, first ask the user for the desired architecture diagram complexity.** The complexity level controls the upper limits of all constraint parameters below, affecting the diagram's conciseness.

```python
AskUserQuestion(
  questions=[{
    "question": "What complexity level do you need for the architecture diagram?",
    "header": "Architecture Diagram Complexity",
    "options": [
      {"label": "Simple", "description": "Highly concise, only core modules/dependencies/steps, suitable for quick browsing"},
      {"label": "Medium (Recommended)", "description": "Balances information and readability, suitable for daily development and communication"},
      {"label": "Detailed", "description": "Full expansion, all modules/steps/data flows retained, suitable for complex projects or archival documentation"}
    ],
    "multiSelect": False
  }]
)
```

**Parameter Reference Table (LLM uses the selected level as the upper constraint limit):**

| Parameter | Simple | Medium (Default) | Detailed |
|-----------|:------:|:----------------:|:--------:|
| `architecture` total modules | ≤6 | ≤10 | ≤16 |
| Max modules per layer | ≤2 | ≤4 | ≤6 |
| `cross_layer_deps` total edges | ≤6 | ≤12 | ≤20 |
| `cross_layer_deps[].label` | ≤4 chars | ≤6 chars | ≤10 chars |
| `role` | ≤8 chars | ≤10 chars | ≤14 chars |
| `flow[]` total steps | ≤5 | ≤8 | ≤14 |
| `flow[].action` | ≤4 chars | ≤6 chars | ≤8 chars |
| `flow[].detail` | ≤8 chars | ≤12 chars | ≤16 chars |
| `data_flow[]` total edges | ≤2 | ≤4 | ≤8 |
| `data_flow[].data` | ≤6 chars | ≤8 chars | ≤12 chars |

**After selection, the LLM strictly adheres to the values in the corresponding column as upper limits.** All constraint descriptions in Step 3 are based on the selected level. Default: Medium.

### Step 1: LLM Reads Schema → Understands Structure

Read the intermediate JSON schema:

```
G:/MicroPython_Skills/upy-project-gen-toolchain-spec/diagram.schema.json
```

Understand the 4 required fields: `meta`, `architecture`, `flow`, `data_flow`, and the optional fields `task_registry`, `diagnostics`.

### Step 2: LLM Reads Source Code → Analyzes Structure

Read all of the following files (each file must be read thoroughly):

```
{project_dir}/project-manifest.json
{project_dir}/firmware/main.py            ← Entry point: DI assembly chain + flow steps
{project_dir}/firmware/conf.py            ← Configuration constants
{project_dir}/firmware/board.py           ← Board-level pin constant mapping
{project_dir}/firmware/boot.py            ← Boot code
{project_dir}/firmware/lib/               ← Base libraries (logger/scheduler/time_helper, etc.)
{project_dir}/firmware/drivers/           ← Driver factories + mocks (one package per driver)
{project_dir}/firmware/tasks/             ← Business task files
```

### Step 3: LLM Analyzes and Fills diagram.json

#### 3A: `meta` — Metadata

Extract from manifest: `project`, `mode`, `mcu`, `source_phase`.
`generated_at` should be the current UTC time (ISO 8601).

#### 3B: `architecture.layers[]` — Layered Architecture

**Layer Definitions (bottom-up):**

| Layer ID | Label | Modules Included |
|----------|-------|-----------------|
| `board` | Board Layer | `board.py` — pin constant mapping |
| `lib` | Library Layer | All .py files under `lib/`: logger, scheduler, time_helper, etc. |
| `driver` | Driver Layer | `drivers/<name>_driver/__init__.py` — factories for each component |
| `task` | Task Layer | `tasks/*.py` — business task functions |
| `entry` | Entry Layer | `main.py` — DI assembly entry point |
| `test` | Test Layer | `test/pc/*.py` — PC-side tests; `test/device/*.py` — device-side tests |

Optional additional layer: `host` (when code exists under `host/`).

**Each module object:**
- `name`: Python import path, e.g., `tasks.sensor_task`
- `path`: Relative file path, e.g., `firmware/tasks/sensor_task.py`
- `role`: Brief Chinese description of the module's responsibility, upper limit based on the level selected in Step 0 (extracted from the first line of the docstring; if no docstring, the LLM writes one. Node box width is limited; overly long text causes node expansion and layout crowding)
- `provides`: List of exported function/class names (extracted from `def` / `class`, excluding private symbols prefixed with `_`)
- `depends_on`: List of dependent module names (extracted from `import` / `from X import`, excluding `machine` and standard library)
- `depends_on_machine`: Whether it directly `import machine` (true only for main.py)
- `has_mock`: Whether `drivers/<name>_driver/mock.py` exists
- `is_generated`: Whether the file was generated by upy-generate (`@Generated : upy-generate` marker)
- `is_template`: Whether the file comes from a scaffold template
- `source`: Source enum (`scaffold_template` / `llm_generated` / `upypi_download` / `github_download` / `cold_driver` / `user_custom`)

**LLM decides autonomously:**
- Whether to split a single task file into multiple modules (if a task file has multiple independent functions); however, the total number of modules, per-layer limits, cross-layer dependency edge count, and label length are all bounded by the complexity level selected in Step 0. If exceeded, merge functionally similar modules.
- `cross_layer_deps[].label`: Edge label length is bounded by the level selected in Step 0 (e.g., "import", "inject", "log"; overly long edge labels make connections crowded and hard to read).
- `cross_layer_deps[].style`: solid (direct dependency) / dashed (DI injection dependency) / dotted (test dependency).
- **16:9 ratio preview: For each module/edge/step written, ensure that elements in all directions do not exceed 70% capacity within a 16:9 canvas; otherwise, merge or delete.**

#### 3C: `flow[]` — Execution Flow

Extract the sequence of execution steps from `main.py`. Each step:

- `seq`: Sequence number starting from 1
- `phase`: Step phase
  - `boot` → Startup delay, WDT setup
  - `init` → I2C/SPI bus initialization, log initialization
  - `scan` → I2C device scanning (`scan_xxx_i2c()`)
  - `create` → Driver instance creation (`create_xxx()`)
  - `assembly` → DI assembly (driver injection into tasks)
  - `run` → Scheduler start / event loop run
  - `shutdown` → Cleanup (if exists)
- `action`: Short Chinese title, upper limit based on the level selected in Step 0 (e.g., "Initialize I2C"; participant width in sequence diagrams is limited, overly long text will be truncated).
- `detail`: Specific parameters, upper limit based on the level selected in Step 0 (I2C address, Pin number, frequency, etc.; will be displayed on a new line below the action).
- `source_line`: Line number in main.py
- `depends_on_step`: Sequence number of the prerequisite step (e.g., create depends on scan success)
- `on_error`: Failure strategy (`fatal` terminate / `skip_device` skip the device and continue / `retry` retry / `degrade` degrade operation)
- `is_conditional` + `branches`: Conditional branches (e.g., scan success → create, failure → skip)

**LLM decides autonomously:** Step granularity (a single init action can be split into multiple steps or merged), **total number of steps is bounded by the level selected in Step 0** (merge similar operations, do not make every function call a separate step); details of conditional branches.

#### 3D: `data_flow[]` — Data Flow

Analyze data transfer between task functions:

- `from` / `to`: Data source and destination (module name or function name)
- `data`: Description of the transferred data, upper limit based on the level selected in Step 0 (e.g., "Temperature and humidity reading", "Alarm status")
- `channel`: Transmission channel
  - `shared_dict` → Transfer via shared dict (e.g., `data["temp"] = ...`)
  - `function_return` → Transfer via function return value
  - `global_var` → Global variable
  - `queue` → Transfer via Queue (async mode)
  - `callback_param` → Callback function parameter
- `rate`: Refresh frequency (e.g., `1Hz`, `on_change`, `100ms`)

**LLM decides autonomously:** Granularity of data_flow (can merge similar flows or list them individually), **total number of edges is bounded by the level selected in Step 0** (only retain core data flows; omit overly detailed or unidirectional flows without branches).

#### 3E: `task_registry[]` — Task Registration List

Extract scheduler registration information from main.py (timer mode from `sc.register()`, async mode from `asyncio.create_task()`):

- `name`: Task name
- `callback`: Callback function name
- `interval_ms`: Execution interval
- `mode`: `periodic` / `once` / `on_event`

#### 3F: `diagnostics` — Diagnostic Information

Filled by the LLM after analyzing the code:

- `total_modules`: Total number of modules in architecture
- `total_dependencies`: Total number of dependency edges in depends_on
- `max_depth`: Maximum depth of the dependency graph (counting down from entry)
- `circular_deps`: Detected circular dependencies (should be an empty array)
- `orphan_modules`: Modules not depended upon by any other module (e.g., pure utility functions)
- `machine_direct_access`: Modules that directly import machine (should warn for any module other than main.py)

### Step 4: Validate diagram.json

```bash
python G:/MicroPython_Skills/upy-project-gen-toolchain-spec/scripts/validate_json.py \
  --schema G:/MicroPython_Skills/upy-project-gen-toolchain-spec/diagram.schema.json \
  --json {project_dir}/docs/diagram.json
```

Validation fails → modify diagram.json → re-validate until it passes.

### Step 5: Generate Mermaid .md + SVG + PNG + HTML Files (Combined Required Output)

**This is the primary output of this skill.** The script generates 3 Markdown files (containing Mermaid code blocks) + 3 SVGs + 3 PNGs + 3 HTMLs from diagram.json. CLI directly readable, natively rendered in VS Code / GitHub, HTML double-clickable in a browser.

```bash
python G:/MicroPython_Skills/upy-diagram/scripts/render_diagram_local.py \
  --input {project_dir}/docs/diagram.json \
  --output {project_dir}/docs/
```

The script defaults to `--format all`, outputting .md, .svg, .png, and .html simultaneously:

| File | Mermaid Diagram Type | Content |
|------|----------------------|---------|
| `docs/architecture.md` + `.svg` + `.png` + `.html` | `graph TB` | Layered architecture diagram: subgraphs grouped by layer, nodes=modules, edges=dependencies |
| `docs/flowchart.md` + `.svg` + `.png` + `.html` | `sequenceDiagram` | Execution flow diagram: MCU participants, grouped by phase, conditional branches + error handling |
| `docs/data_flow.md` + `.svg` + `.png` + `.html` | `graph LR` | Data flow diagram: data channels between modules, different arrow types for different channels |

SVG is rendered via the mermaid.ink API (zero local dependencies, requires network), vector format for crisp clarity.

### Step 6: SVG Rendering (Required, Already Included in Step 5's --format all)

The script defaults to using the mermaid.ink API for SVG rendering (zero local dependencies, requires network):

```bash
# SVG only (skip .md rewriting):
python G:/MicroPython_Skills/upy-diagram/scripts/render_diagram_local.py \
  --input {project_dir}/docs/diagram.json \
  --output {project_dir}/docs/ \
  --format svg
```

Principle: Mermaid code Base64 encoded → GET `https://mermaid.ink/img/{base64}?type=svg` → Save SVG.

HTML uses the Mermaid.js CDN to render directly in the browser, independent of mermaid.ink.

**Alternative — PNG (also supported by mermaid.ink):**

```bash
python G:/MicroPython_Skills/upy-diagram/scripts/render_diagram_local.py \
  --input {project_dir}/docs/diagram.json \
  --output {project_dir}/docs/ \
  --format png
```

**Alternative — mermaid-cli (local rendering, requires Node.js):**

```bash
npm install -g @mermaid-js/mermaid-cli
python G:/MicroPython_Skills/upy-diagram/scripts/render_diagram_local.py \
  --input {project_dir}/docs/diagram.json \
  --output {project_dir}/docs/ \
  --format png-local
```

### Step 7: Update Manifest

```bash
cd {project_dir} && python -c "
import json, os
from datetime import datetime, timezone
path = 'project-manifest.json'
with open(path, 'r', encoding='utf-8') as f:
    m = json.load(f)
m['diagrams'] = m.get('diagrams', {})
m['diagrams']['json'] = 'docs/diagram.json'
m['diagrams']['architecture'] = 'docs/architecture.md'
m['diagrams']['architecture_svg'] = 'docs/architecture.svg'
m['diagrams']['architecture_png'] = 'docs/architecture.png'
m['diagrams']['architecture_html'] = 'docs/architecture.html'
m['diagrams']['flowchart'] = 'docs/flowchart.md'
m['diagrams']['flowchart_svg'] = 'docs/flowchart.svg'
m['diagrams']['flowchart_png'] = 'docs/flowchart.png'
m['diagrams']['flowchart_html'] = 'docs/flowchart.html'
m['diagrams']['data_flow'] = 'docs/data_flow.md'
m['diagrams']['data_flow_svg'] = 'docs/data_flow.svg'
m['diagrams']['data_flow_png'] = 'docs/data_flow.png'
m['diagrams']['data_flow_html'] = 'docs/data_flow.html'
m['diagrams']['generated_at'] = datetime.now(timezone.utc).isoformat()
with open(path, 'w', encoding='utf-8') as f:
    json.dump(m, f, ensure_ascii=False, indent=2)
print('[OK] manifest diagrams updated')
"
```

---

## Relationship with Other Skills

- ← `upy-generate`: Input complete firmware code + manifest
- Parallel with `upy-wiring`: Can be generated simultaneously
- → VS Code Plugin WebView: Display Mermaid diagrams (Markdown preview) or SVG

---

## Hard Constraints

- **LLM generates JSON, script only validates + renders**: Consistent with the `upy-generate` pattern
- **Schema is the sole contract**: diagram.json must pass `validate_json.py` validation
- **Must read all firmware/*.py files thoroughly**: Do not skip any file; architecture analysis is based on real code
- **Layer IDs must use enum values**: `board`, `lib`, `driver`, `task`, `entry`, `host`, `test`
- **Flow phases must use enum values**: `boot`, `init`, `scan`, `create`, `assembly`, `run`, `shutdown`
- **Data flow channels must use enum values**: `function_return`, `shared_dict`, `global_var`, `queue`, `callback_param`
- **module.source must use enum values**: `scaffold_template`, `llm_generated`, `upypi_download`, `github_download`, `cold_driver`, `user_custom`
- **provides/depends_on extracted from real imports and defs**: Do not fabricate symbols
- **diagnostics filled truthfully**: Includes orphan_modules and machine_direct_access warnings
- **Rendering script reads defensively**: Missing fields will not cause crashes, but will output warnings to stderr
- **SVG + PNG + HTML are required outputs**: The script defaults to `--format all`, generating .md, .svg, .png, and .html simultaneously; only `--format md` can skip images and HTML
- **Readability constraints (upper limits for each level are in the Step 0 parameter reference table; default is Medium. Ensure PNG is clearly readable at a 16:9 aspect ratio)**:

  | Field | Simple | Medium (Default) | Detailed | Description |
  |-------|:------:|:----------------:|:--------:|-------------|
  | `architecture` total modules | ≤6 | ≤10 | ≤16 | Merge functionally similar modules |
  | Max modules per layer | ≤2 | ≤4 | ≤6 | Upper limit per layer |
  | `role` | ≤8 chars | ≤10 chars | ≤14 chars | Node box line 2; overly long causes node expansion |
  | `cross_layer_deps[].label` | ≤4 chars | ≤6 chars | ≤10 chars | Edge label embedded in arrow middle; overly long makes connections crowded |
  | `cross_layer_deps[]` total edges | ≤6 | ≤12 | ≤20 | Cross-layer edges are the main cause of crowding; only retain core dependencies |
  | `flow[].action` | ≤4 chars | ≤6 chars | ≤8 chars | Sequence diagram vertical space limited by 16:9 |
  | `flow[].detail` | ≤8 chars | ≤12 chars | ≤16 chars | Wraps below action; overly long encroaches on vertical space |
  | `flow[]` total steps | ≤5 | ≤8 | ≤14 | Merge similar steps; do not translate code line by line |
  | `data_flow[].data` | ≤6 chars | ≤8 chars | ≤12 chars | Edge label; overly long causes arrow compression |
  | `data_flow[]` total edges | ≤2 | ≤4 | ≤8 | Only retain core data flows |
  | 16:9 ratio | ≤70% | ≤70% | ≤70% | LLM previews Mermaid rendering; merge if exceeded |
