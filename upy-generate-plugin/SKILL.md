---
name: upy-generate-plugin
description: Plugin-based MicroPython business code generation phase. Used after receiving a scaffold phase_complete with next_phase=upy-generate-plugin to generate driver dependencies, factory/mock, tasks, conf.py, main.py, tests, lint/check/git commit, and phase_complete; also used in mode=fix for minimal code repair after deploy/simulate/autofix or user feedback.
---

# upy-generate-plugin Plugin Workflow

`upy-generate-plugin` is the business code generation phase of the MicroPython project pipeline. It consumes the `manifest_content` and project skeleton from `upy-scaffold-plugin`, generating complete firmware business code, tests, dependency files, and `phase_complete`. It must retain the embedded constraints of the old `upy-generate`, but replace direct reads/writes with the plugin protocol:

```text
file_operation(read/write)
script_run(...)
approval_request(...)
permission_request(...)
status_update(...)
phase_complete(...)
```

Formal pipeline:

```text
upy-analyze-plugin
-> upy-select-hw-plugin
-> upy-flash-mpy-firmware-plugin
-> upy-scaffold-plugin
-> upy-generate-plugin
-> upy-deploy-plugin or upy-simulate-plugin
```

Failure or feedback loop:

```text
deploy / simulate / test failure
-> upy-autofix-plugin
-> upy-generate-plugin(mode=fix)
```

If `upy-autofix-plugin` is not yet implemented, allow:

```text
upy-deploy-plugin
-> user inputs symptoms/feedback
-> upy-generate-plugin(mode=fix, source=user_feedback_after_deploy)
-> upy-deploy-plugin
```

## Required Reading

Before starting full/fix generation, read these reference files in order. They are migrated from the key constraints and templates of the old `G:\MicroPython_Skills\upy-generate\SKILL.md` and take precedence over the summary in this file:

| Timing | Required File |
|---|---|
| Parsing protocol, writing `phase_complete`, interpreting JSON fields | `references/protocol_fields.md` |
| Before starting full/fix generation | `references/legacy_constraints.md` |
| Before generating driver factory/mock | `references/driver_factory_templates.md` |
| Before generating tasks and PC tests | `references/task_generation_rules.md` |
| Before generating device MicroPython unittest tests | `references/device_unittest_subset.md` |
| Before modifying `conf.py` or `main.py` | `references/main_conf_rules.md` |
| Before using MicroPython hardware/peripheral/port APIs | `knowledge/micropython_official_library_index.json` |
| Before needing LLM/ASR/TTS/IoT/MQTT/Webhook/Cloud APIs | `references/cloud_integrations.md` |
| Before running quality gates | `references/validation_gates.md` |
| Before outputting success and git commit | `references/final_review_checklist.md` |

## Start Message

Full mode:

```json
{
  "protocol_version": "1.0",
  "type": "start_phase",
  "phase": "upy-generate-plugin",
  "session_id": "uuid",
  "idempotency_key": "upy-generate-plugin:<session_id>:full:v1",
  "payload": {
    "mode": "full",
    "source_phase": "upy-scaffold-plugin",
    "source_phase_complete_path": "sessions/<session_id>/phase_complete.upy_scaffold_plugin.json",
    "next_phase_preference": "deploy",
    "runtime_context": {
      "artifact_root": ".",
      "artifact_root_mode": "cwd",
      "session_root": "sessions/<session_id>",
      "project_root": "sessions/<session_id>/project",
      "file_operation_root": "sessions/<session_id>/project",
      "resource_root": "upy-generate-plugin"
    },
    "capabilities": {
      "approval_request": true,
      "file_operation": true,
      "script_run": true,
      "git_operation": false,
      "checkpoint_resume": true,
      "cancellation": true
    }
  }
}
```

Fix mode:

```json
{
  "type": "start_phase",
  "phase": "upy-generate-plugin",
  "payload": {
    "mode": "fix",
    "source": "user_feedback_after_deploy",
    "error_context": {
      "user_feedback": "After power-on, the OLED does not display; serial only prints boot ok",
      "deploy_result_path": "sessions/<session_id>/phase_complete.upy_deploy_plugin.json",
      "serial_excerpt": "...",
      "previous_generate_commit": "abc123"
    }
  }
}
```

