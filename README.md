# MicroPython Skills for GraftSense

GraftSense MicroPython Skill collection, containing **25 dedicated Skills**, divided into two major systems:

**A. One-Sentence Hardware Building — AI Embedded Code Generation Pipeline (10 skills)**: Starting from natural language requirements, automatically complete the full closed loop of hardware selection, code generation, PC simulation, deployment, and error fixing.

**B. Driver Development Standardization (15 skills)**: Based on the complete writing specification (22 chapters, 2200+ lines) from the [GraftSense-Drivers-MicroPython](https://github.com/FreakStudioCN/GraftSense-Drivers-MicroPython) repository, covering driver standardization, test file generation, README generation, performance optimization, memory optimization, packaging, and device deployment.

---

## 📚 Repository Documentation

This repository contains the following core documents. It is recommended to read them as needed:

| Document | Description | Use Case |
|---|---|---|
| [README.md](README.md) | This document, Skill installation and usage guide | Quick start, installing Skills |
| [upy_driver_dev_spec_summary.md](upy_driver_dev_spec_summary.md) | **Complete GraftSense Driver Development Specification** (22 chapters, 2200+ lines), covering file structure, class design, docstring, type annotations, parameter validation, exception handling, ISR specifications, and all other rules | In-depth understanding of specification details, reference when manually writing drivers |
| [MicroPython_Performance_Optimization_Guide.md](MicroPython_Performance_Optimization_Guide.md) | **MicroPython Performance Optimization Guide**, detailing `@viper`, `@native`, `const()`, pre-allocated buffers, `memoryview`, pointer access, and other optimization techniques, with measured data and code examples | Optimizing driver execution speed, understanding the optimization principles of `upy-opt-driver` |
| [MicroPython_Memory_Footprint_Minimization_Guide.md](MicroPython_Memory_Footprint_Minimization_Guide.md) | **MicroPython Memory Footprint Minimization Guide**, detailing frozen modules, `.mpy` files, `const()`, buffer reuse, `gc` control, `__slots__`, generators, and other memory optimization techniques, with REPL test code | Reducing driver RAM usage, understanding the optimization principles of `upy-slim-driver` |

**Reading Suggestions**:
- Beginners: First read this README to install the Skills, then directly use commands like `/upy-norm-driver` to standardize code
- Advanced: Read `upy_driver_dev_spec_summary.md` to understand specification details, manually write compliant drivers
- Optimization: Read the performance/memory optimization guides to understand the optimization principles of `upy-opt-driver` and `upy-slim-driver`

---

## Table of Contents

- [Installation Methods](#installation-methods)
- [One-Sentence Hardware Building — AI Embedded Code Generation Pipeline](#one-sentence-hardware-building--ai-embedded-code-generation-pipeline)
- [Driver Development Standardization Skill List](#skill-list)
  - [upy-norm-driver](#upy-norm-driver--driver-file-standardization)
  - [upy-norm-main](#upy-norm-main--test-file-standardization)
  - [upy-gen-main](#upy-gen-main--generate-test-file-from-scratch)
  - [upy-gen-readme](#upy-gen-readme--generate-readme-from-scratch)
  - [upy-gen-pkg](#upy-gen-pkg--generate-packagejson-from-scratch)
  - [upy-norm-pkg](#upy-norm-pkg--full-driver-package-standardization)
  - [upy-deploy-test](#upy-deploy-test--device-deployment-and-verification)
  - [upy-opt-driver](#upy-opt-driver--driver-performance-optimization)
  - [upy-slim-driver](#upy-slim-driver--driver-memory-optimization)
  - [upy-pack-driver](#upy-pack-driver--package-into-standard-directory-structure)
  - [upy-pkg-guide](#upy-pkg-guide--device-driver-usage-query)
  - [fetch-doc](#fetch-doc--url-content-fetching)
  - [upy-project](#upy-project--micropython-end-to-end-project-generation)
  - [mpremote-device-interaction](#mpremote-device-interaction--device-connection-and-status-query)
  - [mpremote-file-transfer](#mpremote-file-transfer--device-file-transfer)
  - [mpremote-live-session](#mpremote-live-session--persistent-connection-and-output-monitoring)
- [How It Works](#how-it-works)
- [Specification Documents](#specification-documents)
- [Version History](#version-history)
- [License](#license)

---

## Installation Methods

> **Network restricted?** It is recommended to use the "Local Installation" method below. No network required; simply clone the repository and copy.

### Method 1: Local Installation (Recommended, No Network Required)

**Use Case**: Network restricted, offline environment, or if you have already cloned this repository locally.

**Step 1**: Clone this repository (or download the ZIP and extract it)

```bash
git clone https://github.com/FreakStudioCN/MicroPython_Skills.git
```

**Step 2**: Copy the skill directories to Claude Code's skills directory

The skills directory is fixed at `~/.claude/skills/`. It expands as follows depending on the operating system:

| System | Actual Path |
|---|---|
| Windows | `C:\Users\<username>\.claude\skills\` |
| macOS | `/Users/<username>/.claude/skills/` |
| Linux | `/home/<username>/.claude/skills/` |

**macOS / Linux**:
```bash
# Install a single skill
cp -r MicroPython_Skills/upy-norm-driver ~/.claude/skills/

# Install all skills (execute inside the cloned directory)
cd MicroPython_Skills
for skill in upy-analyze upy-select-hw upy-scaffold upy-generate upy-simulate \
             upy-deploy upy-deploy-test upy-autofix upy-wiring upy-diagram upy-gen-driver \
             upy-norm-driver upy-norm-main upy-gen-main upy-gen-readme \
             upy-gen-pkg upy-norm-pkg upy-opt-driver upy-slim-driver upy-pack-driver \
             upy-pkg-guide fetch-doc upy-project review \
             mpremote-device-interaction mpremote-file-transfer mpremote-live-session; do
  cp -r $skill ~/.claude/skills/
done
```

**Windows (PowerShell)**:
```powershell
# Install a single skill
Copy-Item -Recurse MicroPython_Skills\upy-norm-driver $env:USERPROFILE\.claude\skills\

# Install all skills (execute inside the cloned directory)
cd MicroPython_Skills
$skills = @("upy-analyze","upy-select-hw","upy-scaffold","upy-generate","upy-simulate",
            "upy-deploy","upy-deploy-test","upy-autofix","upy-wiring","upy-diagram","upy-gen-driver",
            "upy-norm-driver","upy-norm-main","upy-gen-main","upy-gen-readme",
            "upy-gen-pkg","upy-norm-pkg","upy-opt-driver","upy-slim-driver","upy-pack-driver",
            "upy-pkg-guide","fetch-doc","upy-project","review",
            "mpremote-device-interaction","mpremote-file-transfer","mpremote-live-session")
foreach ($skill in $skills) {
  Copy-Item -Recurse $skill $env:USERPROFILE\.claude\skills\
}
```

**Step 3**: Restart Claude Code. The skills will take effect.

---

### Method 2: Online Installation (Requires Network + Node.js)

```bash
npx skillfish add FreakStudioCN/MicroPython_Skills upy-norm-driver
npx skillfish add FreakStudioCN/MicroPython_Skills upy-norm-main
npx skillfish add FreakStudioCN/MicroPython_Skills upy-gen-main
npx skillfish add FreakStudioCN/MicroPython_Skills upy-gen-readme
npx skillfish add FreakStudioCN/MicroPython_Skills upy-gen-pkg
npx skillfish add FreakStudioCN/MicroPython_Skills upy-norm-pkg
npx skillfish add FreakStudioCN/MicroPython_Skills upy-opt-driver
npx skillfish add FreakStudioCN/MicroPython_Skills upy-slim-driver
npx skillfish add FreakStudioCN/MicroPython_Skills upy-pack-driver
```

Or install all at once:

```bash
for skill in upy-norm-driver upy-norm-main upy-gen-main upy-gen-readme \
             upy-gen-pkg upy-norm-pkg upy-opt-driver upy-slim-driver upy-pack-driver; do
  npx skillfish add FreakStudioCN/MicroPython_Skills $skill
done
```

---

## One-Sentence Hardware Building — AI Embedded Code Generation Pipeline

The user only needs to describe the requirement in natural language ("Make a temperature and humidity monitor, buzzer alarm when threshold is exceeded"), and the system automatically completes the entire process from selection, code generation, PC simulation, flashing, to error fixing.

### Pipeline Overview

```
User says one sentence
    ↓
Phase 1: upy-analyze    → Requirement analysis + Driver search
Phase 2: upy-select-hw  → MCU selection + Pin assignment + BOM
Phase 3: upy-scaffold   → Project skeleton generation
Phase 4: upy-generate   → Business code generation
Phase 4.5: upy-simulate → Full PC-side simulation (no hardware required)
Phase 5: upy-deploy     → One-click flash and run
Phase 6: upy-autofix    → Error classification decision + Delegated fix
Phase 7: upy-wiring     → Wiring diagram generation
       upy-diagram      → Architecture diagram + Flowchart
Exception path: upy-gen-driver → Uncommon hardware driver generation
```

### Skill List

| # | Skill | Phase | Status | Description |
|---|-------|------|------|------|
| 1 | `upy-analyze` | Phase 1 | Implemented | Natural language → Device list + Driver API reference |
| 2 | `upy-select-hw` | Phase 2 | Implemented | MCU selection + Firmware verification + Pin assignment + BOM |
| 3 | `upy-scaffold` | Phase 3 | Implemented | Generate complete firmware/ skeleton (Timer/asyncio/Thread) |
| 4 | `upy-generate` | Phase 4 | Implemented | Driver download + DI architecture business code + Mock + unittest |
| 5 | `upy-simulate` | Phase 4.5 | Implemented | Full PC-side CLI+rich simulation (data generator + multiple scenarios) |
| 6 | `upy-deploy` | Phase 5 | Implemented | mpremote upload + flash + persistent session + initial PASS/FAIL judgment |
| 7 | `upy-autofix` | Phase 6 | Implemented | Orchestration coordination layer: triage.py collects → LLM hierarchical decision → delegates to upstream skill |
| 8 | `upy-wiring` | Phase 7 | Implemented | Wiring diagram generation (Mermaid .md + SVG + PNG + HTML) |
| 9 | `upy-diagram` | Phase 7 | Implemented | Architecture diagram + Flowchart + Data flow diagram (Mermaid .md + SVG + PNG + HTML) |
| 10 | `upy-gen-driver` | Exception path | Implemented | PDF/Arduino → Debug version driver → Hardware verification loop → Standardized MPY driver |

**Supporting Tools:**
- `upy-project-gen-toolchain-spec` — Overall architecture document + manifest/schema definitions
- `upy-pkg-guide` — Device driver usage query (called by upy-analyze)
- `fetch-doc` — URL content fetching (called by upy-pkg-guide)

### Skill Descriptions

#### `/upy-analyze` — Requirement Analysis + Driver Search

Takes user natural language description, LLM decomposes intent → multi-keyword parallel search on upypi + awesome-micropython → extracts driver API reference → outputs `project-manifest.json` (phase: analyze).

#### `/upy-select-hw` — MCU Selection + Pin Assignment

Reads manifest → recommends MCU based on scenario/power consumption/network requirements → I2C address conflict detection → generates pin assignment table (with electrical type enumeration + physical pin numbers) → outputs BOM bill of materials.

#### `/upy-scaffold` — Project Skeleton Generation

Reads manifest → AskUserQuestion to select scheduling mode (Timer/asyncio/_thread) and optional modules → calls `init_scaffold.py` to generate complete `firmware/` skeleton (board.py, conf.py, boot.py, main.py, drivers/*, tasks/*, lib/*, tools/*).

#### `/upy-generate` — Business Code Generation

Reads firmware/ skeleton + driver API reference → downloads driver → upy-norm-driver standardizes → generates DI architecture task code + conf.py + main.py + Mock layer + unittest → black + flake8 + pylint validation.

#### `/upy-simulate` — Full PC-side Simulation

LLM reads all firmware/ code → autonomously designs: scheduling scheme + data generator `gen_xxx(tick)` + visualization (CLI+rich preferred) + multi-scenario coverage → generates `test/pc/sim_main.py` → flake8 + pylint validation → runs. **No real hardware needed to verify business logic.**

#### `/upy-deploy` — One-click Flash and Run

mpremote uploads firmware/ → verifies file integrity → soft reset + reconnection wait → persistent session collects output → device-side log capture → local rule-based initial PASS/FAIL judgment.

#### `/upy-autofix` — Orchestration Coordination Layer

Automatically enters after deploy fails. `triage.py` collects structured data (error parsing + I2C hardware detection + git management) → LLM reads JSON + raw logs → hierarchical decision (P0~P3) → delegates to upstream skill for fix (generate/select-hw/analyze) → optional PC verification → redeploy. Maximum 3 attempts.

#### `/upy-wiring` — Wiring Diagram Generation

Reads all .py source code in firmware/ to extract actual pins/addresses/buses → cross-validates with manifest → LLM generates intermediate JSON → script renders Mermaid wiring diagram .md + SVG + PNG + self-contained HTML (double-click browser to view) + pin cross-reference table.

#### `/upy-diagram` — Architecture Diagram + Flowchart

Scans firmware/ code structure + manifest → LLM generates intermediate JSON → script renders Mermaid architecture diagram + flowchart + data flow diagram, each outputting .md + SVG + PNG + self-contained HTML (Tabs to switch between diagram/source code, dark mode adaptive). Supports simple/medium/detailed three levels of complexity.

#### `/upy-gen-driver` — Driver Code Generation (Exception Path)

Triggered when no driver is found on upypi or GitHub. Extracts information from PDF datasheet or Arduino code → LLM generates debug version single-file driver (with full self-check logic) → `mpremote resume run` hardware verification loop (up to 10 rounds) → remove debug → `upy-norm-driver` standardizes. Can be called by `upy-analyze`, `upy-autofix`, or directly by the user.

---

### `/upy-norm-driver` — Driver File Standardization

**Purpose**: Rewrite a functional but non-standard MicroPython driver `.py` file (not `main.py`) according to the GraftSense specification, outputting the complete standardized file.

**Input**: Existing driver `.py` file path

**Output**: Standardized complete `.py` file + rewrite description table

**Covered Rules**: P0 mandatory 38 items, P2 optional 7 items, including:

| Category | Main Rewrite Items |
|---|---|
| File Structure | File header 7-line comment, 4 module global variables, 6 section markers, section content specification |
| Class Design | Class structure layout, `__slots__` optimization, avoid multiple inheritance, explicit dependency injection, constant specification |
| docstring | Class-level bilingual (Chinese/English) (including Attributes/Methods/Notes), method-level bilingual (Chinese/English), ISR-safe annotation, side-effect annotation |
| Type Annotations | `__init__` parameter annotations, public method return value annotations, callback uses `callable` |
| Parameter Validation | Three modes: `isinstance`/`hasattr`/value range, `__init__` two-step validation |
| Exception Handling | Exception type standardization, `OSError` wrapping and re-raising (preserve `from e`), retry mechanism |
| ISR Specification | Prohibit memory allocation/blocking I/O/raising exceptions, `micropython.schedule`, concurrency protection |
| Function Design | Naming conventions, return value design, `debug` log switch |

**Core Constraint**: Do not modify external API names, method signature semantics, business logic, or hardware communication timing.

**Usage Example**:
```
/upy-norm-driver sensors/bh1750_driver/code/bh_1750.py
```

---

### `/upy-norm-main` — Test File Standardization

**Purpose**: Rewrite an existing `main.py` test file according to the specification without changing the test logic.

**Input**: Existing `main.py` file path

**Output**: Standardized complete `main.py`

**P0 Mandatory Items (10 items)**:

| # | Rewrite Item |
|---|---|
| 1 | File header 7-line comment |
| 2 | 6 section marker comments (correct order) |
| 3 | Initialization configuration section must have `time.sleep(3)` |
| 4 | Initialization configuration section must have `print("FreakStudio: ...")` |
| 5 | Global variable section prohibits instantiation, move to initialization configuration section |
| 6 | `while` loop only allowed in the main program section |
| 7 | `raise`/`print` strings all in English |
| 8 | Main program section wrapped with `try/except KeyboardInterrupt/OSError/Exception/finally` |
| 9 | `finally` calls `close()`/`deinit()`, `del` hardware objects, prints exit prompt |
| 10 | Inline comments changed to Chinese |

**P1 Try to Change**: High-frequency function comments with default calls (for REPL manual invocation), three types of test scenario coverage check.

**Usage Example**:
```
/upy-norm-main sensors/bh1750_driver/main.py
```

---

### `/upy-gen-main` — Generate Test File from Scratch

**Purpose**: Given a driver `.py` file, analyze all its public APIs, and generate a complete `main.py` from scratch that conforms to the specification.

**Input**: Driver `.py` file path

**Output**: Complete `main.py` + API coverage description

**Full Coverage Principle**:

Classify all APIs by chip type functional dimensions:

| Chip Type | Coverage Dimensions |
|---|---|
| Sensor Class | Basic status query, core data acquisition, parameter configuration, mode switching, calibration/compensation |
| Motor Driver Class | Hardware initialization, motion control, status reading, reset/sleep |
| Communication Module Class | Network/protocol configuration, data send/receive, status query, power control |
| Memory Chip Class | Data read/write, address configuration, erase/reset |
| GPIO/Bus Expander Class | Pin configuration, level read/write, interrupt configuration |

Cover three types of test scenarios: normal parameters, boundary parameters (hardware limit values), abnormal parameters (verify exceptions are correctly raised).

API handling method: Low-frequency APIs are executed automatically; high-frequency/mode-switching APIs are called via comments (for REPL manual triggering).

**Usage Example**:
```
/upy-gen-main sensors/bh1750_driver/code/bh_1750.py
```

---

### `/upy-gen-readme` — Generate README from Scratch

**Purpose**: Given a driver `.py` file, analyze its functionality and APIs, and generate a complete `README.md` from scratch.

**Input**: Driver `.py` file path (optional: existing README as reference)

**Output**: Complete `README.md`

**13 Mandatory Sections**:

| # | Section | Content |
|---|---|---|
| 1 | Title | `# [Chip Name] MicroPython Driver` |
| 2 | Table of Contents | Anchor links for all sections |
| 3 | Introduction | Driver purpose, functionality, applicable scenarios |
| 4 | Key Features | List of feature highlights |
| 5 | Hardware Requirements | Recommended hardware + pin description table |
| 6 | Software Environment | Firmware version, dependency libraries |
| 7 | File Structure | File tree (`├──` format) |
| 8 | File Description | Explain purpose of each file |
| 9 | Quick Start | Step-by-step instructions + minimal runnable code example |
| 10 | Notes | Operating conditions, limitations, compatibility |
| 11 | Version History | Table: Version/Date/Author/Change Description |
| 12 | Contact | Email + GitHub |
| 13 | License | MIT License |

**Usage Example**:
```
/upy-gen-readme sensors/bh1750_driver/code/bh_1750.py
```

---

### `/upy-norm-pkg` — Full Driver Package Standardization

**Purpose**: For an already verified driver file, perform the complete standardization workflow on the entire driver package directory. This is an Orchestrator Skill.

**Input**: Driver package directory path

**Output**: Fully standardized driver package (all driver files + main.py + README.md + package.json + standard directory structure)

**Execution Flow (6 Steps)**:

| Step | Operation |
|---|---|
| 0 | Scan directory, classify driver files and `main.py`; for multiple driver files, list them and ask user to confirm the scope |
| 1 | Execute `/upy-norm-driver` for each driver file sequentially, pause for confirmation after each file |
| 2 | Execute `/upy-norm-main` (if `main.py` exists) or `/upy-gen-main` (if no `main.py`) |
| 3 | Execute `/upy-gen-readme` |
| 4 | Execute `/upy-gen-pkg` |
| 5 | Execute `/upy-pack-driver` |
| 6 | Execute `/upy-deploy-test` (upload to device and verify after user confirmation) |

**Key Rule**: After each step, display `[Step X/6 — skill_name: file_name completed]`, pause and wait for user confirmation before continuing.

**Usage Example**:
```
/upy-norm-pkg sensors/bh1750_driver/
```

---

### `/upy-deploy-test` — Device Deployment and Verification

**Purpose**: After `upy-norm-pkg` completes, upload the standardized driver files and `main.py` to the MicroPython device, run them, and verify the output.

**Input**: Standardized `code/` directory path + user-confirmed COM port

**Output**: Upload progress + verification report (success/failure + error analysis)

**Execution Flow (6 Steps)**:

| Step | Operation |
|---|---|
| 0 | Ask and confirm the COM port (can execute `mpremote connect list` for assistance) |
| 1 | Scan files to upload (`.py` files + sub-package directories) |
| 2 | Upload files one by one (`mpremote connect <COM> resume fs cp`) |
| 3 | Verify device file integrity (`fs ls`) |
| 4 | Run `main.py` (`mpremote resume run main.py`) |
| 5 | Analyze output, output verification report |

**Failure Diagnosis**: `ImportError` → missing file; `OSError -110` → I2C wiring; `RuntimeError: WiFi` → check if credential placeholders have been replaced.

**mpremote Reference**: `/mpremote-device-interaction`, `/mpremote-file-transfer`, `/mpremote-live-session`, [Official Documentation](https://docs.micropython.org/en/latest/reference/mpremote.html)

**Usage Example**:
```
/upy-deploy-test bh1750_driver/code/
```

---

### `/upy-opt-driver` — Performance Optimization

**Purpose**: Rewrite any MicroPython `.py` file (driver file, `main.py`, or other files) according to the GraftSense performance optimization guide, focusing on **execution speed** improvement.

**Input**: Driver `.py` file path or directory path (supports batch optimization of multiple files)

**Output**: Optimized complete `.py` file + optimization description table

**Optimization Priority**:

| Priority | Item | Typical Speedup |
|---|---|---|
| P0 | Pre-allocated buffers | Eliminate GC jitter |
| P0 | `memoryview` slicing | Zero copy (> 32 bytes) |
| P0 | Cache object references | 5–20% (loops > 100 iterations) |
| P0 | `const()` constants | Zero overhead |
| P1 | Manual GC control | Controllable latency |
| P1 | `@native` decorator | ~2x |
| P1 | `@viper` decorator | ~58x (integer operations) |
| P1 | Integer instead of float | ~57% (no FPU chip) |
| P2 | `viper ptr8/ptr16/ptr32` | ~23x (large loop traversal) |
| P2 | SIO register direct write | ~48% (RP2040 specific) |
| P2 | `array` instead of `list` | Contiguous memory |

**Core Constraints**: `@viper` rewrites must annotate integer overflow risk; `@native` must annotate limitations (no generators/keyword arguments); SIO registers must annotate "RP2040 specific".

**Usage Example**:
```
/upy-opt-driver sensors/bh1750_driver/code/bh_1750.py
/upy-opt-driver sensors/bh1750_driver/code/
```

---

### `/upy-slim-driver` — Memory Optimization

**Purpose**: Rewrite any MicroPython `.py` file (driver file, `main.py`, or other files) according to the GraftSense memory minimization guide, focusing on **RAM usage** reduction.

**Input**: Driver `.py` file path or directory path (supports batch optimization of multiple files)

**Output**: Optimized complete `.py` file + optimization description table

**Optimization Priority**:

| Priority | Item | Typical Savings |
|---|---|---|
| P0 | Pre-allocated buffers | Eliminate peak heap allocation |
| P0 | Private `_CONST` | ~40 bytes/constant |
| P0 | Avoid string concatenation in loops | Eliminate temporary objects |
| P0 | `bytes`/`bytearray` instead of `list` | ~90% (register tables) |
| P1 | `gc.collect()` pre-positioning | Reduce randomness |
| P1 | `gc.disable()`/`gc.enable()` | Prevent GC interruption mid-operation |
| P1 | `struct.pack_into()` | Eliminate temporary bytes |
| P2 | `__slots__` | 50–200 bytes/instance |
| P2 | Generator instead of list | Peak RAM O(N)→O(1) |

**Core Constraints**: `_CONST` rewrites only apply to module-internal constants; `gc.disable()` intervals must be short and bounded, and must not contain blocking I/O; overlaps with `upy-opt-driver`'s P0#1 (pre-allocated buffers), do not execute redundantly.

**Usage Example**:
```
/upy-slim-driver sensors/bh1750_driver/code/bh_1750.py
/upy-slim-driver sensors/bh1750_driver/code/
```

---

### `/upy-pack-driver` — Package into Standard Directory Structure

**Purpose**: After other Skills have completed execution, organize the driver file, `main.py`, `README.md`, and `package.json` into a standard driver package directory structure, and generate a `LICENSE` file.

**Input**: Driver `.py` file path (`main.py`, `README.md`, `package.json` must already exist in the same directory)

**Output**: Standard directory structure:
```
<chip>_driver/
├── code/
│   ├── <chip>.py
│   └── main.py
├── package.json
├── README.md
└── LICENSE
```

**Core Constraint**: Does not generate any content, only responsible for organizing files; if files are missing, it will prompt the user to run the corresponding Skill first.

**Usage Example**:
```
/upy-pack-driver bmp280.py
```

---

### `/upy-pkg-guide` — Device Driver Usage Query

**Purpose**: Given a device name, automatically fetch all files of the corresponding driver package from upypi, perform comprehensive analysis, and output usage highlights.

**Input**: Device/chip name (e.g., BMP280, DS18B20, MPR121)

**Output**: Package information, installation command, initialization example, core API table, notes

**Execution Flow**: curl search upypi → fetch package.json → parallel download driver.py + main.py + README.md → comprehensive output

**Usage Example**:
```
/upy-pkg-guide BMP280
/upy-pkg-guide DS18B20
```

---

### `/fetch-doc` — URL Content Fetching

**Purpose**: Given any URL, automatically fetch the content and extract key information. Supports GitHub files, upypi package pages, and regular web pages.

**Input**: URL (GitHub blob links are automatically converted to raw URLs)

**Output**: Key information extracted based on content type (README summary, driver API table, package.json fields, etc.)

**Dependency**: Requires Python + requests library (`pip install requests`)

**Usage Example**:
```
/fetch-doc https://github.com/FreakStudioCN/GraftSense-Drivers-MicroPython/blob/main/sensors/bmp280_driver/README.md
```

---

### `/review` — MicroPython Code Review

**Purpose**: Based on MicroPython maintainer historical review patterns (~19.5K classified review comments), perform AI-assisted review of MPY driver code.

**Input**: MicroPython code changes (branch, commit, diff, PR)

**Output**: Semantically searched matching historical review patterns + review context suggestions

**Core Capabilities**:
- Semantic search across ~19.5K classified review comments to find relevant historical review patterns
- Supports MCP server (`review_diff`, `search_reviews` tools, etc.) and CLI methods
- MCP server keeps the embedding model warm, eliminating the 2-3s cold start for each query

**Usage Example**:
```
/review Review the diff of the current branch against main
/review Check sensors/bmp280_driver/code/bmp280.py
```

---

### `/upy-project` — MicroPython End-to-End Project Generation

**Purpose**: The user describes the project requirements, and the system automatically completes the entire process from requirement clarification, device selection, code generation, to device debugging.

**Input**: Project description (main controller model, sensor list, functional requirements, serial port)

**Output**: Complete project code (`xx_task.py` + `main.py`) + mpremote automatic debugging

**Execution Flow (5 Phases)**:

| Phase | Operation |
|---|---|
| Pre-check | Verify mpremote availability |
| Phase 0 | Parse GitHub links in user input (call fetch-doc skill) |
| Phase 1 | List all missing information at once, no multiple rounds of questioning |
| Phase 2 | Select devices from upypi, call upy-pkg-guide to get API usage |
| Phase 3 | Generate task files + main.py (unified scheduling) |
| Phase 4 | mpremote automatic debugging, up to 3 times, parse output and fix |

**Code Structure**:
```
/lib/<driver>.py       ← Downloaded from upypi
<function>_task.py     ← Single function module (contains init() + run())
main.py                ← Unified scheduling
```

**Usage Example**:
```
/upy-project Use ESP32 and BMP280 to make a temperature monitor, print every 5 seconds, COM3 port
```

---

### `/upy-gen-pkg` — Generate package.json from Scratch

**Purpose**: Given a driver directory or driver file, analyze the structure and dependencies, and generate a compliant `package.json` from scratch.

**Input**: Driver directory path or driver `.py` file path

**Output**: Complete `package.json` + three installation method commands

**Dependency Handling Three-Step Priority**:

```
1. MicroPython built-in modules (machine, time, sys, etc.) → Do not write to deps
2. micropython-lib standard library → Use mip standard format
3. Other third-party dependencies → Query https://upypi.net/api/search?q={dependency_name}
   If result exists → Write deps using upypi URL
   If no result → Use github: placeholder format, annotate ⚠️ requires manual confirmation
```

**Usage Example**:
```
/upy-gen-pkg sensors/bh1750_driver/
```

---

### `/mpremote-device-interaction` — Device Connection and Status Query

**Purpose**: Connect to a MicroPython device via mpremote, execute code, query device status (memory, firmware version, file list, etc.).

**Platform Support**: Windows (COMn), macOS (/dev/tty.usbmodem*), Linux (mpy-dev or /dev/serial/by-id/)

**Core Principle**: Connecting to a running device must use `resume`, otherwise a soft reset will interrupt the program.

**Covered Scenarios**:

| Scenario | Command Example |
|---|---|
| List available devices | `mpremote connect list` |
| Windows connection | `mpremote c3 resume` / `mpremote connect COM3 resume` |
| macOS connection | `mpremote connect /dev/tty.usbmodem1101 resume` |
| Linux connection | `mpremote connect $(mpy-dev tty my-board) resume` |
| Query firmware version | `mpremote <device> resume exec "import sys; print(sys.version)"` |
| Query free memory | `mpremote <device> resume exec "import gc; gc.collect(); print(gc.mem_free())"` |
| Soft reset | `mpremote <device> soft-reset` |

**Usage Example**:
```
/mpremote-device-interaction  Connect to COM3, check firmware version and free memory
```

---

### `/mpremote-file-transfer` — Device File Transfer

**Purpose**: Use mpremote to copy files between the local machine and the device, manage the device file system (ls, mkdir, rm, tree).

**Platform Support**: Windows, macOS, Linux. Device path syntax for each platform is detailed within the Skill.

**Key Rule**: File operations must include `resume`, otherwise the device will be soft-reset before each operation.

**Covered Scenarios**:

| Scenario | Command Example |
|---|---|
| Upload file | `mpremote <device> resume fs cp main.py :main.py` |
| Download file | `mpremote <device> resume fs cp :main.py .` |
| Recursively sync directory | `mpremote <device> resume fs cp -r utils/ :utils/` |
| Restart after updating driver | `mpremote <device> resume fs cp driver.py :driver.py + soft-reset repl` |
| List files | `mpremote <device> resume fs ls :` |
| View storage space | `mpremote <device> resume exec "import os; print(os.statvfs('/'))"` |

**Usage Example**:
```
/mpremote-file-transfer  Sync the local utils/ directory to the device, then restart and monitor
```

---

### `/mpremote-live-session` — Persistent Connection and Output Monitoring

**Purpose**: Establish a persistent connection to the device, continuously send commands and capture output. Suitable for devices running asyncio, stress testing, long-term monitoring.

**Platform Support**: Linux/macOS uses PTY solution; Windows uses subprocess pipe alternative (has limitations, see Skill documentation).

**Core Principle**: Repeatedly calling `mpremote resume exec` will send Ctrl+C to asyncio devices, killing the event loop; a persistent session must be used instead.

**When to Use**:

| Scenario | Recommended Solution |
|---|---|
| Single quick query | `mpremote <device> resume exec "..."` |
| Multi-command sequence / monitoring output | This Skill (persistent session) |
| Device running asyncio/aiorepl | This Skill (mandatory) |
| File copying | mpremote-file-transfer |

**Usage Example**:
```
/mpremote-live-session  Establish a persistent connection to /dev/tty.usbmodem1101, query memory every second and log to a file
```

---

## How It Works

Each Skill is a `SKILL.md` file containing:

- **Role Definition**: Tells the AI what role to play
- **Core Constraints**: Clearly states what cannot be modified
- **Rewrite Priority Table**: P0 mandatory / P2 optional, each item corresponds to a specific chapter in the specification document
- **Key Specification Summary**: Embeds the most important code templates to avoid looking up the full specification document each time

### Trigger Flow

```
User inputs /upy-norm-driver xxx.py
    ↓
Claude loads the specification summary and priority table from SKILL.md
    ↓
Reads the target file, analyzes structure (communication interface type, classes, methods, ISR callbacks, etc.)
    ↓
Rewrites item by item according to P0→P2 priority (does not change API or business logic)
    ↓
Outputs complete standardized file + rewrite description table
```

### Why Split into Multiple Skills

The specification document has 22 chapters and 2200+ lines. Embedding the entire specification in a single Skill would lead to excessive context length and reduced rewrite quality. By splitting according to "rewrite target" and "optimization goal", each Skill only embeds the specification summary for its corresponding chapters, keeping the context manageable.

**Skill Categories**:
- **AI Code Generation Pipeline** (10): `upy-analyze`, `upy-select-hw`, `upy-scaffold`, `upy-generate`, `upy-simulate`, `upy-deploy`, `upy-autofix`, `upy-wiring`, `upy-diagram`, `upy-gen-driver`
- **Code Review**: `review` (mpy-review, MPY driver code review)
- **Standardization**: `upy-norm-driver`, `upy-norm-main`, `upy-norm-pkg` (Orchestrator)
- **Generation**: `upy-gen-main`, `upy-gen-readme`, `upy-gen-pkg`
- **Optimization**: `upy-opt-driver` (performance), `upy-slim-driver` (memory)
- **Packaging**: `upy-pack-driver`
- **Project Generation**: `upy-project` (end-to-end)
- **Tools**: `upy-pkg-guide` (device usage), `fetch-doc` (URL content fetching)

---

## Specification Documents

Full specification: [upy_driver_dev_spec_summary.md](https://github.com/FreakStudioCN/GraftSense-Drivers-MicroPython/blob/main/upy_driver_dev_spec_summary.md)

---

## Version History

| Version | Date | Author | Description |
|---|---|---|---|
| v1.0.0 | 2026-04-24 | leezisheng | Initial version, includes 5 skills |
| v1.1.0 | 2026-04-26 | leezisheng | Added upy-pack-driver; upy-norm-driver supplemented 16a/16b/16c; unified license to MIT; I2C scan specification |
| v1.2.0 | 2026-04-27 | leezisheng | Added upy-norm-pkg (Orchestrator), upy-opt-driver (performance optimization), upy-slim-driver (memory optimization); improved multi-file batch processing mode |
| v1.3.0 | 2026-04-29 | leezisheng | Added upy-pkg-guide (device usage query), fetch-doc (URL content fetching), upy-project (end-to-end project generation); upy-gen-pkg query logic changed to Bash curl automatic execution |
| v1.4.0 | 2026-05-04 | leezisheng | Added mpremote-device-interaction, mpremote-file-transfer, mpremote-live-session; based on andrewleech/claude-mpy-marketplace architecture, supplemented Windows (COMn) and macOS platform support |
| v1.5.0 | 2026-05-14 | leezisheng | Added upy-deploy-test (device deployment and verification); upy-norm-pkg added step 6 calling upy-deploy-test; each skill added middleware library type judgment branch and sensitive data replacement rules |
| v1.6.0 | 2026-06-02 | leezisheng | Added "One-Sentence Hardware Building" AI code generation pipeline (10 skills): analyze/select-hw/scaffold/generate/simulate/deploy/autofix/wiring/diagram/cold-driver + overall architecture document. upy-simulate changed to CLI+rich preferred. upy-select-hw added pin electrical type enumeration + physical pin rules. Total skills increased from 15 to 25. |
| v1.7.0 | 2026-06-03 | leezisheng | upy-cold-driver renamed to upy-gen-driver, positioned as an independently callable skill (not just an exception path). upy-gen-driver process implemented: debug version driver → mpremote hardware verification loop → remove debug → standardize. upy-wiring + upy-diagram added HTML output (self-contained browser page, Mermaid.js CDN + Tab switching), `--format all` now outputs all four formats: md + svg + png + html. All 25 skills supplemented with .skillfish.json. |
| v1.7.1 | 2026-06-03 | leezisheng | README.md installation script supplemented with upy-deploy-test + review skill. Feature planning.md fixed: Module 4 visualization scheme (Pillow→Mermaid), Module 7 gen-driver process supplemented with hardware verification loop, triage.py line count correction, project architecture script name refresh, /cold-driver→/gen-driver. |

---

## License

MIT License

Copyright (c) 2026 leezisheng

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
