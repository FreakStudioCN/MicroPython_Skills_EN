---
name: mpos-gen-app
description: 'Generate, update, and repeatedly repair MicroPythonOS App code after requirements are confirmed. Use after mpos-analyze-app and optionally mpos-prepare-deps to create or modify an internal_filesystem/apps package directory with root MANIFEST.JSON, root icon_64x64.png, assets/*.py entrypoints/dependencies, dependency adapters, and validation results. Always defaults to a two-phase flow: first produce a generation plan and ask for confirmation, then write files only after explicit user confirmation. Supports repeated calls for user feature changes and test-failure repair loops. Does not analyze vague requirements, prepare external dependencies, package MPK files, deploy devices, flash firmware, publish to upystore, or rebuild lvgl_micropython.'
---

# MicroPythonOS App Code Generation

## Role

Translate confirmed MicroPythonOS App requirements into code. A two-phase flow is mandatory by default:

1. **Confirmation Phase**: Read-only context, output a generation/update plan listing files to be created or modified, version strategy, dependency integration, icon plan, and validation commands, and ask for user confirmation. No files are modified in this phase.
2. **Execution Phase**: Only after explicit user confirmation of the plan, create, modify, or repair App files and run validation.

If the user provides natural language ideas directly without `mpos-analyze-app` results, first route to `mpos-analyze-app`. If external pure Python drivers are needed but no `mpos-prepare-deps` handoff exists, first route to `mpos-prepare-deps`.

## Unified Project Log

Every plan/create/update/repair output (`generation_result.json` or confirmation plan) must be recorded in the project state directory:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-plan-app/scripts/update_plan_state.py record \
  --repo <repo-root> \
  --fullname <fullname> \
  --skill mpos-gen-app \
  --phase generate \
  --result <planned|success|partial|failed|blocked> \
  --artifact generation_result=<generation_result.json> \
  --next-skill <handoff.next_skill-or-null> \
  --event "Generated, updated, or repaired App files"
```

When the user modifies requirements, do not decide to directly overwrite old artifacts yourself; first let `mpos-plan-app` list the artifacts that will become invalid and wait for user confirmation. The two-phase confirmation is still enforced; `mpos-plan-app` cannot confirm file writing on behalf of the user.

## Required Context

Before generating or modifying, first load `mpos-dev` and read as needed:

- App/Activity/Service/Intent: `mpos-dev/reference/docs-app-model.md`
- System manager, TaskManager, DownloadManager, WebServer, Service: `mpos-dev/reference/docs-frameworks.md`
- Packaging and manifest validation: `mpos-dev/reference/docs-packaging.md`
- MPOS API precise index: `mpos-dev/reference/mpos_api_summary.json`
- LVGL API precise index: `mpos-dev/reference/lvgl_api_summary.json`
- Upstream analysis template: `mpos-analyze-app/templates/analysis_result.json`
- Dependency handoff template: `mpos-prepare-deps/templates/dependency_handoff.json`

Local facts take precedence:

- `<repo-root>/AGENTS.md`
- `<repo-root>/tests/test_apps_manifest.py`
- `<repo-root>/internal_filesystem/lib/mpos/content/app_manager.py`

## Modes

### plan

Default mode. Suitable for creating a new App, modifying features, or as the first step in a repair request after test failure.

Must output:

- Target App: `fullname`, `name`, `category`, `version`.
- Operation type: `create`, `update`, or `repair`.
- File plan: files to create/modify/keep.
- Activity/Service plan.
- Dependency plan: whether to consume `mpos-prepare-deps`, whether to generate a synchronous adaptation layer.
- Icon plan: use `scripts/generate_icon.py` to generate root `icon_64x64.png` based on the user's feature description.
- Version strategy: new App `1.0.0`; feature modification bumps patch; test failure repair does not bump.
- Validation plan: list the gates to run during the execution phase.
- Questions requiring confirmation; even without blocking issues, must ask "I will write files after your confirmation."

### create

Create a new App after user confirmation. The structure must be:

```text
internal_filesystem/apps/<fullname>/
  MANIFEST.JSON
  icon_64x64.png
  assets/<entrypoint>.py