## Full Flow

1. Validate upstream `phase_complete(upy-scaffold-plugin)`: `result=success` and `next_phase=upy-generate-plugin`. During migration, direct testing may pass manifest directly, but the formal pipeline must not skip scaffold. Then run `scripts/check_session_state.py --session-dir <session_root> --project-dir <project_root>`. If a stale old generate record is found, archive/ignore the old `phase_complete.upy_generate_plugin.json` and `generate_phase_log.md`; do not treat it as the current success/resume state.
2. Read `payload.manifest_content`, `runtime_context.project_root`, `firmware/board.py`, `firmware/conf.py`, `firmware/main.py`, `.flake8`.
3. Before writing `generate_plan.json`, Mock, business framework, or business code, you must gate on the explicit driver workflow status in `manifest_content.devices[]`. If `driver.status`, `driver.driver_status`, or `device.driver_status` is `cold_driver_required`, `pending_validation`, `unverified`, `failed`, or an unknown value, you must output `partial`, `next_phase=null`, `structured_errors[]`, and guide the user to first run `upy-gen-driver-plugin` pipeline mode to generate `firmware/drivers/<driver_id>_driver/`. An explicit `ready` status only passes if it also has `driver.path` and `hardware_marker=SELF_TEST_PASS:<driver_id>:<scenario>`; ordinary `builtin_runtime`, `micropython_lib`, `upypi`, `awesome-micropython`, `github`, `none`, `manual` source-only dependencies without a workflow status continue with existing dependency resolution.
4. Before running, ask the user if they want to supplement device behavior. Only allow supplementing business behavior, thresholds, periods, state machines, logging, and simulation scenarios; adding new hardware or changing pins must fall back to analyze/select-hw/scaffold.
5. Write `project/generate_plan.json`, planning only, without writing runtime code. The plan must include scheduler_mode, driver adapters, tasks, config_constants, main_assembly, tests, resource_plan, cloud_integrations (if cloud APIs are needed). Voice, sensor, cloud API, state machine, or cross-tick business flows must also include `data_flow_contract[]`, and declare contract test coverage for each critical data flow. Then run `scripts/check_generate_plan.py --project-dir <project_root> --require-plan`; if it fails, stop at partial and do not proceed with large-scale code writing.
6. Resolve driver and middleware dependencies using English keywords. First run `scripts/resolve_upypi_packages.py` to enumerate `https://upypi.net/packages.json`, then search by English keywords, awesome-micropython, or GitHub fallback.
7. If the requirements involve LLM, ASR, TTS, vision, IoT/MQTT, Webhook, weather maps, object storage, third-party REST APIs, or any paid/credentialed cloud service, read `references/cloud_integrations.md` and `knowledge/cloud_service_catalog.json`, and initiate user confirmation: service provider, official documentation/console/pricing links, whether billing/tokens have been activated, whether API Keys are ready, and whether a gateway/proxy is needed. Do not write real tokens into the code.
8. Run `scripts/download_drivers.py`. The script only reads manifest/stdin and outputs JSON to stdout; it must not write directly to the project directory or modify `project-manifest.json` directly.
9. Convert each `files[]` entry from the script output into `file_operation(write)`, with the target path under `firmware/lib/...`.
10. Read `references/driver_factory_templates.md`, then read the driver source code, README, example, and package metadata to generate `firmware/drivers/<name>_driver/__init__.py` and `mock.py`. Mock method signatures must come from the driver source code.
11. Read `references/task_generation_rules.md` and generate tasks according to the scheduling mode selected by scaffold:
    - `timer`: periodic tick, avoid blocking, prefer `time_helper.timed_function`.
    - `async`: use `uasyncio`, prefer `timed_coro`, change blocking sleep to `await asyncio.sleep_ms`.
    - `thread`: use `_thread` worker, locks, and main loop heartbeat.
