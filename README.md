# MicroPython Skills for GraftSense

GraftSense MicroPython Skill collection, containing three categories of Skill assets: GraftSense hardware generation, driver normalization, and MicroPythonOS App publishing:

**A. One-sentence hardware generation — AI embedded code generation pipeline (10 skills)**: Starting from natural language requirements, automatically complete the full closed loop of hardware selection, code generation, PC simulation, flashing deployment, and error fixing.

**B. Driver development normalization (15 skills)**: Based on the complete writing specification (22 chapters, 2200+ lines) of the [GraftSense-Drivers-MicroPython](https://github.com/FreakStudioCN/GraftSense-Drivers-MicroPython) repository, covering driver normalization, test file generation, README generation, performance optimization, memory optimization, packaging, and device deployment.

**C. MicroPythonOS App / MPK / upystore pipeline (9 mpos skills)**: For [MicroPythonOS](https://docs.micropythonos.com/) App development, covering requirements analysis, dependency preparation, App generation, desktop testing, MPK packaging, real device/preview deployment, and upystore publishing preparation. This system does not include `mpos-debug-app`.

> **Current source of truth (2026-07)**: This repository has completed the VS Code plugin version of the 8-process Skill/plugin. If the submodule in a downstream repository only shows 6 plugins, it means the downstream pinned commit is outdated. You should bump/sync to the latest commit of this repository, rather than assuming wiring/diagram is missing.

---

## Current Skill / Plugin Full Overview

This repository maintains two types of assets simultaneously:

- **Plugin version Skill/plugin**: For the Blockless VS Code plugin and automated workflow host. Directory names end with `-plugin` and contain `.codex-plugin/plugin.json`.
- **Classic Skill**: For direct invocation by Claude Code / Skillfish. Directory names typically do not end with `-plugin` and retain the original `/upy-*` imperative usage.

### Plugin Version 8-Process Skill/plugin

| # | Skill/plugin | Type | Process Position | Description |
|---|---|---|---|---|
| 1 | `upy-analyze-plugin` | Main chain mandatory | analyze | Requirements parsing, device identification, driver data search, output manifest_content |
| 2 | `upy-select-hw-plugin` | Main chain mandatory | select-hw | Official board selection, local overlay merging, pin plan, BOM, `board_unavailable` |
| 3 | `upy-flash-mpy-firmware-plugin` | Main chain mandatory | flash | MicroPython firmware parsing, download, flashing or UF2/manual guidance |
| 4 | `upy-scaffold-plugin` | Main chain mandatory | scaffold | Generate project skeleton, directory structure, templates, session/checkpoint/file_manifest |
| 5 | `upy-generate-plugin` | Main chain mandatory | generate | Business code, driver adaptation, quality gates, optional wiring/diagram entry point |
| 6 | `upy-deploy-plugin` | Main chain mandatory | deploy | mpremote upload, run, REPL log, marker, device-side verification |
| 7 | `upy-wiring-plugin` | Optional product flow | wiring | Generate wiring diagram and artifact from manifest/code/pin plan |
| 8 | `upy-diagram-plugin` | Optional product flow | diagram | Generate architecture diagram, flowchart, data flow diagram and artifact |

### Missing Hardware Driver Branch

| Skill/plugin | Type | Description |
|---|---|---|
| `upy-gen-driver-plugin` | Plugin version missing driver branch | Missing hardware driver generation flow for VS Code plugin, supporting pipeline/standalone/fix, PDF/Arduino/GitHub/chip model/manual fact input, hardware verification status and pre-generate gate |
| `upy-gen-driver` | Classic Skill | Original missing driver generation Skill, retains direct invocation and rule sources; should not overwrite this directory when pluginizing |

### Classic One-Sentence Hardware Generation Pipeline Skills

| Skill | Description |
|---|---|
| `upy-analyze` | Natural language requirements parsing, device list, driver API reference |
| `upy-select-hw` | MCU/board selection, pin assignment, BOM |
| `upy-scaffold` | Generate firmware/ project skeleton |
| `upy-generate` | Download drivers, generate DI architecture business code, Mock and unittest |
| `upy-simulate` | PC-side CLI/rich full-process simulation, no real hardware required |
| `upy-deploy` | mpremote upload, flashing, persistent session and PASS/FAIL initial judgment |
| `upy-autofix` | Triage after deploy failure, hierarchical decision-making and upstream skill delegation for repair |
| `upy-wiring` | Classic wiring diagram generation Skill |
| `upy-diagram` | Classic architecture diagram, flowchart, data flow diagram generation Skill |
| `upy-project` | Early end-to-end project generation entry point, suitable for generating code and debugging flow directly from project description |

### Driver Normalization, Generation, Optimization and Packaging Skills

| Skill | Description |
|---|---|
| `upy-norm-driver` | Rewrite a usable but non-standard MicroPython driver into GraftSense standard format |
| `upy-norm-main` | Normalize `main.py` test file without changing test logic |
| `upy-norm-pkg` | Full-process normalization orchestrator for driver packages |
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
| `review` | MicroPython code review, assisted checking based on historical review patterns |
| `mpremote-device-interaction` | Device connection, status query, firmware version, memory and file information |
| `mpremote-file-transfer` | Copy files between local and device, manage device file system |
| `mpremote-live-session` | Persistent connection and output monitoring, suitable for asyncio/aiorepl or long-running scenarios |

### MicroPythonOS App / MPK / upystore Skills

| Skill | Type | Description |
|---|---|---|
| `mpos-dev` | Shared base layer | MicroPythonOS architecture, App/MPK constraints, MPOS/LVGL API reference, deployment targets and Web Port reference |
| `mpos-plan-app` | Orchestration entry point | State machine from natural language requirements to release handover, maintains `tmp/mpos-plan-app/<fullname>/plan_state.json` |
| `mpos-analyze-app` | Requirements analysis | Generate App identity, manifest draft, API/dependency/test/deployment plan |
| `mpos-prepare-deps` | Dependency preparation | Prepare App-layer pure Python/MPY dependencies, mark synchronous dependency adaptation requirements |
| `mpos-gen-app` | Code generation | Two-stage generation/repair of `internal_filesystem/apps/<fullname>/`, run static gates |
| `mpos-test-app` | Run tests | Linux SDL desktop runtime smoke, optional Web Port check, screenshots and widget tree |
| `mpos-package-app` | MPK packaging | Validate manifest/icon/entrypoint, generate `<fullname>_rN.mpk` and `app_index_entry.json` |
| `mpos-deploy-app` | Deployment/preview | desktop/web preview, `device-copy`, `mpk-install`, installer/flash guidance |
| `mpos-publish-app` | Publishing preparation | Simultaneously read package/test/deploy results, prepare upystore manual upload handover |

### Supporting Directories

| Directory | Description |
|---|---|
| `shared-plugin-scripts` | Shared scripts and device/mpremote tools for plugin version |
| `upy-project-gen-toolchain-spec` | Project generation toolchain, protocols, manifest/schema and plugin interface reference |
| `scripts` | Repository maintenance scripts, e.g., documentation sync and translation tools |

---

## 📚 Repository Documentation Description

This repository contains the following core documents. It is recommended to read them as needed:

| Document | Description | Applicable Scenario |
|---|---|---|
| [README.md](README.md) | This document, Skill installation and usage guide | Quick start, installing Skills |
| [upy_driver_dev_spec_summary.md](upy_driver_dev_spec_summary.md) | **Complete GraftSense driver writing specification** (22 chapters, 2200+ lines), covering file structure, class design, docstring, type annotations, parameter validation, exception handling, ISR specification and all other rules | Deep understanding of specification details, reference when manually writing drivers |
| [MicroPython_Performance_Optimization_Guide.md](MicroPython_Performance_Optimization_Guide.md) | **MicroPython performance optimization guide**, detailed explanation of `@viper`, `@native`, `const()`, pre-allocated buffers, `memoryview`, pointer access and other optimization techniques, with measured data and code examples | Optimize driver execution speed, understand the optimization principles of `upy-opt-driver` |
| [MicroPython_Memory_Footprint_Minimization_Guide.md](MicroPython_Memory_Footprint_Minimization_Guide.md) | **MicroPython memory footprint minimization guide**, detailed explanation of frozen modules, `.mpy` files, `const()`, buffer reuse, `gc` control, `__slots__`, generators and other memory optimization techniques, with REPL test code | Reduce driver RAM usage, understand the optimization principles of `upy-slim-driver` |

**Reading suggestions**:
- Beginners: First read this README to install Skills, directly use `/upy-norm-driver` and other commands to normalize code
- Advanced: Read `upy_driver_dev_spec_summary.md` to understand specification details, manually write compliant drivers
- Optimization: Read the performance/memory optimization guides to understand the optimization principles of `upy-opt-driver` and `upy-slim-driver`

---

## Table of Contents

- [Current Skill / Plugin Full Overview](#current-skill--plugin-full-overview)
- [Installation Methods](#installation-methods)
- [Maintainers: Automatic English Repository Sync](#maintainers-automatic-english-repository-sync)
- [MicroPythonOS App / MPK / upystore Workflow](#micropythonos-app--mpk--upystore-workflow)
- [One-Sentence Hardware Generation — AI Embedded Code Generation Pipeline](#one-sentence-hardware-generation--ai-embedded-code-generation-pipeline)
- [Driver Development Normalization Skill List](#skill-list)
  - [upy-norm-driver](#upy-norm-driver--driver-file-normalization)
  - [upy-norm-main](#upy-norm-main--test-file-normalization)
  - [upy-gen-main](#upy-gen-main--generate-test-file-from-scratch)
  - [upy-gen-readme](#upy-gen-readme--generate-readme-from-scratch)
  - [upy-gen-pkg](#upy-gen-pkg--generate-packagejson-from-scratch)
  - [upy-norm-pkg](#upy-norm-pkg--driver-package-full-process-normalization)
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
- [Specification Documents](#specification-documents)
- [Version History](#version-history)
- [License](#license)

---

## Installation Methods

> **Network restricted?** It is recommended to use the "Local Installation" method below, which requires no network. Simply clone the repository and copy.

### Method 1: Local Installation (Recommended, No Network Required)

**Applicable scenarios**: Network restricted, offline environment, or if you have already cloned this repository locally.

**Step 1**: Clone this repository (or directly download and extract the ZIP)

```bash
git clone https://github.com/FreakStudioCN/MicroPython_Skills.git
```

**Step 2**: Copy the skill directories to Claude Code's skills directory

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
             mpremote-device-interaction mpremote-file-transfer mpremote-live-session \
             mpos-dev mpos-plan-app mpos-analyze-app mpos-prepare-deps mpos-gen-app \
             mpos-test-app mpos-package-app mpos-deploy-app mpos-publish-app; do
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
            "mpremote-device-interaction","mpremote-file-transfer","mpremote-live-session",
            "mpos-dev","mpos-plan-app","mpos-analyze-app","mpos-prepare-deps","mpos-gen-app",
            "mpos-test-app","mpos-package-app","mpos-deploy-app","mpos-publish-app")
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

MicroPythonOS related skills are recommended for local installation to ensure `mpos-dev/reference/` and `scripts/` are copied together. If online installation is needed, you can add them one by one:

```bash
for skill in mpos-dev mpos-plan-app mpos-analyze-app mpos-prepare-deps mpos-gen-app \
             mpos-test-app mpos-package-app mpos-deploy-app mpos-publish-app; do
  npx skillfish add FreakStudioCN/MicroPython_Skills $skill
done
```

---

## Maintainers: Automatic English Repository Sync

The Chinese content in this repository is the source of truth. The English repository is located at:

```text
/home/leeqingshui/MicroPython_Skills_EN
```

On Linux hosts, automatic sync is not a default Git capability. You must first install the post-commit hook provided by this repository. After installation, every time you execute `git commit` in `/home/leeqingshui/MicroPython_Skills`, Git will automatically run:

```text
.githooks/post-commit
  -> scripts/sync_english_repo.py
     -> scripts/translate_to_english.py
     -> /home/leeqingshui/MicroPython_Skills_EN
     -> git add/commit
     -> git push
```

### Installing or Fixing the Hook

Execute in the Chinese repository root directory:

```bash
cd /home/leeqingshui/MicroPython_Skills
python3 scripts/install_git_hooks.py \
  --repo . \
  --english-repo /home/leeqingshui/MicroPython_Skills_EN
```

The script will write these local Git configurations:

```bash
git config core.hooksPath .githooks
git config skills.englishRepo /home/leeqingshui/MicroPython_Skills_EN
git config skills.englishPush true
```

Check if the installation was successful:

```bash
git config --get core.hooksPath
git config --get skills.englishRepo
git config --bool --get skills.englishPush
test -x .githooks/post-commit
```

Expected output should respectively contain:

```text
.githooks
/home/leeqingshui/MicroPython_Skills_EN
true
```

### API Key and Model

After the hook is triggered, it will call the LLM for translation, so it must be able to read the translation API key. It can be placed in shell environment variables or in the `env` section of `~/.claude/settings.json`.

Common configuration methods:

```bash
export DEEPSEEK_API_KEY=sk-...
export SKILLS_TRANSLATE_BACKEND=deepseek
export SKILLS_TRANSLATE_MODEL=deepseek-chat
```

If using an Anthropic-compatible gateway:

```bash
export ANTHROPIC_AUTH_TOKEN=sk-...
export ANTHROPIC_BASE_URL=https://example.com
export SKILLS_TRANSLATE_BACKEND=anthropic
export SKILLS_TRANSLATE_MODEL=deepseek-v4-pro
```

Without an API key, the hook will exit normally but will not sync the English repository; this is to prevent the Chinese repository's commit from failing.

### Viewing Hook Logs

The complete output of the post-commit hook is written to this repository's Git directory:

```bash
cd /home/leeqingshui/MicroPython_Skills
tail -n 120 "$(git rev-parse --git-path english-sync.log)"
```

If the English repository has no changes after a commit, check this log first. Common reasons:

- `core.hooksPath` is not set to `.githooks`
- No translation API key
- `/home/leeqingshui/MicroPython_Skills_EN` has uncommitted changes; the script will stop to avoid overwriting
- English repository push permission or network failure
- Translation interface model, base URL, or quota anomaly

Temporarily skip one English sync:

```bash
SKILLS_SKIP_EN_SYNC=1 git commit -m "your message"
```

Manually re-sync the current HEAD:

```bash
cd /home/leeqingshui/MicroPython_Skills
python3 scripts/sync_english_repo.py \
  --src-repo /home/leeqingshui/MicroPython_Skills \
  --en-repo /home/leeqingshui/MicroPython_Skills_EN \
  --source-mode head \
  --commit \
  --push
```

If you only want to see which files will be translated, without calling the API or modifying the English repository:

```bash
python3 scripts/sync_english_repo.py \
  --src-repo /home/leeqingshui/MicroPython_Skills \
  --en-repo /home/leeqingshui/MicroPython_Skills_EN \
  --source-mode head \
  --dry-run
```

---

## MicroPythonOS App / MPK / upystore Workflow

MicroPythonOS skills are an App development chain independent of the regular `upy-*` hardware projects. Their goal is to generate, test, package, and publish an App within the MicroPythonOS repository, specifically this directory:

```text
<MicroPythonOS repo>/internal_filesystem/apps/<fullname>/
  MANIFEST.JSON
  icon_64x64.png
  assets/main.py
```

It is not used for regular MicroPython bare-metal scripts, nor for casually modifying MicroPythonOS OS/framework/build source code. Simple understanding:

- Want to create an App that can be opened on MicroPythonOS: use `mpos-*`.
- Want to write a regular `main.py` for ESP32 / Pico: use `upy-*`.
- Want to fix MicroPythonOS system itself, LVGL binding, Web build, firmware build: This is not the default App flow. You must explicitly tell the AI "allow OS modification".

### Quick Start for Beginners

1. Confirm the skills are installed in Claude Code:

```bash
ls ~/.claude/skills/mpos-plan-app
ls ~/.claude/skills/mpos-gen-app
ls ~/.claude/skills/mpos-test-app
```

2. Prepare a MicroPythonOS repository. It is recommended to first use an isolated test repository, not directly in the main repository `/home/leeqingshui/MicroPythonOS`:

```bash
cd /home/leeqingshui/tmp/mpos-skill-cc-test-20260717
test -d internal_filesystem/apps
test -d scripts
```

3. It is recommended to first use `cc-switch` to switch the model used by Claude Code. MicroPythonOS App generation reads a large amount of API, JSON, and logs. Differences in model capabilities can affect stability; it is recommended to switch Claude Code's provider/model to the DeepSeek series models in `cc-switch`, e.g., `deepseek-chat` or `deepseek-reasoner`. The specific model name depends on the options available in your local `cc-switch` interface.

```bash
cc-switch
```

After opening the interface, select the corresponding Claude Code configuration, switch to DeepSeek, and then start Claude Code.

4. Start Claude Code in the root directory of the MicroPythonOS repository:

```bash
cd /home/leeqingshui/tmp/mpos-skill-cc-test-20260717
claude
```

5. Enter the slash command in the Claude Code conversation. The format is fixed as:

```text
/skill-name your requirements description
```

Note that `/skill-name` is not a shell command; it is not executed in the terminal. It is sent to the AI in Claude Code's chat input box.

It is recommended to start from the main entry point:

```text
/mpos-plan-app Create a four-function calculator App in /home/leeqingshui/tmp/mpos-skill-cc-test-20260717, use com.example.cc_skill_smoke as fullname, go through analysis, generation, test, package, deployment record and upystore publishing preparation
```

If you only want to test an existing App:

```text
/mpos-test-app Test com.example.cc_skill_smoke in /home/leeqingshui/tmp/mpos-skill-cc-test-20260717, read tmp/mpos-plan-app/com.example.cc_skill_smoke/generation_result.json, run desktop smoke, generate PNG screenshot, and provide the complete simulator launch command
```

If the previous step fails, feed the failure information back to the corresponding skill, rather than starting over from the beginning:

```text
/mpos-gen-app Fix com.example.cc_skill_smoke. Below is the failure output from mpos-test-app and the path to app_test_result.json: ...
```

### How the 9 mpos Skills Divide Work

Generally, beginners only need to remember: **Use `/mpos-plan-app` for the complete flow; use the corresponding stage skill only if a specific stage has a problem.**

| Stage | Which to Use | What It Does Internally |
|---|---|---|
| Main entry / Continue task | `/mpos-plan-app` | Find the current App, read `plan_state.json`, decide the next step: analysis, generation, test, package, deploy, or publish |
| Requirements analysis | `/mpos-analyze-app` | Convert natural language requirements into App name, `fullname`, `publisher`, manifest draft, API plan, test plan, deployment plan |
| External dependencies | `/mpos-prepare-deps` | Only prepare App-layer pure Python/MPY dependencies, determine if synchronous libraries need an adapter, do not modify the OS |
| Generate / Fix App | `/mpos-gen-app` | Create or modify `internal_filesystem/apps/<fullname>/`, then run static gates: manifest, syntax, API, lint, App-only |
| Run tests | `/mpos-test-app` | Use Linux desktop runtime and `mpos_controller.py` to start the App, check startup, visible text, widget tree, screenshots |
| Package MPK | `/mpos-package-app` | Validate manifest/icon/entrypoint, generate `<fullname>_rN.mpk` and `app_index_entry.json` |
| Preview / Real device deployment | `/mpos-deploy-app` | Perform desktop preview, web preview, `device-copy` or `mpk-install` record |
| Publishing preparation | `/mpos-publish-app` | Simultaneously read package/test/deploy results, check upystore version and publishing metadata, only provide manual upload guidance |
| Shared knowledge base | `mpos-dev` | Other skills read it automatically; users usually do not need to call it directly |

### Internal Logic

Each App will have a project state directory:

```text
<repo-root>/tmp/mpos-plan-app/<fullname>/
  plan_state.json
  activity_log.jsonl
```

`plan_state.json` records the current step, which artifacts exist, and which skill should be called next. `activity_log.jsonl` is the activity log. Do not manually write these two files; the skills will automatically maintain them via `update_plan_state.py`.

Common artifact paths:

```text
tmp/mpos-plan-app/<fullname>/analysis_result.json
tmp/mpos-plan-app/<fullname>/generation_result.json
tmp/mpos-test-app/<fullname>/app_test_result.json
tmp/mpos-package-app/<fullname>/package_result.json
tmp/mpos-package-app/<fullname>/<fullname>_r1.mpk
tmp/mpos-deploy-app/<fullname>/deploy_result.json
tmp/mpos-publish-app/<fullname>/publish_result.json
```

If you want to continue after an interruption, simply tell Claude Code:

```text
/mpos-plan-app Continue com.example.cc_skill_smoke, read the existing plan_state.json and artifacts from each stage, do not start from scratch
```

### Mandatory Usage Requirements

- Must be run within a MicroPythonOS repo; the repository root should contain `internal_filesystem/apps` and `scripts`.
- Build, desktop simulator, Web preview, and integration testing should be placed in an isolated clone/worktree/temporary copy by default; do not directly pollute the main repository at `/home/leeqingshui/MicroPythonOS`.
- mpos-related skills must fully read `mpos-dev/reference/mpos_api_summary.json` and `mpos-dev/reference/lvgl_api_summary.json`. Do not omit this step because the task seems simple.
- Only modify the target App directory `internal_filesystem/apps/<fullname>/` and `tmp/mpos-*` artifacts. Do not modify MicroPythonOS OS/build/framework/existing App code, unless the user explicitly requests OS modification.
- The generation phase must run API cross-validation and App-only change checks. Unknown `lv.*` / `mpos.*` APIs, missing `buttonmatrix.set_map()` terminators, or accidental modification of files outside the target App should be returned to the AI for repair first.
- The manifest must contain a non-empty `publisher`, e.g., `com.example`. MPK filenames must use the upystore revision format: `<fullname>_rN.mpk`.
- Currently, `mpos-debug-app` is not needed and should not be used.

### How to Describe Requirements When Generating an App

The more specific the description, the less likely the AI will go off track. It is recommended to include at least:

- Repository path: e.g., `/home/leeqingshui/tmp/mpos-skill-cc-test-20260717`.
- App fullname: e.g., `com.example.cc_skill_smoke`.
- App goal: One sentence describing what it does.
- UI requirements: What buttons, text, lists, input fields are needed.
- Whether a real device is needed: If no real device, say "only do desktop-preview"; if there is a real device, provide the board and serial port.
- Whether to publish to upystore: Publishing requires `publisher`, screenshots, short description, long description, release notes.

Example:

```text
/mpos-plan-app Create com.example.cc_skill_smoke in /home/leeqingshui/tmp/mpos-skill-cc-test-20260717.
The App is a minimal four-function calculator, publisher is com.example, version 1.0.0.
The UI needs to display an input field, result text, number buttons, and +-*/=C buttons.
First, do Linux desktop smoke and PNG screenshot; I currently have no physical device, so the deploy record can use desktop-preview.
Finally, prepare upystore manual upload materials, but do not actually upload.
```

### Test and Run Commands

For Linux desktop smoke, it is recommended to use `mpos-test-app`:

```bash
cd <repo-root>
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-test-app/scripts/run_app_smoke.py \
  --repo <repo-root> \
  --app-fullname <fullname> \
  --generation-result <generation_result.json> \
  --screenshot
```

To open the full Linux SDL simulator:

```bash
cd <repo-root>
scripts/run_desktop.sh <fullname>
```

This opens the full Linux SDL simulator, suitable for manually checking if the UI is correct. Automated smoke should still use `/mpos-test-app` or `run_app_smoke.py`.

Before real device deployment, you must first confirm:

- Whether a physical device is available.
- The board model and serial port, e.g., ESP32-S3, `/dev/ttyACM0`.
- Whether the device already has MicroPythonOS installed; if not installed or unsure, first use `https://install.micropythonos.com/`.

There are two paths for device deployment:

- `device-copy`: Use `mpremote connect <port> fs cp -r <app_dir> :/apps/` to directly copy the App. Suitable when AIOREPL/`mpos_controller.py` probe fails but the file system is accessible.
- `mpk-install`: Upload the MPK and then call `AppManager.install_mpk()`. This is the preferred path for release verification, but requires the target device to be able to `import mpos`.

The essence of `device-copy` is "copy the App directory to the device file system"; it cannot prove that AppManager successfully installed the MPK. `mpk-install` is closer to the real installation path after release, but is more dependent on the MicroPythonOS runtime state on the device.

### Web Preview Status

Web preview is just an optional browser preview, not a default gate, and cannot replace Linux desktop smoke or real hardware verification.

Local Web command:

```bash
cd <repo-root>
scripts/build_mpos.sh web
scripts/run_web.sh
```

The current Web preview / Web build may encounter OS/Web port toolchain issues, such as `machine_timer_type` link errors, missing Emscripten/emsdk, or missing `web/micropython.wasm` or `web/micropython.data` artifacts. Such issues should be classified as MicroPythonOS Web target or toolchain problems. Do not let `mpos-gen-app` modify the target App, and do not misdiagnose them as missing regular Python dependencies.

Therefore, beginners are advised to run Linux desktop smoke first; Web preview should only be run when you explicitly want to see the browser effect or verify WebAssembly behavior.

### Error Feedback and Step-by-Step Repair

LLM model capabilities vary. The first generation or repair may miss reading APIs, misdiagnose toolchain issues, or not cover all execution paths. When using, do not just tell the AI "it failed". Return the complete error and artifact paths to the AI, and let it repair step by step by stage.

Recommended feedback format:

```text
I ran <command> in <repo-root> and it failed.
Target App: <fullname>
Failed stage: analysis / generation / test / package / deploy / publish
returncode: <code>
Key stdout/stderr/traceback:
<Paste the last 100-200 lines>
Related artifacts:
- tmp/mpos-plan-app/<fullname>/plan_state.json
- tmp/mpos-plan-app/<fullname>/generation_result.json
- tmp/mpos-test-app/<fullname>/app_test_result.json
- tmp/mpos-package-app/<fullname>/package_result.json
- tmp/mpos-deploy-app/<fullname>/deploy_result.json
Please only repair the target App or the corresponding skill artifact, do not modify MicroPythonOS OS/framework/build code.
```

Common handling principles:

- App's own traceback, missing API, UI crash: Return to `/mpos-gen-app` for repair.
- Missing `uv`, `ruff`, `mpy-cross`, Emscripten, desktop binary: Record as toolchain issues, return the complete installation/build error to the AI for judgment on the next step.
- `deploy_result.json` is failed: First check if it's a `device-copy` failure, `mpk-install` probe failure, or if the user confirmed no hardware and should use `desktop-preview`/`web-preview` record.
- Screenshot upload only supports PNG, JPEG, WebP; BMP is just a raw smoke artifact and cannot be used as an upystore release screenshot.

If the AI repairs once and fails again, paste the new error back into the same stage. Do not manually modify the code extensively. The design of mpos skills is a cycle of "generate → test → repair with logs → test again".

---

## One-Sentence Hardware Generation — AI Embedded Code Generation Pipeline

The user only needs to describe the requirements in natural language ("Make a temperature and humidity monitor, buzzer alarm when threshold exceeded"), and the system automatically completes the entire process from selection, code generation, PC simulation, flashing to error repair.

### Pipeline Overview

```
User says one sentence
    ↓
Phase 1: upy-analyze    → Requirements parsing + Driver search
Phase 2: upy-select-hw  → MCU selection + Pin assignment + BOM
Phase 3: upy-scaffold   → Project skeleton generation
Phase 4: upy-generate   → Business code generation
Phase 4.5: upy-simulate → PC-side full-process simulation (no hardware required)
Phase 5: upy-deploy     → One-click flashing and running
Phase 6: upy-autofix    → Error hierarchical decision-making + Delegated repair
Phase 7: upy-wiring     → Wiring diagram generation
       upy-diagram      → Architecture diagram + Flowchart
Exception path: upy-gen-driver → Uncommon hardware driver generation
```

### Skill List

| # | Skill | Phase | Status | Description |
|---|-------|-------|--------|-------------|
| 1 | `upy-analyze` | Phase 1 | Implemented | Natural language → Device list + Driver API reference |
| 2 | `upy-select-hw` | Phase 2 | Implemented | MCU selection + Firmware verification + Pin assignment + BOM |
| 3 | `upy-scaffold` | Phase 3 | Implemented | Generate firmware/ complete skeleton (Timer/asyncio/Thread) |
| 4 | `upy-generate` | Phase 4 | Implemented | Driver download + DI architecture business code + Mock + unittest |
| 5 | `upy-simulate` | Phase 4.5 | Implemented | PC-side CLI+rich full-process simulation (data generator + multiple scenarios) |
| 6 | `upy-deploy` | Phase 5 | Implemented | mpremote upload + Flashing + Persistent session + Initial PASS/FAIL judgment |
| 7 | `upy-autofix` | Phase 6 | Implemented | Orchestration coordination layer: triage.py collection → LLM hierarchical decision → Delegate to upstream skill |
| 8 | `upy-wiring` | Phase 7 | Implemented | Wiring diagram generation (Mermaid .md + SVG + PNG + HTML) |
| 9 | `upy-diagram` | Phase 7 | Implemented | Architecture diagram + Flowchart + Data flow diagram (Mermaid .md + SVG + PNG + HTML) |
| 10 | `upy-gen-driver` | Exception path | Implemented | PDF/Arduino → Debug version driver → Hardware verification loop → Normalized MPY driver |

**Supporting infrastructure:**
- `upy-project-gen-toolchain-spec` — Overall architecture documentation + manifest/schema definitions
- `upy-pkg-guide` — Device driver usage query (called by upy-analyze)
- `fetch-doc` — URL content fetching (called by upy-pkg-guide)

### Introduction to Each Skill

#### `/upy-analyze` — Requirements Parsing + Driver Search

Input user natural language description, LLM decomposes intent → Multi-keyword parallel search upypi + awesome-micropython → Extract driver API reference → Output `project-manifest.json` (phase: analyze).

#### `/upy-select-hw` — MCU Selection + Pin Assignment

Read manifest → Recommend MCU based on scenario/power consumption/network requirements → I2C address conflict detection → Generate pin assignment table (including electrical type enumeration + physical pin number) → Output BOM bill of materials.

#### `/upy-scaffold` — Project Skeleton Generation

Read manifest → AskUserQuestion to select scheduling mode (Timer/asyncio/_thread) and optional modules → Call `init_scaffold.py` to generate complete `firmware/` skeleton (board.py, conf.py, boot.py, main.py, drivers/*, tasks/*, lib/*, tools/*).

#### `/upy-generate` — Business Code Generation

Read firmware/ skeleton + Driver API reference → Download driver → upy-norm-driver normalization → Generate DI architecture task code + conf.py + main.py + Mock layer + unittest → black + flake8 + pylint validation.

#### `/upy-simulate` — PC-side Full-Process Simulation

LLM reads all firmware/ code → Self-design: scheduling scheme + data generator `gen_xxx(tick)` + visualization (CLI+rich preferred) + multi-scenario coverage → Generate `test/pc/sim_main.py` → flake8 + pylint validation → Run. **No real hardware required to verify business logic.**

#### `/upy-deploy` — One-Click Flashing and Running

mpremote upload firmware/ → Verify file integrity → Soft reset + Reconnect wait → Persistent session output collection → Device-side log capture → Local rule initial PASS/FAIL judgment.

#### `/upy-autofix` — Orchestration Coordination Layer

Automatically enters after deploy failure. `triage.py` collects structured data (error parsing + I2C hardware detection + git management) → LLM reads JSON + raw logs → Hierarchical decision (P0~P3) → Delegate to upstream skill for repair (generate/select-hw/analyze) → Optional PC verification → Redeploy. Maximum 3 attempts.

#### `/upy-wiring` — Wiring Diagram Generation

Read all .py source code in firmware/ to extract actual pins/addresses/buses → Cross-validate with manifest → LLM generates intermediate JSON → Script renders Mermaid wiring diagram .md + SVG + PNG + self-contained HTML (double-click browser to view) + Pin cross-reference table.

#### `/upy-diagram` — Architecture Diagram + Flowchart

Scan firmware/ code structure + manifest → LLM generates intermediate JSON → Script renders Mermaid architecture diagram + flowchart + data flow diagram, each output .md + SVG + PNG + self-contained HTML (Tabs to switch between diagram/source code, dark mode adaptive). Supports simple/medium/detailed three levels of complexity.

#### `/upy-gen-driver` — Driver Code Generation (Exception Path)

Triggered when no driver is found on upypi + GitHub. Extract information from PDF datasheet or Arduino code → LLM generates debug version single-file driver (including full self-check logic) → `mpremote resume run` hardware verification loop (maximum 10 rounds) → Remove debug → `upy-norm-driver` normalization. Can be called by `upy-analyze`, `upy-autofix`, or the user directly.

---

### `/upy-norm-driver` — Driver File Normalization

**Purpose**: Rewrite a usable but non-standard MicroPython driver `.py` file (not `main.py`) according to the GraftSense specification, outputting the complete normalized file.

**Input**: Path to an existing driver `.py` file

**Output**: Complete normalized `.py` file + Rewrite description table

**Coverage rules**: P0 mandatory 38 items, P2 optional 7 items, including:

| Category | Main Rewrite Items |
|---|---|
| File structure | 7-line file header comment, 4 module global variables, 6 section markers, section content specification |
| Class design | Class structure layout, `__slots__` optimization, avoid multiple inheritance, explicit dependency injection, constant specification |
| docstring | Class-level bilingual Chinese/English (including Attributes/Methods/Notes), method-level bilingual Chinese/English, ISR-safe annotation, side-effect annotation |
| Type annotations | `__init__` parameter annotations, public method return value annotations, callback using `callable` |
| Parameter validation | Three modes: `isinstance`/`hasattr`/value range, `__init__` two-step validation |
| Exception handling | Exception type normalization, `OSError` wrapping re-raise (preserve `from e`), retry mechanism |
| ISR specification | Prohibit memory allocation/blocking IO/raising exceptions, `micropython.schedule`, concurrency protection |
| Function design | Naming conventions, return value design, `debug` log switch |

**Core constraint**: Do not modify external API names, method signature semantics, business logic, or hardware communication timing.

**Usage example**:
```
/upy-norm-driver sensors/bh1750_driver/code/bh_1750.py
```

---

### `/upy-norm-main` — Test File Normalization

**Purpose**: Rewrite an existing `main.py` test file according to the specification without changing the test logic.

**Input**: Path to an existing `main.py` file

**Output**: Complete normalized `main.py`

**P0 mandatory items (10 items)**:

| # | Rewrite Item |
|---|---|
| 1 | 7-line file header comment |
| 2 | 6 section marker comments (correct order) |
| 3 | Initialization configuration area must have `time.sleep(3)` |
| 4 | Initialization configuration area must have `print("FreakStudio: ...")` |
| 5 | Global variable area prohibits instantiation, move to initialization configuration area |
| 6 | `while` loop only allowed in the main program area |
| 7 | `raise`/`print` strings all in English |
| 8 | Main program area wrapped with `try/except KeyboardInterrupt/OSError/Exception/finally` |
| 9 | In `finally`, call `close()`/`deinit()`, `del` hardware objects, print exit prompt |
| 10 | Inline comments changed to Chinese |

**P1 try to change**: High-frequency function comments with default calls (for REPL manual invocation), three types of test scenario coverage check.

**Usage example**:
```
/upy-norm-main sensors/bh1750_driver/main.py
```

---

### `/upy-gen-main` — Generate Test File from Scratch

**Purpose**: Given a driver `.py` file, analyze all its public APIs, and generate a complete `main.py` from scratch that conforms to the specification.

**Input**: Path to driver `.py` file

**Output**: Complete `main.py` + API coverage description

**Full coverage principle**:

Classify all APIs by chip type functional dimensions:

| Chip Type | Coverage Dimensions |
|---|---|
| Sensor type | Basic status query, core data acquisition, parameter configuration, mode switching, calibration/compensation |
| Motor driver type | Hardware initialization, motion control, status reading, reset/sleep |
| Communication module type | Network/protocol configuration, data send/receive, status query, power control |
| Storage chip type | Data read/write, address configuration, erase/reset |
| GPIO/Bus expander type | Pin configuration, level read/write, interrupt configuration |

Cover three types of test scenarios: normal parameters, boundary parameters (hardware limit values), abnormal parameters (verify exceptions are correctly raised).

API handling method: Low-frequency APIs execute automatically, high-frequency/mode-switching APIs are commented out for invocation (for REPL manual triggering).

**Usage example**:
```
/upy-gen-main sensors/bh1750_driver/code/bh_1750.py
```

---

### `/upy-gen-readme` — Generate README from Scratch

**Purpose**: Given a driver `.py` file, analyze its functionality and APIs, and generate a complete `README.md` from scratch.

**Input**: Path to driver `.py` file (optional: existing README as reference)

**Output**: Complete `README.md`

**13 mandatory chapters**:

| # | Chapter | Content |
|---|---|---|
| 1 | Title | `# [Chip Name] MicroPython Driver` |
| 2 | Table of Contents | Anchor links to all chapters |
| 3 | Introduction | Driver purpose, functionality, applicable scenarios |
| 4 | Key Features | List of feature highlights |
| 5 | Hardware Requirements | Recommended hardware + Pin description table |
| 6 | Software Environment | Firmware version, dependency libraries |
| 7 | File Structure | File tree (`├──` format) |
| 8 | File Description | Explain each file's purpose one by one |
| 9 | Quick Start | Step-by-step instructions + Minimal runnable code example |
| 10 | Notes | Operating conditions, limitations, compatibility |
| 11 | Version History | Table: Version/Date/Author/Change Description |
| 12 | Contact | Email + GitHub |
| 13 | License | MIT License |

**Usage example**:
```
/upy-gen-readme sensors/bh1750_driver/code/bh_1750.py
```

---

### `/upy-norm-pkg` — Driver Package Full-Process Normalization

**Purpose**: For an existing verified driver file, perform the complete normalization process on the entire driver package directory as an Orchestrator Skill.

**Input**: Path to driver package directory

**Output**: Complete normalized driver package (all driver files + main.py + README.md + package.json + standard directory structure)

**Execution flow (6 steps)**:

| Step | Operation |
|---|---|
| 0 | Scan directory, classify driver files and `main.py`; list multiple driver files and ask user to confirm scope |
| 1 | Execute `/upy-norm-driver` for each driver file sequentially, pause for confirmation after each file |
| 2 | Execute `/upy-norm-main` (if `main.py` exists) or `/upy-gen-main` (if `main.py` does not exist) |
| 3 | Execute `/upy-gen-readme` |
| 4 | Execute `/upy-gen-pkg` |
| 5 | Execute `/upy-pack-driver` |
| 6 | Execute `/upy-deploy-test` (upload to device and verify after user confirmation) |

**Key rules**: After each step, display `[Step X/6 — skill name: file name completed]`, pause and wait for user confirmation before continuing.

**Usage example**:
```
/upy-norm-pkg sensors/bh1750_driver/
```

---

### `/upy-deploy-test` — Device Deployment and Verification

**Purpose**: After `upy-norm-pkg` completes, upload the normalized driver files and `main.py` to the MicroPython device, run and verify the output.

**Input**: Path to normalized `code/` directory + User-confirmed COM port

**Output**: Upload progress + Verification report (success/failure + error analysis)

**Execution flow (6 steps)**:

| Step | Operation |
|---|---|
| 0 | Ask and confirm COM port (can execute `mpremote connect list` for assistance) |
| 1 | Scan files to upload (`.py` files + sub-package directories) |
| 2 | Upload files one by one (`mpremote connect <COM> resume fs cp`) |
| 3 | Verify device file integrity (`fs ls`) |
| 4 | Run `main.py` (`mpremote resume run main.py`) |
| 5 | Analyze output, output verification report |

**Failure diagnosis**: `ImportError` → Missing file; `OSError -110` → I2C wiring; `RuntimeError: WiFi` → Check if credential placeholders have been replaced.

**mpremote reference**: `/mpremote-device-interaction`, `/mpremote-file-transfer`, `/mpremote-live-session`, [Official documentation](https://docs.micropython.org/en/latest/reference/mpremote.html)

**Usage example**:
```
/upy-deploy-test bh1750_driver/code/
```

---

### `/upy-opt-driver` — Performance Optimization

**Purpose**: For any MicroPython `.py` file (driver file, `main.py`, or other files), rewrite according to the GraftSense performance optimization guide, focusing on **execution speed** improvement.

**Input**: Path to driver `.py` file or directory path (supports batch optimization of multiple files)

**Output**: Optimized complete `.py` file + Optimization description table

**Optimization priority**:

| Priority | Item | Typical Speedup |
|---|---|---|
| P0 | Pre-allocated buffers | Eliminate GC jitter |
| P0 | `memoryview` slicing | Zero copy (> 32 bytes) |
| P0 | Cache object references | 5–20% (loops > 100 iterations) |
| P0 | `const()` constants | Zero overhead |
| P1 | Manual GC control | Controllable latency |
| P1 | `@native` decorator | ~2x |
| P1 | `@viper` decorator | ~58x (integer operations) |
| P1 | Integer instead of float | ~57% (chips without FPU) |
| P2 | `viper ptr8/ptr16/ptr32` | ~23x (large loop traversal) |
| P2 | SIO register direct write | ~48% (RP2040 specific) |
| P2 | `array` instead of `list` | Contiguous memory |

**Core constraints**: `@viper` rewrites must annotate integer overflow risk; `@native` must annotate limitations (no generators/keyword arguments); SIO registers must annotate "RP2040 specific".

**Usage example**:
```
/upy-opt-driver sensors/bh1750_driver/code/bh_1750.py
/upy-opt-driver sensors/bh1750_driver/code/
```

---

### `/upy-slim-driver` — Memory Optimization

**Purpose**: For any MicroPython `.py` file (driver file, `main.py`, or other files), rewrite according to the GraftSense memory minimization guide, focusing on **RAM usage** reduction.

**Input**: Path to driver `.py` file or directory path (supports batch optimization of multiple files)

**Output**: Optimized complete `.py` file + Optimization description table

**Optimization priority**:

| Priority | Item | Typical Savings |
|---|---|---|
| P0 | Pre-allocated buffers | Eliminate peak heap allocation |
| P0 | Private `_CONST` | ~40 bytes/constant |
| P0 | Avoid string `+` in loops | Eliminate temporary objects |
| P0 | `bytes`/`bytearray` instead of `list` | ~90% (register tables) |
| P1 | `gc.collect()` pre-positioning | Reduce randomness |
| P1 | `gc.disable()`/`gc.enable()` | Prevent GC interruption |
| P1 | `struct.pack_into()` | Eliminate temporary bytes |
| P2 | `__slots__` | 50–200 bytes/instance |
| P2 | Generator instead of list | Peak RAM O(N)→O(1) |

**Core constraints**: `_CONST` rewrite only applies to module-internal constants; `gc.disable()` intervals must be short and bounded, must not contain blocking I/O; overlaps with `upy-opt-driver`'s P0#1 (pre-allocated buffers), do not execute repeatedly.

**Usage example**:
```
/upy-slim-driver sensors/bh1750_driver/code/bh_1750.py
/upy-slim-driver sensors/bh1750_driver/code/
```

---

### `/upy-pack-driver` — Package into Standard Directory Structure

**Purpose**: After other Skills have completed, organize the driver file, `main.py`, `README.md`, and `package.json` into a standard driver package directory structure, and generate a `LICENSE` file.

**Input**: Path to driver `.py` file (the same directory must already contain `main.py`, `README.md`, `package.json`)

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

**Core constraint**: Does not generate any content, only responsible for organizing files; missing files will prompt to run the corresponding Skill first.

**Usage example**:
```
/upy-pack-driver bmp280.py
```

---

### `/upy-pkg-guide` — Device Driver Usage Query

**Purpose**: Given a device name, automatically fetch all files of the corresponding driver package from upypi, comprehensively analyze and output usage points.

**Input**: Device/chip name (e.g., BMP280, DS18B20, MPR121)

**Output**: Package information, installation command, initialization example, core API table, notes

**Execution flow**: curl search upypi → Get package.json → Parallel download driver.py + main.py + README.md → Comprehensive output

**Usage example**:
```
/upy-pkg-guide BMP280
/upy-pkg-guide DS18B20
```

---

### `/fetch-doc` — URL Content Fetching

**Purpose**: Given any URL, automatically fetch the content and extract key information. Supports GitHub files, upypi package pages, and regular web pages.

**Input**: URL (GitHub blob links are automatically converted to raw URLs)

**Output**: Extract key information based on content type (README summary, driver API table, package.json fields, etc.)

**Dependencies**: Requires Python + requests library (`pip install requests`)

**Usage example**:
```
/fetch-doc https://github.com/FreakStudioCN/GraftSense-Drivers-MicroPython/blob/main/sensors/bmp280_driver/README.md
```

---

### `/review` — MicroPython Code Review

**Purpose**: Based on MicroPython maintainer historical review patterns (~19.5K classified review comments), perform AI-assisted review of MPY driver code.

**Input**: MicroPython code changes (branch, commit, diff, PR)

**Output**: Semantically searched matching historical review patterns + Review context suggestions

**Core capabilities**:
- Semantic search of ~19.5K classified review comments to find relevant historical review patterns
- Supports MCP server (`review_diff`, `search_reviews` and other tools) and CLI methods
- MCP server keeps embedding model warm, eliminating 2-3s cold start per query

**Usage example**:
```
/review Review the diff of the current branch against main
/review Check sensors/bmp280_driver/code/bmp280.py
```

---

### `/upy-project` — MicroPython Project End-to-End Generation

**Purpose**: User describes project requirements, automatically completes the full process from requirements clarification, device selection, code generation to device debugging.

**Input**: Project description (main controller model, sensor list, functional requirements, serial port)

**Output**: Complete project code (`xx_task.py` + `main.py`) + mpremote automatic debugging

**Execution flow (5 phases)**:

| Phase | Operation |
|---|---|
| Pre-check | Verify mpremote availability |
| Phase 0 | Parse GitHub links in user input (call fetch-doc skill) |
| Phase 1 | List all missing information at once, do not ask multiple rounds |
| Phase 2 | Select devices from upypi, call upy-pkg-guide to get API usage |
| Phase 3 | Generate task file + main.py (unified scheduling) |
| Phase 4 | mpremote automatic debugging, maximum 3 times, parse output and fix |

**Code structure**:
```
/lib/<driver>.py       ← Downloaded from upypi
<function>_task.py     ← Single function module (contains init() + run())
main.py                ← Unified scheduling
```

**Usage example**:
```
/upy-project Use ESP32 and BMP280 to make a temperature monitor, print every 5 seconds, COM3 port
```

---

### `/upy-gen-pkg` — Generate package.json from Scratch

**Purpose**: Given a driver directory or driver file, analyze the structure and dependencies, and generate a compliant `package.json` from scratch.

**Input**: Path to driver directory or driver `.py` file

**Output**: Complete `package.json` + Three installation method commands

**Three-step dependency processing priority**:

```
1. MicroPython built-in modules (machine, time, sys, etc.) → Do not write into deps
2. micropython-lib standard library → Use mip standard format
3. Other third-party dependencies → Query https://upypi.net/api/search?q={dependency name}
   If result exists → Use upypi URL to write into deps
   If no result → Use github: placeholder format, annotate ⚠️ requires manual confirmation
```

**Usage example**:
```
/upy-gen-pkg sensors/bh1750_driver/
```

---

### `/mpremote-device-interaction` — Device Connection and Status Query

**Purpose**: Connect to a MicroPython device via mpremote, execute code, query device status (memory, firmware version, file list, etc.).

**Platform support**: Windows (COMn), macOS (/dev/tty.usbmodem*), Linux (mpy-dev or /dev/serial/by-id/)

**Core principle**: Connecting to a running device must use `resume`, otherwise a soft reset will interrupt the program.

**Covered scenarios**:

| Scenario | Command Example |
|---|---|
| List available devices | `mpremote connect list` |
| Windows connection | `mpremote c3 resume` / `mpremote connect COM3 resume` |
| macOS connection | `mpremote connect /dev/tty.usbmodem1101 resume` |
| Linux connection | `mpremote connect $(mpy-dev tty my-board) resume` |
| Query firmware version | `mpremote <device> resume exec "import sys; print(sys.version)"` |
| Query free memory | `mpremote <device> resume exec "import gc; gc.collect(); print(gc.mem_free())"` |
| Soft reset | `mpremote <device> soft-reset` |

**Usage example**:
```
/mpremote-device-interaction  Connect to COM3, check firmware version and free memory
```

---

### `/mpremote-file-transfer` — Device File Transfer

**Purpose**: Use mpremote to copy files between local and device, manage the device file system (ls, mkdir, rm, tree).

**Platform support**: Windows, macOS, Linux. Device path conventions for each platform are detailed within the Skill.

**Key rule**: File operations must include `resume`, otherwise the device will be soft-reset before each operation.

**Covered scenarios**:

| Scenario | Command Example |
|---|---|
| Upload file | `mpremote <device> resume fs cp main.py :main.py` |
| Download file | `mpremote <device> resume fs cp :main.py .` |
| Recursive directory sync | `mpremote <device> resume fs cp -r utils/ :utils/` |
| Restart after driver update | `mpremote <device> resume fs cp driver.py :driver.py + soft-reset repl` |
| List files | `mpremote <device> resume fs ls :` |
| Check storage space | `mpremote <device> resume exec "import os; print(os.statvfs('/'))"` |

**Usage example**:
```
/mpremote-file-transfer  Sync the local utils/ directory to the device, then restart and monitor
```

---

### `/mpremote-live-session` — Persistent Connection and Output Monitoring

**Purpose**: Establish a persistent connection to the device, continuously send commands and capture output. Suitable for running asyncio devices, stress testing, long-term monitoring.

**Platform support**: Linux/macOS use PTY solution; Windows uses subprocess pipe alternative (has limitations, see Skill documentation).

**Core principle**: Repeatedly calling `mpremote resume exec` will send Ctrl+C to asyncio devices, killing the event loop; a persistent session must be used instead.

**When to use**:

| Scenario | Recommended Solution |
|---|---|
| Single quick query | `mpremote <device> resume exec "..."` |
| Multi-command sequence / Monitor output | This Skill (persistent session) |
| Device running asyncio/aiorepl | This Skill (mandatory) |
| File copy | mpremote-file-transfer |

**Usage example**:
```
/mpremote-live-session  Establish a persistent connection to /dev/tty.usbmodem1101, query memory once per second and log to file
```

---

## How It Works

Each Skill is a `SKILL.md` file containing:

- **Role positioning**: Tells the AI what role to play
- **Core constraints**: Clearly states what cannot be modified
- **Rewrite priority table**: P0 mandatory / P2 optional, each item corresponds to a specific chapter in the specification document
- **Key specification summary**: Embeds the most important code templates to avoid consulting the full specification document each time

### Trigger Flow

```
User inputs /upy-norm-driver xxx.py
    ↓
Claude loads the specification summary and priority table from SKILL.md
    ↓
Reads the target file, analyzes structure (communication interface type, classes, methods, ISR callbacks, etc.)
    ↓
Rewrites item by item according to P0→P2 priority (does not change API and business logic)
    ↓
Outputs complete normalized file + rewrite description table
```

### Why Split into Multiple Skills

The specification document has 22 chapters and 2200+ lines. Embedding the entire specification in a single Skill would lead to excessive context length and reduced rewrite quality. By splitting according to "rewrite target" and "optimization goal", each Skill only embeds the specification summary for the corresponding chapters, keeping the context manageable.

**Skill classification**:
- **AI code generation pipeline** (10): `upy-analyze`, `upy-select-hw`, `upy-scaffold`, `upy-generate`, `upy-simulate`, `upy-deploy`, `upy-autofix`, `upy-wiring`, `upy-diagram`, `upy-gen-driver`
- **Code review**: `review` (mpy-review, MPY driver code review)
- **Normalization**: `upy-norm-driver`, `upy-norm-main`, `upy-norm-pkg` (Orchestrator)
- **Generation**: `upy-gen-main`, `upy-gen-readme`, `upy-gen-pkg`
- **Optimization**: `upy-opt-driver` (performance), `upy-slim-driver` (memory)
- **Packaging**: `upy-pack-driver`
- **Project generation**: `upy-project` (end-to-end)
- **Tools**: `upy-pkg-guide` (device usage), `fetch-doc` (URL content fetching)

---

## Specification Documents

Full specification: [upy_driver_dev_spec_summary.md](https://github.com/FreakStudioCN/GraftSense-Drivers-MicroPython/blob/main/upy_driver_dev_spec_summary.md)

---

## Version History

| Version | Date | Author | Description |
|---|---|---|---|
| v1.0.0 | 2026-04-24 | leezisheng | Initial version, contains 5 skills |
| v1.1.0 | 2026-04-26 | leezisheng | Added upy-pack-driver; upy-norm-driver supplemented 16a/16b/16c; unified license to MIT; I2C scan specification |
| v1.2.0 | 2026-04-27 | leezisheng | Added upy-norm-pkg (Orchestrator), upy-opt-driver (performance optimization), upy-slim-driver (memory optimization); improved multi-file batch processing mode |
| v1.3.0 | 2026-04-29 | leezisheng | Added upy-pkg-guide (device usage query), fetch-doc (URL content fetching), upy-project (project end-to-end generation); upy-gen-pkg query logic changed to Bash curl automatic execution |
| v1.4.0 | 2026-05-04 | leezisheng | Added mpremote-device-interaction, mpremote-file-transfer, mpremote-live-session; based on andrewleech/claude-mpy-marketplace architecture, supplemented Windows (COMn) and macOS platform support |
| v1.5.0 | 2026-05-14 | leezisheng | Added upy-deploy-test (device deployment and verification); upy-norm-pkg added step 6 calling upy-deploy-test; each skill added middleware library type judgment branch and sensitive data replacement rules |
| v1.6.0 | 2026-06-02 | leezisheng | Added "one-sentence hardware generation" AI code generation pipeline (10 skills): analyze/select-hw/scaffold/generate/simulate/deploy/autofix/wiring/diagram/cold-driver + overall architecture documentation. upy-simulate changed to CLI+rich priority. upy-select-hw added pin electrical type enumeration + physical pin rules. Skill count increased from 15 to 25. |
| v1.7.0 | 2026-06-03 | leezisheng | upy-cold-driver renamed to upy-gen-driver, positioned as an independently callable skill (not just an exception path). upy-gen-driver flow implemented: debug version driver → mpremote hardware verification loop → remove debug → normalization. upy-wiring + upy-diagram added HTML output (self-contained browser page, Mermaid.js CDN + Tab switching), --format all now outputs all four formats: md + svg + png + html. All 25 skills supplemented with .skillfish.json. |
| v1.7.1 | 2026-06-03 | leezisheng | README.md installation script supplemented with upy-deploy-test + review skill. Function planning.md fixed: Module 4 visualization scheme (Pillow→Mermaid), Module 7 gen-driver flow supplemented hardware verification loop, triage.py line count correction, project architecture script name refresh, /cold-driver→/gen-driver. |
| v1.8.0 | 2026-07-05 | leezisheng | README.md added current Skill / Plugin full overview, clarified plugin version 8 processes: `upy-analyze-plugin`, `upy-select-hw-plugin`, `upy-flash-mpy-firmware-plugin`, `upy-scaffold-plugin`, `upy-generate-plugin`, `upy-deploy-plugin`, `upy-wiring-plugin`, `upy-diagram-plugin`; supplemented `upy-gen-driver-plugin` as missing hardware driver branch, and distinguished plugin version Skill/plugin from Classic Skill. |
| v1.9.0 | 2026-07-20 | leezisheng | README.md added independent MicroPythonOS App / MPK / upystore chapter, supplemented 9 `mpos-*` skill responsibilities, Claude Code slash command usage, quick start for beginners, `cc-switch` DeepSeek model switching suggestion, internal artifact/plan_state logic, full API reading requirement, App-only modification boundary, Web preview known issues, error feedback template and step-by-step repair flow; installation script synchronized to include `mpos-*`, clarified that `mpos-debug-app` is no longer needed. |

---

## License

MIT License

Copyright (c) 2026 leezisheng

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