```

`MANIFEST.JSON` uses full objects, not string-type activities:

```json
{
  "classname": "ExampleActivity",
  "entrypoint": "assets/main.py",
  "intent_filters": [{"action": "main", "category": "launcher"}]
}
```

### update

When the user modifies features, first read the existing App's manifest, entrypoint, assets, and related tests, then make minimal changes. Do not rewrite unrelated files, do not overwrite existing user resources. Feature modifications bump patch by default, e.g., `1.0.0` -> `1.0.1`.

If the user proposes adding new hardware, external drivers, or protocol libraries without a dependency handoff, stop and route to `mpos-analyze-app` or `mpos-prepare-deps`.

### repair

When tests fail or the user reports runtime failure, read the failure logs, command output, traceback, the last `generation_result.json`, and related source code. Only repair App files generated by this skill or explicitly involved in the current round.

Allows unlimited automatic repair of files generated/modified by this skill: each round makes a minimal patch, re-runs the failed gates and necessary full gates until passing or encountering an external blocker. External blockers include missing user requirements, missing dependency handoff, uninstalled tools, changed hardware facts, or failures originating from non-this-App files.

Repair does not bump version unless the user explicitly requests the fix as a release version.

## Requirements Confirmation

Before writing files, these items must be confirmed:

- Whether `fullname` is acceptable, and whether the directory is `internal_filesystem/apps/<fullname>`.
- App visible behavior and MVP scope.
- Number of Activities/Services and entry point files.
- Whether to integrate runtime files, imports, and adapter requirements from `mpos-prepare-deps`.
- Whether to generate the icon automatically based on the feature description.
- Whether the version strategy is acceptable.
- Whether the validation scope is acceptable.

Provide default values for missing non-blocking fields, but still include the default values in the confirmation plan. Do not write files without user confirmation.

## Dependency Integration

If there is a `mpos-prepare-deps` handoff:

- Place runtime files at the handoff's `target_path`, typically `assets/<module>.py` or `assets/<package>/...`.
- If the handoff has a `staged_path`, copy from the staged cache to the App directory.
- For dependencies with `async_compatible=true`, use them directly according to `imports[]`.
- For dependencies with `sync_needs_adapter=true`, a `assets/<name>_adapter.py` or equivalent adaptation layer must be generated, and the handoff's `adapter_requirements[]` must be implemented item by item.
- If `requires_vendor_path_injection=true`, only add minimal path injection at the beginning of the entrypoint:

```python
import sys

_VENDOR_DIR = sys.path[0] + "/vendor"
if _VENDOR_DIR not in sys.path:
    sys.path.insert(0, _VENDOR_DIR)
```

Do not place synchronous libraries directly into `async def` or LVGL event callbacks to block execution. Use `TaskManager.create_task`, `TaskManager.sleep_ms`, short-cycle state machines, timeouts, and cancellation paths.

## Icon Generation

For new Apps or missing icons, use this skill's script to generate the icon:

```bash
python3 /home/leeqingshui/MicroPython_Skills/mpos-gen-app/scripts/generate_icon.py \
  --prompt "<user feature description>" \
  --label "<App name>" \
  --output internal_filesystem/apps/<fullname>/icon_64x64.png
```

The script only uses Python standard library to generate a 64x64 PNG, no Pillow dependency. The icon should be based on simple symbols derived from feature keywords; if keywords are ambiguous, use the first letter of the App name.

## Code Rules

- New code should prefer imports from the root `mpos` module: `from mpos import Activity, TaskManager, SharedPreferences`.
- UI code must `import lvgl as lv` and follow `mpos-dev`'s LVGL rules.
- Do not hardcode screen resolution; use `lv.pct(100)`, flex, align.
- New labels should immediately `set_text("")` or set the final text.
- After `lv.style_t()`, must call `init()` before setters.
- Event callbacks accept an event parameter; use `obj.add_event_cb(callback, lv.EVENT.CLICKED, None)`.
- Do not arbitrarily assign Python attributes to LVGL objects; use closures, dicts, or parallel lists to store state.
- Use `SharedPreferences(self.appFullName)` for persistence.
- Do not write real API keys, tokens, passwords, or Bearer tokens into code, manifests, tests, logs, or JSON.
- Do not use CPython-only runtime modules: `typing`, `dataclasses`, `pathlib`, `logging`, `requests`, `subprocess`, `multiprocessing`.
- When asyncio is needed, prefer `uasyncio`; CPython fallback is only allowed for test compatibility mode:

```python
try:
    import uasyncio as asyncio