12. Reuse scaffold assets: `firmware/lib/logger`, `time_helper`, `maintenance`, `scheduler`. Do not regenerate the logging system.
13. Read `references/main_conf_rules.md`, update `firmware/conf.py`. All thresholds, periods, retries, and log configurations must be in conf, not hardcoded in task/main; cloud APIs can only write non-secret endpoints, model names, timeouts, retries, feature toggles, and secret names, not real keys.
14. When involving `machine`, `network`, `neopixel`, `esp32`, `rp2`, `bluetooth`, or other hardware/peripheral/port APIs, you must first check `knowledge/micropython_official_library_index.json` for the corresponding MicroPython official page, and record `module`, official `url`, and `reason` in `manifest_content.generate.doc_evidence[]`. Only CPython reference links or insufficient MicroPython page content are not sufficient evidence for peripheral implementation; you must supplement port documentation evidence or output partial.
15. Continue updating `firmware/main.py` according to `references/main_conf_rules.md`, retain the startup delay, install the rotating logger, complete the `machine -> factory -> driver -> task` DI assembly, perform an I2C scan at startup, and dual-write critical status to print and logger. Immediately after writing `conf.py/main.py`, run `scripts/check_conf_contract.py --project-dir <project_root>`.
16. Generate PC `unittest` and device MicroPython `unittest` tests. Before generating device-side tests, you must read `references/device_unittest_subset.md`; device-side tests are not CPython-only tests, nor should they be just import smoke tests. If device-side tests import `unittest`, you must declare `unittest` in `runtime_dependencies.mip` for the deploy phase to install; do not default to copying micropython-lib source into the project.
17. Read `references/validation_gates.md` and run the complete quality gate suite: `.pylintrc`, generate_plan, py_compile, conf_contract, driver compile, flake8, pylint, PC unittest, MicroPython import, dead config, task no-machine, device unittest subset, runtime dependencies, doc evidence, skeleton compliance, generated semantics, cloud integrations, session checkpoint.
18. Read `references/final_review_checklist.md`, perform the final review item by item, and output structured `review_findings`.
19. After drafting `phase_complete`, run `scripts/check_final_review_consistency.py` and `scripts/check_phase_complete_consistency.py --phase-complete <phase_complete> --project-dir <project_root>`; if they fail, change to `partial/failed`, `next_phase=null`, and record a structured error.
20. After checks and final review pass, initiate a git commit permission request. Both full and fix modes must commit after passing validation.
21. Output `phase_complete`, default `next_phase=upy-deploy-plugin`; the user may choose `upy-simulate-plugin` or `null`. If the cloud service is `mock_only` or `blocked`, do not enter deploy.
22. After success, ask if the user wants to generate additional artifacts: `upy-diagram-plugin` and `upy-wiring-plugin`. They can only go into `optional_next_phases` and must not override the main `next_phase`.

Additional hard rules:

