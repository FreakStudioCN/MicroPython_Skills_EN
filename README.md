# MicroPython Skills for GraftSense

GraftSense MicroPython Skill collection, containing **25 dedicated Skills**, divided into two major systems:

**A. One-Sentence Hardware Building — AI Embedded Code Generation Pipeline (10 skills)**: From natural language requirements, automatically complete the full closed loop of hardware selection, code generation, PC simulation, flashing deployment, and error fixing.

**B. Driver Development Standardization (15 skills)**: Based on the complete writing specification (22 chapters, 2200+ lines) from the [GraftSense-Drivers-MicroPython](https://github.com/FreakStudioCN/GraftSense-Drivers-MicroPython) repository, covering driver standardization, test file generation, README generation, performance optimization, memory optimization, packaging, and device deployment.

> **Current source of truth (2026-07)**: This repository has completed the VS Code plugin version 8-process Skill/plugin. If the downstream repository's submodule only shows 6 plugins, it means the downstream pinned commit is outdated. You should bump/sync to the latest commit of this repository, rather than assuming wiring/diagram is missing.

---

## Current Skill / Plugin Full Overview

This repository maintains two types of assets:

- **Plugin version Skill/plugin**: For Blockless VS Code plugin and automation workflow host. Directory names end with `-plugin` and include `.codex-plugin/plugin.json`.
- **Classic Skill**: For direct invocation by Claude Code / Skillfish. Directory names typically do not end with `-plugin` and retain the original `/upy-*` command-style usage.

### Plugin Version 8-Process Skill/plugin

| # | Skill/plugin | Type | Process Position | Description |
|---|---|---|---|---|
| 1 | `upy-analyze-plugin` | Main chain mandatory | analyze | Requirement parsing, device identification, driver data search, output manifest_content |
| 2 | `upy-select-hw-plugin` | Main chain mandatory | select-hw | Official board selection, local overlay merge, pin plan, BOM, `board_unavailable` |
| 3 | `upy-flash-mpy-firmware-plugin` | Main chain mandatory | flash | MicroPython firmware parsing, download, flashing or UF2/manual guidance |
| 4 | `upy-scaffold-plugin` | Main chain mandatory | scaffold | Generate project skeleton, directory structure, templates, session/checkpoint/file_manifest |
| 5 | `upy-generate-plugin` | Main chain mandatory | generate | Business code, driver adaptation, quality gates, optional wiring/diagram entry |
| 6 | `upy-deploy-plugin` | Main chain mandatory | deploy | mpremote upload, run, REPL log, marker, device-side verification |
| 7 | `upy-wiring-plugin` | Optional artifact process | wiring | Generate wiring diagram and artifact from manifest/code/pin plan |
| 8 | `upy-diagram-plugin` | Optional artifact process | diagram | Generate architecture diagram, flowchart, data flow diagram and artifact |

### Missing Hardware Driver Branch

| Skill/plugin | Type | Description |
|---|---|---|
| `upy-gen-driver-plugin` | Plugin version missing driver branch | Missing hardware driver generation process for VS Code plugin, supporting pipeline/standalone/fix, PDF/Arduino/GitHub/chip model/manual fact input, hardware verification status and pre-generate gate |
| `upy-gen-driver` | Classic Skill | Original missing driver generation Skill, retains direct invocation and rule sources; plugin version should not overwrite this directory |

### Classic One-Sentence Hardware Building Pipeline Skills

| Skill | Description |
|---|---|
| `upy-analyze` | Natural language requirement parsing, device list, driver API reference |
| `upy-select-hw` | MCU/board selection, pin assignment, BOM |
| `upy-scaffold` | Generate firmware/ project skeleton |
| `upy-generate` | Download drivers, generate DI architecture business code, Mock and unittest |
| `upy-simulate` | PC-side CLI/rich full-process simulation, no real hardware required |
| `upy-deploy` | mpremote upload, flash, persistent session and PASS/FAIL initial judgment |
| `upy-autofix` | Triage after deploy failure, hierarchical decision-making and upstream skill delegation for repair |
| `upy-wiring` | Classic wiring diagram generation Skill |
| `upy-diagram` | Classic architecture diagram, flowchart, data flow diagram generation Skill |
| `upy-project` | Early end-to-end project generation entry, suitable for directly generating code and debugging flow from project description |

### Driver Standardization, Generation, Optimization and Packaging Skills

| Skill | Description |
|---|---|
| `upy-norm-driver` | Rewrite a usable but non-standard MicroPython driver into GraftSense standard format |
| `upy-norm-main` | Standardize `main.py` test file without changing test logic |
| `upy-norm-pkg` | Full-process driver package standardization orchestrator |
| `upy-gen-main` | Generate a complete `main.py` test file from scratch based on driver `.py` |
| `upy-gen-readme` | Generate README from scratch based on driver `.py` |
| `upy-gen-pkg` | Generate `package.json` from scratch based on driver directory or `.py` |
| `upy-opt-driver` | Rewrite MicroPython code according to performance optimization guide |
| `upy-slim-driver` | Reduce RAM usage according to memory footprint minimization guide |
| `upy-pack-driver` | Organize driver, main, README, package.json into standard driver package directory |
| `upy-deploy-test` | Device deployment and verification Skill, can be used for driver package acceptance |

### Query, Review and Device Tool Skills

| Skill | Description |
|---|---|
| `upy-pkg-guide` | Query device driver package usage, integrate upypi, awesome-micropython, README/API information |
| `fetch-doc` | Fetch URL / GitHub / upypi page content for other Skills to supplement data |
| `review` | MicroPython code review, based on historical review patterns for auxiliary checking |
| `mpremote-device-interaction` | Device connection, status query, firmware version, memory and file information |
| `mpremote-file-transfer` | Copy files between local and device, manage device file system |
| `mpremote-live-session` | Long connection and output monitoring, suitable for asyncio/aiorepl or long-running scenarios |

### Supporting Directories

| Directory | Description |
|---|---|
| `shared-plugin-scripts` | Shared scripts for plugin version and device/mpremote tools |
| `upy-project-gen-toolchain-spec` | Project generation toolchain, protocol, manifest/schema and plugin interface reference |
| `scripts` | Repository maintenance scripts, e.g., documentation sync and translation tools |

---

## 📚 Repository Documentation Description

This repository contains the following core documents. It is recommended to read as needed:

| Document | Description | Applicable Scenario |
|---|---|---|
| [README.md](README.md) | This document, Skill installation and usage guide | Quick start, install Skills |
| [upy_driver_dev_spec_summary.md](upy_driver_dev_spec_summary.md) | **GraftSense Driver Writing Specification Full Version** (22 chapters, 2200+ lines), covering file structure, class design, docstring, type annotations, parameter validation, exception handling, ISR specification and all other rules | Deep understanding of specification details, reference when manually writing drivers |
| [MicroPython_Performance_Optimization_Guide.md](MicroPython_Performance_Optimization_Guide.md) | **MicroPython Performance Optimization Guide**, detailed explanation of `@viper`, `@native`, `const()`, pre-allocated buffers, `memoryview`, pointer access and other optimization techniques, with measured data and code examples | Optimize driver execution speed, understand the optimization principles of `upy-opt-driver` |
| [MicroPython_Memory_Footprint_Minimization_Guide.md](MicroPython_Memory_Footprint_Minimization_Guide.md) | **MicroPython Memory Footprint Minimization Guide**, detailed explanation of frozen modules, `.mpy` files, `const()`, buffer reuse, `gc` control, `__slots__`, generators and other memory optimization techniques, with REPL test code | Reduce driver RAM usage, understand the optimization principles of `upy-slim-driver` |

**Reading Suggestions**:
- Beginners: First read this README to install Skills, directly use `/upy-norm-driver` and other commands to standardize code
- Advanced: Read `upy_driver_dev_spec_summary.md` to understand specification details, manually write compliant drivers
- Optimization: Read the performance/memory optimization guides to understand the optimization principles of `upy-opt-driver` and `upy-slim-driver`

---

## Table of Contents

- [Current Skill / Plugin Full Overview](#current-skill--plugin-full-overview)
- [Installation Methods](#installation-methods)
- [One-Sentence Hardware Building — AI Embedded Code Generation Pipeline](#one-sentence-hardware-building--ai-embedded-code-generation-pipeline)
- [Driver Development Standardization Skill List](#skill-list)
  - [upy-norm-driver](#upy-norm-driver--driver-file-standardization)
  - [upy-norm-main](#upy-norm-main--test-file-standardization)
  - [upy-gen-main](#upy-gen-main--generate-test-file-from-scratch)
  - [upy-gen-readme](#upy-gen-readme--generate-readme-from-scratch)
  - [upy-gen-pkg](#upy-gen-pkg--generate-packagejson-from-scratch)
  - [upy-norm-pkg](#upy-norm-pkg--driver-package-full-process-standardization)
  - [upy-deploy-test](#upy-deploy-test--device-deployment-and-verification)
  - [upy-opt-driver](#upy-opt-driver--driver-performance-optimization)
  - [upy-slim-driver](#upy-slim-driver--driver-memory-optimization)
  - [upy-pack-driver](#upy-pack-driver--package-into-standard-directory-structure)
  - [upy-pkg-guide](#upy-pkg-guide--device-driver-usage-query)
  - [fetch-doc](#fetch-doc--url-content-fetching)
  - [upy-project](#upy-project--micropython-project-end-to-end-generation)
  - [mpremote-device-interaction](#mpremote-device-interaction--device-connection-and-status-query)
  - [mpremote-file-transfer](#mpremote-file-transfer--device-file-transfer)
  - [mpremote-live-session](#mpremote-live-session--persistent-connection-and-output-monitoring)
- [How It Works](#how-it-works)
- [Specification Document](#specification-document)
- [Version History](#version-history)
- [License](#license)

---

## Installation Methods

> **Network restricted?** It is recommended to use the "Local Installation" method below. No network required, just clone the repository and copy.

### Method 1: Local Installation (Recommended, No Network Required)

**Applicable Scenario**: Network restricted, offline environment, or already cloned this repository locally.

**Step 1**: Clone this repository (or directly download ZIP and extract)

```bash
git clone https://github.com/FreakStudioCN/MicroPython_Skills.git
```

**Step 2**: Copy the skill directory to Claude Code's skills directory

The skills directory is fixed at `~/.claude/skills/`, expanded by operating system as follows:

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

**Step 3**: Restart Claude Code, the skills will take effect.

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

Or one-click install all:

```bash
for skill in upy-norm-driver upy-norm-main upy-gen-main upy-gen-readme \
             upy-gen-pkg upy-norm-pkg upy-opt-driver upy-slim-driver upy-pack-driver; do
  npx skillfish add FreakStudioCN/MicroPython_Skills $skill
done
```

---

## One-Sentence Hardware Building — AI Embedded Code Generation Pipeline

Users only need to describe requirements in natural language ("Make a temperature and humidity monitor, buzzer alarm when threshold exceeded"), and the system automatically completes the full process from selection, code generation, PC simulation, flashing to error fixing.

### Pipeline Overview

```
User says one sentence
    ↓
Phase 1: upy-analyze    → Requirement parsing + driver search
Phase 2: upy-select-hw  → MCU selection + pin assignment + BOM
Phase 3: upy-scaffold   → Project skeleton generation
Phase 4: upy-generate   → Business code generation
Phase 4.5: upy-simulate → PC-side full-process simulation (no hardware required)
Phase 5: upy-deploy     → One-click flash and run
Phase 6: upy-autofix    → Error hierarchical decision + delegated repair
Phase 7: upy-wiring     → Wiring diagram generation
       upy-diagram      → Architecture diagram + flowchart
Exception path: upy-gen-driver → Uncommon hardware driver generation
```

### Skill List

| # | Skill | Phase | Status | Description |
|---|-------|------|------|------|
| 1 | `upy-analyze` | Phase 1 | Implemented | Natural language → Device list + Driver API reference |
| 2 | `upy-select-hw` | Phase 2 | Implemented | MCU selection + firmware verification + pin assignment + BOM |
| 3 | `upy-scaffold` | Phase 3 | Implemented | Generate firmware/ complete skeleton (Timer/asyncio/Thread) |
| 4 | `upy-generate` | Phase 4 | Implemented | Driver download + DI architecture business code + Mock + unittest |
| 5 | `upy-simulate` | Phase 4.5 | Implemented | PC-side CLI+rich full-process simulation (data generator + multiple scenarios) |
| 6 | `upy-deploy` | Phase 5 | Implemented | mpremote upload + flash + persistent session + initial PASS/FAIL judgment |
| 7 | `upy-autofix` | Phase 6 | Implemented | Orchestration coordination layer: triage.py collection → LLM hierarchical decision → delegate upstream skill |
| 8 | `upy-wiring` | Phase 7 | Implemented | Wiring diagram generation (Mermaid .md + SVG + PNG + HTML) |
| 9 | `upy-diagram` | Phase 7 | Implemented | Architecture diagram + flowchart + data flow diagram (Mermaid .md + SVG + PNG + HTML) |
| 10 | `upy-gen-driver` | Exception path | Implemented | PDF/Arduino → debug version driver → hardware verification loop → standardized MPY driver |

**Supporting Tools:**
- `upy-project-gen-toolchain-spec` — Overall architecture documentation + manifest/schema definitions
- `upy-pkg-guide` — Device driver usage query (called by upy-analyze)
- `fetch-doc` — URL content fetching (called by upy-pkg-guide)

### Each Skill Introduction

#### `/upy-analyze` — Requirement Parsing + Driver Search

Input user natural language description, LLM decomposes intent → multi-keyword parallel search upypi + awesome-micropython → extract driver API reference → output `project-manifest.json` (phase: analyze).

#### `/upy-select-hw` — MCU Selection + Pin Assignment

Read manifest → recommend MCU based on scenario/power consumption/network requirements → I2C address conflict detection → generate pin assignment table (with electrical type enum + physical pin number) → output BOM bill of materials.

#### `/upy-scaffold` — Project Skeleton Generation

Read manifest → AskUserQuestion to select scheduling mode (Timer/asyncio/_thread) and optional modules → call `init_scaffold.py` to generate complete `firmware/` skeleton (board.py, conf.py, boot.py, main.py, drivers/*, tasks/*, lib/*, tools/*).

#### `/upy-generate` — Business Code Generation

Read firmware/ skeleton + driver API reference → download driver → upy-norm-driver standardize → generate DI architecture task code + conf.py + main.py + Mock layer + unittest → black + flake8 + pylint verification.

#### `/upy-simulate` — PC-side Full-Process Simulation

LLM reads all firmware/ code → self-design: scheduling scheme + data generator `gen_xxx(tick)` + visualization (CLI+rich preferred) + multi-scenario coverage → generate `test/pc/sim_main.py` → flake8 + pylint verification → run. **No real hardware required to verify business logic.**

#### `/upy-deploy` — One-Click Flash and Run

mpremote upload firmware/ → verify file integrity → soft reset + reconnect wait → persistent session collect output → device-side log capture → local rule initial PASS/FAIL judgment.

#### `/upy-autofix` — Orchestration Coordination Layer

Automatically enters after deploy failure. `triage.py` collects structured data (error parsing + I2C hardware detection + git management) → LLM reads JSON + raw logs → hierarchical decision (P0~P3) → delegate upstream skill for repair (generate/select-hw/analyze) → optional PC verification → redeploy. Maximum 3 attempts.

#### `/upy-wiring` — Wiring Diagram Generation

Read all .py source code in firmware/ to extract actual pins/addresses/buses → cross-verify with manifest → LLM generates intermediate JSON → script renders Mermaid wiring diagram .md + SVG + PNG + self-contained HTML (double-click browser to view) + pin cross-reference table.

#### `/upy-diagram` — Architecture Diagram + Flowchart

Scan firmware/ code structure + manifest → LLM generates intermediate JSON → script renders Mermaid architecture diagram + flowchart + data flow diagram, each output .md + SVG + PNG + self-contained HTML (Tabs switch between diagram/source code, dark mode adaptive). Supports simple/medium/detailed three complexity levels.

#### `/upy-gen-driver` — Driver Code Generation (Exception Path)

Triggered when no driver exists on upypi + GitHub. Extract information from PDF datasheet or Arduino code → LLM generates debug version single-file driver (with full self-check logic) → `mpremote resume run` hardware verification loop (up to 10 rounds) → remove debug → `upy-norm-driver` standardize. Can be called by `upy-analyze`, `upy-autofix` or directly by user.

---

### `/upy-norm-driver` — Driver File Standardization

**Purpose**: Rewrite a usable but non-standard MicroPython driver `.py` file (not `main.py`) according to GraftSense specification, outputting the complete standardized file.

**Input**: Existing driver `.py` file path

**Output**: Standardized complete `.py` file + rewrite description table

**Covered Rules**: P0 mandatory 38 items, P2 optional 7 items, including:

| Category | Main Rewrite Items |
|---|---|
| File Structure | File header 7-line comment, 4 module global variables, 6 section markers, section content specification |
| Class Design | Class structure layout, `__slots__` optimization, avoid multiple inheritance, explicit dependency injection, constant specification |
| Docstring | Class-level bilingual Chinese/English (including Attributes/Methods/Notes), method-level bilingual Chinese/English, ISR-safe annotation, side effect annotation |
| Type Annotations | `__init__` parameter annotations, public method return value annotations, callback using `callable` |
| Parameter Validation | Three modes: `isinstance`/`hasattr`/value range, `__init__` two-step validation |
| Exception Handling | Exception type standardization, `OSError` wrap re-raise (preserve `from e`), retry mechanism |
| ISR Specification | Prohibit memory allocation/blocking IO/exception throwing, `micropython.schedule`, concurrency protection |
| Function Design | Naming conventions, return value design, `debug` log switch |

**Core Constraint**: Do not modify external API names, method signature semantics, business logic, hardware communication timing.

**Usage Example**:
```
/upy-norm-driver sensors/bh1750_driver/code/bh_1750.py
```

---

### `/upy-norm-main` — Test File Standardization

**Purpose**: Rewrite an existing `main.py` test file according to specification without changing test logic.

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
| 6 | `while` loop only allowed in main program section |
| 7 | `raise`/`print` strings all in English |
| 8 | Main program section wrapped with `try/except KeyboardInterrupt/OSError/Exception/finally` |
| 9 | In `finally`, call `close()`/`deinit()`, `del` hardware objects, print exit prompt |
| 10 | Inline comments changed to Chinese |

**P1 Try to change**: High-frequency function comment default calls (for REPL manual invocation), three types of test scenario coverage check.

**Usage Example**:
```
/upy-norm-main sensors/bh1750_driver/main.py
```

---

### `/upy-gen-main` — Generate Test File from Scratch

**Purpose**: Given a driver `.py` file, analyze all its public APIs, generate a complete compliant `main.py` from scratch.

**Input**: Driver `.py` file path

**Output**: Complete `main.py` + API coverage description

**Full Coverage Principle**:

Classify all APIs by chip type functional dimensions:

| Chip Type | Coverage Dimensions |
|---|---|
| Sensor Type | Basic status query, core data acquisition, parameter configuration, mode switching, calibration/compensation |
| Motor Driver Type | Hardware initialization, motion control, status reading, reset/sleep |
| Communication Module Type | Network/protocol configuration, data send/receive, status query, power control |
| Memory Chip Type | Data read/write, address configuration, erase/reset |
| GPIO/Bus Expander Type | Pin configuration, level read/write, interrupt configuration |

Cover three types of test scenarios: normal parameters, boundary parameters (hardware limit values), abnormal parameters (verify exceptions are correctly thrown).

API handling method: Low-frequency APIs execute automatically, high-frequency/mode switching APIs comment calls (for REPL manual triggering).

**Usage Example**:
```
/upy-gen-main sensors/bh1750_driver/code/bh_1750.py
```

---

### `/upy-gen-readme` — Generate README from Scratch

**Purpose**: Given a driver `.py` file, analyze its functionality and APIs, generate a complete `README.md` from scratch.

**Input**: Driver `.py` file path (optional: existing README as reference)

**Output**: Complete `README.md`

**13 Mandatory Sections**:

| # | Section | Content |
|---|---|---|
| 1 | Title | `# [Chip Name] MicroPython Driver` |
| 2 | Table of Contents | Anchor links for all sections |
| 3 | Introduction | Driver purpose, functionality, applicable scenarios |
| 4 | Key Features | Feature highlights list |
| 5 | Hardware Requirements | Recommended hardware + pin description table |
| 6 | Software Environment | Firmware version, dependency libraries |
| 7 | File Structure | File tree (`├──` format) |
| 8 | File Description | Explain purpose per file |
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

### `/upy-norm-pkg` — Driver Package Full-Process Standardization

**Purpose**: For an existing verified driver file, execute the complete standardization process on the entire driver package directory as an Orchestrator Skill.

**Input**: Driver package directory path

**Output**: Complete standardized driver package (all driver files + main.py + README.md + package.json + standard directory structure)

**Execution Flow (6 Steps)**:

| Step | Operation |
|---|---|
| 0 | Scan directory, classify driver files and `main.py`; for multiple driver files, list and ask user to confirm scope |
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

**Purpose**: After `upy-norm-pkg` completes, upload the standardized driver files and `main.py` to the MicroPython device, run and verify the output.

**Input**: Standardized `code/` directory path + user confirmed COM port

**Output**: Upload progress + verification report (success/failure + error analysis)

**Execution Flow (6 Steps)**:

| Step | Operation |
|---|---|
| 0 | Ask and confirm COM port (can execute `mpremote connect list` for assistance) |
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

**Purpose**: For any MicroPython `.py` file (driver file, `main.py` or other files), rewrite according to GraftSense performance optimization guide, focusing on **execution speed** improvement.

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

**Core Constraints**: `@viper` rewrite must annotate integer overflow risk; `@native` must annotate limitations (no generators/keyword arguments); SIO registers must annotate "RP2040 specific".

**Usage Example**:
```
/upy-opt-driver sensors/bh1750_driver/code/bh_1750.py
/upy-opt-driver sensors/bh1750_driver/code/
```

---

### `/upy-slim-driver` — Memory Optimization

**Purpose**: For any MicroPython `.py` file (driver file, `main.py` or other files), rewrite according to GraftSense memory minimization guide, focusing on **RAM usage** reduction.

**Input**: Driver `.py` file path or directory path (supports batch optimization of multiple files)

**Output**: Optimized complete `.py` file + optimization description table

**Optimization Priority**:

| Priority | Item | Typical Savings |
|---|---|---|
| P0 | Pre-allocated buffers | Eliminate peak heap allocation |
| P0 | Private `_CONST` | ~40 bytes/constant |
| P0 | Avoid loop string `+` | Eliminate temporary objects |
| P0 | `bytes`/`bytearray` instead of `list` | ~90% (register tables) |
| P1 | `gc.collect()` pre-position | Reduce randomness |
| P1 | `gc.disable()`/`gc.enable()` | Prevent GC interruption mid-operation |
| P1 | `struct.pack_into()` | Eliminate temporary bytes |
| P2 | `__slots__` | 50–200 bytes/instance |
| P2 | Generator instead of list | Peak RAM O(N)→O(1) |

**Core Constraints**: `_CONST` rewrite only applies to module internal constants; `gc.disable()` interval must be short and bounded, must not include blocking I/O; overlaps with `upy-opt-driver` P0#1 (pre-allocated buffers), do not execute repeatedly.

**Usage Example**:
```
/upy-slim-driver sensors/bh1750_driver/code/bh_1750.py
/upy-slim-driver sensors/bh1750_driver/code/
```

---

### `/upy-pack-driver` — Package into Standard Directory Structure

**Purpose**: After other Skills have executed, organize the driver file, `main.py`, `README.md`, `package.json` into a standard driver package directory structure, and generate a `LICENSE` file.

**Input**: Driver `.py` file path (the same directory must already have `main.py`, `README.md`, `package.json`)

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

**Core Constraint**: Does not generate any content, only responsible for organizing files; missing files will prompt to run the corresponding Skill first.

**Usage Example**:
```
/upy-pack-driver bmp280.py
```

---

### `/upy-pkg-guide` — Device Driver Usage Query

**Purpose**: Given a device name, automatically fetch all files of the corresponding driver package from upypi, comprehensively analyze and output usage key points.

**Input**: Device/chip name (e.g., BMP280, DS18B20, MPR121)

**Output**: Package information, installation command, initialization example, core API table, notes

**Execution Flow**: curl search upypi → get package.json → parallel download driver.py + main.py + README.md → comprehensive output

**Usage Example**:
```
/upy-pkg-guide BMP280
/upy-pkg-guide DS18B20
```

---

### `/fetch-doc` — URL Content Fetching

**Purpose**: Given any URL, automatically fetch content and extract key information. Supports GitHub files, upypi package pages, regular web pages.

**Input**: URL (GitHub blob links are automatically converted to raw URLs)

**Output**: Extract key information based on content type (README summary, driver API table, package.json fields, etc.)

**Dependencies**: Requires Python + requests library (`pip install requests`)

**Usage Example**:
```
/fetch-doc https://github.com/FreakStudioCN/GraftSense-Drivers-MicroPython/blob/main/sensors/bmp280_driver/README.md
```

---

### `/review` — MicroPython Code Review

**Purpose**: Based on MicroPython maintainer historical review patterns (~19.5K classified review comments), perform AI-assisted review of MPY driver code.

**Input**: MicroPython code changes (branch, commit, diff, PR)

**Output**: Semantically matched historical review patterns + review context suggestions

**Core Capabilities**:
- Semantic search of ~19.5K classified review comments to find relevant historical review patterns
- Supports MCP server (`review_diff`, `search_reviews` and other tools) and CLI methods
- MCP server keeps embedding model warm, eliminating 2-3s cold start per query

**Usage Example**:
```
/review Review the diff of the current branch against main
/review Check sensors/bmp280_driver/code/bmp280.py
```

---

### `/upy-project` — MicroPython Project End-to-End Generation

**Purpose**: User describes project requirements, automatically completes the full process from requirement clarification, device selection, code generation to device debugging.

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
/upy-project Make a temperature monitor with ESP32 and BMP280, print every 5 seconds, COM3 port
```

---

### `/upy-gen-pkg` — Generate package.json from Scratch

**Purpose**: Given a driver directory or driver file, analyze structure and dependencies, generate a compliant `package.json` from scratch.

**Input**: Driver directory path or driver `.py` file path

**Output**: Complete `package.json` + three installation method commands

**Dependency Handling Three-Step Priority**:

```
1. MicroPython built-in modules (machine, time, sys, etc.) → Do not write to deps
2. micropython-lib standard library → Use mip standard format
3. Other third-party dependencies → Query https://upypi.net/api/search?q={dependency_name}
   Has result → Write deps with upypi URL
   No result → Use github: placeholder format, annotate ⚠️ requires manual confirmation
```

**Usage Example**:
```
/upy-gen-pkg sensors/bh1750_driver/
```

---

### `/mpremote-device-interaction` — Device Connection and Status Query

**Purpose**: Connect to MicroPython device via mpremote, execute code, query device status (memory, firmware version, file list, etc.).

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

**Purpose**: Use mpremote to copy files between local and device, manage device file system (ls, mkdir, rm, tree).

**Platform Support**: Windows, macOS, Linux. Device path writing for each platform is detailed within the Skill.

**Key Rule**: File operations must include `resume`, otherwise the device will soft reset before each operation.

**Covered Scenarios**:

| Scenario | Command Example |
|---|---|
| Upload file | `mpremote <device> resume fs cp main.py :main.py` |
| Download file | `mpremote <device> resume fs cp :main.py .` |
| Recursive sync directory | `mpremote <device> resume fs cp -r utils/ :utils/` |
| Restart after driver update | `mpremote <device> resume fs cp driver.py :driver.py + soft-reset repl` |
| List files | `mpremote <device> resume fs ls :` |
| View storage space | `mpremote <device> resume exec "import os; print(os.statvfs('/'))"` |

**Usage Example**:
```
/mpremote-file-transfer  Sync the local utils/ directory to the device, then restart and monitor
```

---

### `/mpremote-live-session` — Persistent Connection and Output Monitoring

**Purpose**: Establish a persistent connection to the device, continuously send commands and capture output. Suitable for asyncio devices, stress testing, long-term monitoring.

**Platform Support**: Linux/macOS use PTY solution; Windows use subprocess pipe alternative (has limitations, see Skill for details).

**Core Principle**: Repeatedly calling `mpremote resume exec` will send Ctrl+C to asyncio devices, killing the event loop; must use persistent session instead.

**When to Use**:

| Scenario | Recommended Solution |
|---|---|
| Single quick query | `mpremote <device> resume exec "..."` |
| Multi-command sequence / monitor output | This Skill (persistent session) |
| Device running asyncio/aiorepl | This Skill (mandatory) |
| File copy | mpremote-file-transfer |

**Usage Example**:
```
/mpremote-live-session  Establish persistent connection to /dev/tty.usbmodem1101, query memory every second and log to file
```

---

## How It Works

Each Skill is a `SKILL.md` file containing:

- **Role Positioning**: Tells AI what role to play
- **Core Constraints**: Clearly states what cannot be modified
- **Rewrite Priority Table**: P0 mandatory / P2 optional, each item corresponds to a specific chapter in the specification document
- **Key Specification Summary**: Embeds the most important code templates to avoid looking up the full specification document each time

### Trigger Flow

```
User input /upy-norm-driver xxx.py
    ↓
Claude loads the specification summary and priority table from SKILL.md
    ↓
Reads the target file, analyzes structure (communication interface type, class, method, ISR callback, etc.)
    ↓
Rewrites item by item according to P0→P2 priority (does not change API and business logic)
    ↓
Outputs complete standardized file + rewrite description table
```

### Why Split into Multiple Skills

The specification document has 22 chapters, 2200+ lines. Embedding the entire specification in a single Skill would cause excessive context length and degrade rewrite quality. By splitting according to "rewrite target" and "optimization goal", each Skill only embeds the specification summary of the corresponding chapters, keeping context manageable.

**Skill Classification**:
- **AI Code Generation Pipeline** (10): `upy-analyze`, `upy-select-hw`, `upy-scaffold`, `upy-generate`, `upy-simulate`, `upy-deploy`, `upy-autofix`, `upy-wiring`, `upy-diagram`, `upy-gen-driver`
- **Code Review**: `review` (mpy-review, MPY driver code review)
- **Standardization**: `upy-norm-driver`, `upy-norm-main`, `upy-norm-pkg` (Orchestrator)
- **Generation**: `upy-gen-main`, `upy-gen-readme`, `upy-gen-pkg`
- **Optimization**: `upy-opt-driver` (performance), `upy-slim-driver` (memory)
- **Packaging**: `upy-pack-driver`
- **Project Generation**: `upy-project` (end-to-end)
- **Tools**: `upy-pkg-guide` (device usage), `fetch-doc` (URL content fetching)

---

## Specification Document

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
| v1.6.0 | 2026-06-02 | leezisheng | Added "One-Sentence Hardware Building" AI code generation pipeline (10 skills): analyze/select-hw/scaffold/generate/simulate/deploy/autofix/wiring/diagram/cold-driver + overall architecture documentation. upy-simulate changed to CLI+rich preferred. upy-select-hw added pin electrical type enum + physical pin rules. Skill count increased from 15 to 25. |
| v1.7.0 | 2026-06-03 | leezisheng | upy-cold-driver renamed to upy-gen-driver, positioned as an independently callable skill (not just exception path). upy-gen-driver process implemented: debug version driver → mpremote hardware verification loop → remove debug → standardize. upy-wiring + upy-diagram added HTML output (self-contained browser page, Mermaid.js CDN + Tab switching), --format all now outputs all four formats: md + svg + png + html. All 25 skills completed .skillfish.json. |
| v1.7.1 | 2026-06-03 | leezisheng | README.md installation script supplemented upy-deploy-test + review skill. Function planning.md fixed: Module 4 visualization scheme (Pillow→Mermaid), Module 7 gen-driver process added hardware verification loop, triage.py line count correction, project architecture script name refresh, /cold-driver→/gen-driver. |
| v1.8.0 | 2026-07-05 | leezisheng | README.md added current Skill / Plugin full overview, clarified plugin version 8-process: `upy-analyze-plugin`, `upy-select-hw-plugin`, `upy-flash-mpy-firmware-plugin`, `upy-scaffold-plugin`, `upy-generate-plugin`, `upy-deploy-plugin`, `upy-wiring-plugin`, `upy-diagram-plugin`; supplemented `upy-gen-driver-plugin` as missing hardware driver branch, and distinguished between plugin version Skill/plugin and Classic Skill. |

---

## License

MIT License

Copyright (c) 2026 leezisheng

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