except ImportError:
    import asyncio
```

## Validation Gates

After each file write during the execution phase, these gates must be run and recorded. Commands are executed from the `<repo-root>` repository root.

1. App manifest validation:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python -m unittest tests/test_apps_manifest.py
```

2. CPython and MicroPython syntax check:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-gen-app/scripts/check_app_syntax.py \
  --repo <repo-root> \
  --app-fullname <fullname>
```

3. MicroPython import risk check:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-gen-app/scripts/check_app_mpy_imports.py \
  --app-dir internal_filesystem/apps/<fullname>
```

4. Project lint:

```bash
make lint
```

5. flake8:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python -m flake8 \
  --config /home/leeqingshui/MicroPython_Skills/mpos-gen-app/templates/flake8-mpos-app.ini \
  internal_filesystem/apps/<fullname>
```

Use the fixed template `templates/flake8-mpos-app.ini`. This template is calibrated against the current baseline of all real Apps: only selects `E9,F63,F7,F82`, supplements MicroPython/native/viper/RP2 PIO instruction built-in names, does not globally ignore `F821`; only applies file-level `F821` ignore for `rp2_*.py`, `*_pio.py` to prevent PIO assembly pseudo-operands from polluting normal Python checks. If undefined names appear in the newly generated App, fix the code, do not temporarily relax the template.

6. pylint. Use the fixed MicroPython-aware rcfile, do not modify the repository configuration:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python -m pylint \
  --persistent=n \
  --rcfile=/home/leeqingshui/MicroPython_Skills/mpos-gen-app/templates/pylintrc-mpos-app \
  internal_filesystem/apps/<fullname>/assets
```

Use the fixed template `templates/pylintrc-mpos-app`. This template is calibrated against the current baseline of all real Apps: ignores MicroPython/MPOS imports, LVGL dynamic members, docstrings, naming, and historical style noise; retains fatal/error/usage class issues, such as `undefined-variable`, `used-before-assignment`, `function-redefined`, `no-method-argument`. Do not add common variable names like `x`, `y`, `pin` to global builtins; if an RP2 PIO helper is genuinely generated, only allow a local `# pylint: disable=undefined-variable` declaration at the top of that helper file and record the reason in `generation_result.validation.warnings`. Pylint exit code is a bitmask: fatal(1), error(2), usage(32) are hard failures; warning(4), refactor(8), convention(16) are only recorded as warnings unless the user requests strict mode.

7. Clean cache artifacts:

```bash
find internal_filesystem/apps/<fullname> -name __pycache__ -o -name '*.pyc' -print
```

If there is output, delete and re-run relevant gates. Do not write `__pycache__/` or `.pyc` into the handoff JSON.

## Output JSON

At the end of the execution phase, output and optionally save `generation_result.json`. Refer to the structure in `templates/generation_result.json`, and validate with the script:

```bash
python3 /home/leeqingshui/MicroPython_Skills/mpos-gen-app/scripts/validate_generation_result.py \
  /path/to/generation_result.json
```

A successful `create/update/repair` must record:

- `confirmed_by_user: true`
- Created/modified files
- Version change
- Icon generation result
- Dependency and sync adapter result
- All validation gates and their return codes
- `handoff.next_skill: "mpos-test-app"`

For the plan phase, `confirmed_by_user` must be `false`, and `handoff.next_skill` must still point to `mpos-gen-app`.

## Downstream

After successful code generation, hand off to `mpos-test-app`. If the user only requests code generation, `handoff.next_skill` can be set to `null`, but validation results must still be reported.