- Run `check_generate_plan.py --require-plan` before broad code writes, and run `check_generate_plan.py --require-plan --check-files` after code writes. Planned task/driver/middleware/test paths that do not exist are blocking failures.
- For voice/sensor/cloud/state-machine flows, `generate_plan.json` must declare `data_flow_contract[]` with producer, consumer, invariant, storage when cross-stage, and test coverage. Prefer generated contract tests over trying to infer all business semantics from static AST checks.
- Never mark skipped pylint as success. If `.pylintrc` is missing, run `scripts/ensure_pylintrc.py`; then run pylint and record the real integer return code.
- `phase_complete.result=success` requires `file_manifest.files` to include both `project-manifest.json` role `manifest` and `generate_plan.json` role `plan`.
- `phase_complete.result=success` requires `session_state.upy_generate_plugin.json`, `checks.session_state_checkpoint.ok=true`, and an artifact entry with `type=session_state`.
- Write `session_state.upy_generate_plugin.json` only through `scripts/update_session_state.py`; do not hand-write a simplified JSON state. It must include `protocol_version`, `checkpoint`, `attempt`, `idempotency_key`, `manifest_hash`, `git_commit`, and `usage`.
- `manifest_hash` means the SHA256 of `project/project-manifest.json`, not the git commit. `session_state.git_commit` and `phase_complete.payload.generate.git.commit` must record final deliverable project HEAD. If `project-manifest.json` records an earlier code-generation commit, use `generate.git.code_commit` or include an explicit `commit_role`; do not imply it is final HEAD.
- `project-manifest.json` must advance consistently: `phase="generate"`, `domain_phase="generate"` when present, and `final_status="generated"` when present.
- `phase_complete.result=success` requires `payload.artifacts[]` to include both `type=session_state` and `type=file_manifest`.
- `phase_complete.result=success` requires `optional_next_phases` to offer `upy-diagram-plugin` and `upy-wiring-plugin`.
- `phase_complete.result=success` requires a completed git commit after clean gates. Commit denied, dry-run, not-a-git-repository, or commit skipped means `partial` with `next_phase=null`.
- `phase_complete.result=success` must not leave or commit CPython cache files. Run quality gates with bytecode disabled or temporary compile targets, remove project-local `__pycache__/` and `*.pyc` before git commit, and keep them out of `file_manifest` and artifacts.
- `phase_complete.result=success` must include `manifest_content.generate.runtime_dependencies` when generated firmware/device tests require mip packages; deploy installs them with `mpremote mip install`.
- `phase_complete.result=success` must include `manifest_content.generate.doc_evidence[]` for hardware/peripheral MicroPython APIs.
- `phase_complete.result=success` must include a production deploy plan: `manifest_content.generate.deploy_plan.source_only` contains `firmware/main.py`, `firmware/boot.py`, and `firmware/conf.py`; `upload_exclude` contains `firmware/drivers/**/mock.py` and `firmware/drivers/**/mock.mpy`. Driver mocks are test/support artifacts and must not be required by runtime firmware.
- Generate real device-side MicroPython unittest interface/contract tests under `device/tests/` by default. Keep `test/device/` only for legacy compatibility or existing project layout. Device tests should verify generated protocol/state/task/driver/config contracts, not only imports.
- Treat `NETWORK_DISCONNECTED`, `RATE_LIMITED`, and `UPSTREAM_TIMEOUT` as retryable interruption states. Treat `TOKEN_BUDGET_EXCEEDED`, `MODEL_CONTEXT_EXHAUSTED`, and `CANCELLED_BY_USER` as non-retryable unless the user changes budget/model/intent. Record them in `session_state.last_error` and structured errors.
- In async scheduler mode, do not call blocking driver/time operations directly inside `async def`: `time.sleep_ms`, `read_samples`, `play_samples`, `connect`, scan loops, or synchronous HTTP. Use cooperative state machines/adapters, thread mode, or emit `partial`.
- Do not hide blocking async calls with `getattr`, `__getattribute__`, alias variables, lambdas, reflection helpers, or thin sync wrapper functions. Yielding once before `record()`, `play()`, `connect()`, scan loops, or synchronous HTTP is not a non-blocking adapter. Use a real cooperative state machine, thread/worker handoff, genuinely non-blocking API, or emit `partial`.
- Use ASCII comments in generated firmware unless the project already requires non-ASCII. Avoid decorative box-drawing or mojibake separator comments in generated `.py` files.
- When `project-manifest.json` is `phase=scaffold` but a previous `phase_complete.upy_generate_plugin.json` says success, treat that previous generate event as stale/audit-only. If `generate_plan.json` or file_manifest paths are missing, start from scaffold input instead of resuming the stale generate output.
- If timer/scheduler assembly is generated or modified, read `knowledge/esp32_timer_scheduler_api.pitfall.json` before writing `firmware/main.py`. Do not rewrite scaffold-owned `firmware/lib/scheduler/timer_sched.py` just to solve port compatibility; inspect its API/defaults and keep its `timer_id=-1` default because RP2/Pico and Zephyr require virtual timers. Only RP2/Pico/RP2040/RP2350 and Zephyr should use `Timer(-1)` / `Scheduler(timer_id=-1)`. Other MCU/port targets must pass an explicit valid non-negative hardware timer id such as `Scheduler(timer_id=0, error_cb=...)`; do not generate implicit `Scheduler()` / `Scheduler(tick_ms=...)` when the scheduler default maps to `Timer(-1)`.
- Peripheral documentation evidence must be exact enough for the used API. `machine.Pin`, `machine.I2S`, `machine.Timer`, `neopixel`, `network.WLAN`, etc. must cite their corresponding MicroPython official page from `knowledge/micropython_official_library_index.json`; a parent `machine` page is not sufficient for a specific `machine.*` class when the index has a specific page.
- Do not edit scaffold-owned framework files such as `firmware/lib/logger/*`, `firmware/lib/scheduler/*`, `firmware/lib/time_helper.py`, or `firmware/tasks/maintenance.py` unless the user explicitly requests a scaffold library contract change. Fix generator code, entrypoint assembly, validation scripts, or deploy tooling instead.
- When generated `main.py` installs the scaffold rotating logger, do not modify scaffold logger source to add timestamps. Instead, generated calls must mix timestamp/uptime into message text at the call site, for example with `time.ticks_ms()`, `time.ticks_diff()`, `time.localtime()`, or an explicit helper. This preserves scaffold ownership while making `/log/run_*.log` useful after deploy.

## Pre-Run User Supplement

Initiate an optional approval/input request to collect:

- Sampling period, thresholds, alarm strategy, output actions.
- Network retry, offline cache, log level.
- OLED/UI, buzzer, relay, LED, etc. behavior.
- Whether the user prefers to enter deploy, simulate, or stop.
- Expected diagram/wiring artifacts.

Judgment rules:

| User Supplement Content | Handling |
|---|---|
| Business behavior, thresholds, periods, state machines | generate absorbs directly |
| Adding/replacing electronic modules | Fall back to analyze/select-hw |
| Changing pins, buses, power | Fall back to select-hw/scaffold |
| Only want to see business logic simulation | `next_phase=upy-simulate-plugin` |
| Only generate code | `next_phase=null`, with checkpoint |

## Dependency and Driver Rules

- All driver and middleware files must be written to `firmware/lib`, using POSIX relative paths for protocol paths.
- User Chinese requirements must first be converted to English keywords before searching. Example: `温湿度` -> `temperature humidity sensor`, `MQTT 上报` -> `mqtt publish client`.
- V0 may call `upy-pkg-guide` as an adapter, but the output must be normalized to JSON: package name, source, version, files, README, example, API summary, warnings.
- If explicit `devices[].driver.status` / `driver_status` is `cold_driver_required`, `pending_validation`, `unverified`, `failed`, or an unknown value, do not enter business code generation; output partial and suggest running `upy-gen-driver-plugin` pipeline mode first. Do not treat the absence of a status on ordinary source-only dependencies as a signal that a cold-driver is resolved or unresolved.

## Cloud Service/API Integration Rules

- When involving LLM, Volcengine Ark, Alibaba Cloud Bailian/Tongyi, Tencent Hunyuan, Baidu Qianfan, OpenAI, Azure OpenAI, Gemini, Anthropic, ASR/TTS, IoT/MQTT, Webhook, SMS/Email, weather maps, object storage, or any third-party REST API, you must read `references/cloud_integrations.md`.
- First provide the user with official service provider docs/console/pricing links, and let the user decide whether to activate, purchase tokens/quotas, generate API Keys, or use their own HTTPS gateway/proxy.
- `knowledge/cloud_service_catalog.json` is an extensible service catalog. If a service provider is missing, you can generate a `custom_http_proxy` solution, but you must record the reason and user action items.
- `manifest_content.generate.cloud_integrations[]` must record provider, category, services, official_links, credential_management, user_action_required, deploy_ready.
- Do not write real API Key/token/AK/SK/password/Bearer into `conf.py`, tasks, main, tests, logs, phase_complete, or git commit. Only record secret names and deploy-time hints.
- For cloud services requiring HMAC/OAuth/token exchange/mTLS/large SDK/account-level AKSK, prefer generating a gateway/proxy mode; ESP32 only calls a controlled HTTPS gateway.
- When cloud services are unconfirmed, billing not activated, credentials not prepared, or only mocked, `next_phase` must be `upy-simulate-plugin` or `null`, and cannot default to deploy.
- Run `scripts/check_cloud_integrations.py --project-dir <project_root>`; before real success, also pass `check_phase_complete_consistency.py`.

## Code Generation Constraints

- First read `references/legacy_constraints.md`, retaining the unit-test-driven embedded development philosophy of the old `upy-generate`.
- Except for `firmware/main.py` and hardware factory, business tasks must not import `machine`.
- Tasks use dependency injection, not direct hardware instantiation.
- Each sensor/device read/write must have its own try/except; one failure should not affect others.
- Critical states must be dual-written via print + `lib.logger`: startup, driver initialization, readings, alarms, display, network send, exceptions.
- When generating tasks, you must follow the logging matrix in `references/task_generation_rules.md`.
- When generating factory/mock, you must follow the I2C/GPIO/SPI templates and driver API parsing rules in `references/driver_factory_templates.md`.
- When generating `main.py` and `conf.py`, you must follow the rotating logger, I2C scan, boot delay, and config constant rules in `references/main_conf_rules.md`.
- PC tests must use CPython `unittest`, covering three scenarios: normal, device is None, and driver exception.
- Device tests must read and follow `references/device_unittest_subset.md`: use the MicroPython-runnable `unittest` subset, covering device-side testable protocol, state, driver adapter, configuration, or lightweight filesystem behavior; do not generate pytest, `unittest.mock`, `pathlib`, `tempfile`, `typing`, or other CPython-only test code.
- Do not write Wi-Fi passwords, API Keys, or tokens into `conf.py`.
- Do not silently modify `board.py` pinout; output a structured error if pin issues are found.

## MicroPython-Aware Validation

The MicroPython official documentation states that its standard library is a streamlined subset for embedded systems, and different ports/firmware may trim modules; therefore, you cannot rely solely on CPython `flake8`. After generation, you must run the complete quality gate suite. Prefer using the unified script:

```text
python scripts/update_session_state.py --session-dir <session_root> --checkpoint tests_generated --step quality_gates --status running --idempotency-key <stable-key>
python scripts/run_quality_gates.py --project-dir <project_root> --session-dir <session_root>
```

This script should cover:

```text
ensure .pylintrc
check_generate_plan.py
py_compile
check_conf_contract.py
driver source compile
flake8
pylint
PC unittest
check_mpy_imports.py
check_mpy_imports.py --include-lib
check_dead_config.py
check_task_no_machine_import.py
check_device_unittest_subset.py
check_runtime_dependencies.py
check_doc_evidence.py
check_skeleton_compliance.py
check_generated_semantics.py
check_cloud_integrations.py
update_session_state.py --check
check_final_review_consistency.py
check_phase_complete_consistency.py
```

`.flake8` should preferentially reuse the configuration generated by scaffold, only making project-level supplements, not overriding upstream. If scaffold did not generate `.pylintrc`, generate must write or request to write it via `scripts/ensure_pylintrc.py`, and cannot skip pylint. Pylint is a strong gate for files generated by generate: `firmware/main.py`, `firmware/drivers/**/*.py`, `firmware/tasks/*.py`; by default, only fatal/error/usage bits are treated as hard failures; warning/refactor/convention are recorded as warnings, unless `--strict-pylint` is explicitly used. `firmware/lib` and scaffold framework libraries only undergo compile/import risk checks; style noise should not block success.

MicroPython import checks must distinguish between real runtime imports and PC fallbacks. Allow:

```python
try:
    import uasyncio as asyncio
except ImportError:
    import asyncio
```

Such fallbacks are only `MPY_IMPORT_CPYTHON_FALLBACK` warnings; direct `import asyncio`, `typing`, `dataclasses`, `pathlib`, CPython `logging` remain hard failures.

`check_generated_semantics.py` is a strong gate. It must intercept issues like runtime placeholders, resetting state machines every tick, async synchronous network calls, discarding hardware data after reading, and shared I2S/SPI/UART resources without `generate.resource_plan`. When these issues are hit, do not output deploy-ready success. If the generated `main.py` installs a rotating logger, it must have a top-level startup fatal guard: on exception, both `sys.print_exception()` to serial and `logger.exception()` to the device log. The generated `Scheduler(...)` must pass `error_cb`, and task exceptions must also be dual-written via `print + logger.exception`.

## Fix Mode

Fix mode can come from `upy-autofix-plugin` or from user manual feedback after deploy. Rules:

1. Only make minimal changes; do not rewrite the entire project.
2. Input must include `error_context`, including traceback, file paths, device observations, user feedback, triage_json, or previous_attempts.
3. Before modifying, read `generate_fix_history.json` and the last commit.
4. After modification, re-run lint/check.
5. After passing, git commit.
6. Output `code_diff`, `changed_files`, `attempts[]`, `knowledge_refs[]`.

Boundaries:

| Problem Type | Handling |
|---|---|
| Business logic, driver API calls, thresholds, logging | generate fix |
| Wrong pin connections, I2C address changes, bus conflicts | structured error, suggest select-hw or manual confirmation |
| Adding/replacing hardware | Fall back to analyze/select-hw/scaffold |
| Flashing, serial, upload failures | deploy retry or device troubleshooting |
| Driver does not exist | partial, trigger gen-driver or simulate |

## phase_complete Output

Successful output must include:

- `manifest_content.phase="generate"`.
- `manifest_content` retains the complete upstream manifest: `requirements`, non-empty `devices`, `mcu`, `pinout`, `scaffold`/`scaffold_mode` and other fields must not be replaced by a `generate` summary.
- `project/project-manifest.json` is synchronously updated to `phase="generate"`.
- `generate.behavior_spec`.
- `generate.deploy_plan`.
- `generate.simulation_hints`.
- `generate.cloud_integrations` (if cloud API/LLM/IoT/Webhook is involved).
- `generate.git.commit`.
- `lint.flake8` / `lint.pylint` / `tests.pc_unittest` / `checks`.
- `file_manifest`, and it must include a manifest role entry for `project-manifest.json`.
- `file_manifest` should include a `role="plan"` entry for `generate_plan.json`.
- `session_state.upy_generate_plugin.json` artifact, `checks.session_state_checkpoint.ok=true`, `checkpoint`.
- `permissions`.
- `optional_next_phases`.
- `review_findings.blocking=[]`.

`manifest_content` must be the complete updated project manifest, not just summary fields like `phase/schema_version/project_name/updated_at`. Before `phase_complete.result=success`, it must pass `scripts/check_phase_complete_consistency.py`.

Default:

```json
{
  "payload": {
    "phase": "generate",
    "result": "success",
    "next_phase": "upy-deploy-plugin",
    "optional_next_phases": [
      {"phase": "upy-diagram-plugin"},
      {"phase": "upy-wiring-plugin"}
    ],
    "manifest_content": {
      "phase": "generate"
    }
  }
}
```

On partial/failure, `next_phase=null`, and include:

```json
{
  "structured_errors": [
    {
      "code": "lint_failed",
      "severity": "error",
      "phase_step": "lint",
      "retryable": true,
      "message": "flake8 failed"
    }
  ]
}
```

## Local Verification

Run:

```bash
python -X utf8 upy-generate-plugin/test/smoke_tests.py
```

Local runner:

```bash
python -X utf8 upy-generate-plugin/test/run_local_mock_session.py --session-dir <session_root> --write-phase-complete
```

The local runner is only for mock/verification of the plugin protocol; it does not mean the real LLM has completed all business code generation.
## Final Boundary Addendum

- Session ownership and deploy feedback boundaries live in `references/protocol_fields.md`; always distinguish `workflow_session_root` from `diagnostic_log_session`, and route deploy evidence through `error_context` to autofix/generate fix.
- Timer and scheduler port rules live in `knowledge/esp32_timer_scheduler_api.pitfall.json`; read it plus official MicroPython Timer docs before editing `main.py` scheduler assembly.
- Scaffold framework ownership, `main.py` assembly, and logger timestamp call-site rules live in `references/main_conf_rules.md`; do not patch scaffold-owned libraries to hide generated-app bugs.
- MicroPython import policy lives in `references/validation_gates.md` and `knowledge/micropython_imports.pitfall.json`; guarded CPython-only branches are warnings, direct runtime CPython imports remain hard failures.
- Runtime mip dependency policy lives in `knowledge/mip_runtime_dependencies.pitfall.json`; generate declares `runtime_dependencies.mip`, deploy installs/verifies with `mpremote mip install`, and generate does not vendor micropython-lib source by default.
- Official hardware/peripheral documentation evidence rules live in `references/validation_gates.md` and `knowledge/micropython_official_library_index.json`; exact class/module pages are required when available.
