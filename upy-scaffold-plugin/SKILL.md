---
name: upy-scaffold-plugin
description: Plugin-based workflow MicroPython project skeleton generation. Used when Codex receives a phase_complete(upy-flash-mpy-firmware-plugin) with next_phase=upy-scaffold-plugin, or when the user adds a new device in a later phase; consumes the select-hw manifest_content, generates a firmware/ skeleton written via file_operation after approval of scheduling mode and modules, or incrementally generates only the new device driver stub.
---

# upy-scaffold-plugin Plugin Workflow

`upy-scaffold-plugin` is the plugin-based version of the third-phase project skeleton generation. It only builds the project skeleton (`firmware/`, `tools/`, `.upy/`, etc.), without writing business tasks, filling in driver implementations, or performing synchronous/asynchronous driver conversion; these are left to `upy-generate-plugin`.

The upstream formal chain is:

```
upy-analyze-plugin -> upy-select-hw-plugin -> upy-flash-mpy-firmware-plugin -> upy-scaffold-plugin -> upy-generate-plugin
```

Input facts must come from the `manifest_content` of `select-hw`. When the start message originates from `upy-flash-mpy-firmware-plugin`, read `payload.manifest_content` first; if missing, trace back to `phase_complete.select_hw.json.payload.manifest_content` via `payload.source_phase_complete` or `payload.source_phase_complete_path`. Do not infer hardware facts from logs, old drafts, or conversation memory.

## Start Message

Full mode:

```json
{
  "type": "start_phase",
  "phase": "upy-scaffold-plugin",
  "payload": {
    "mode": "full",
    "source_phase": "upy-flash-mpy-firmware-plugin",
    "source_phase_complete_path": "sessions/<session_id>/phase_complete.upy_flash_mpy_firmware_plugin.json",
    "runtime_context": {
      "artifact_root": ".",
      "artifact_root_mode": "cwd",
      "session_root": "sessions/<session_id>",
      "project_root": "sessions/<session_id>/project",
      "resource_root": "<runtime-provided>"
    },
    "capabilities": {
      "approval_request": true,
      "script_run": true,
      "file_operation": true
    }
  }
}
```

Incremental mode:

```json
{
  "type": "start_phase",
  "phase": "upy-scaffold-plugin",
  "payload": {
    "mode": "incremental",
    "manifest": { "phase": "scaffold" },
    "new_devices": [{ "name": "DHT22", "driver": { "source": "upypi" } }]
  }
}
```

## Full Flow

1. Validate the upstream phase: `phase_complete(upy-flash-mpy-firmware-plugin)` must have `result=success` and `next_phase=upy-scaffold-plugin`. During migration, direct testing may pass the `select-hw` manifest directly, but the formal chain must not skip the firmware phase.
2. Read `mcu`, `devices`, `pinout`, `bom`, `requirements` from `manifest_content`.
3. Send `approval_request(scaffold_config)`, merging scheduling mode, extra modules, and custom files into a single card.
4. After user confirmation, run `scripts/init_scaffold.py` to generate stdout JSON.
5. Use `directories[]` from the stdout JSON for plugin-side pre-creation of directories, and convert `files[]` one by one into `file_operation(op=write)`.
6. Send `script_run(flake8)`, to be executed by the host in the project directory; the script itself does not run flake8.
7. After the host writes `file_operations[]` to the project directory, run `python -m flake8 firmware tools`; must use the MicroPython-aware configuration from the project root `.flake8`, and must return 0 to proceed.
8. Output `phase_complete(result=success, next_phase=upy-generate-plugin)`, and include the updated manifest output by the script in `payload.manifest_content`.

## approval_request: scaffold_config

Send only one approval request, with `approval_id` fixed to `scaffold_config`. Must include `item_groups`:

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "scaffold_config",
    "header": "Project Skeleton Configuration",
    "question": "Select scheduling mode and modules to inject",
    "items": [
      {"id": "mode_timer", "name": "Timer tick", "group": "scheduler_mode"},
      {"id": "mode_async", "name": "asyncio", "group": "scheduler_mode"},
      {"id": "mode_thread", "name": "_thread", "group": "scheduler_mode"},
      {"id": "module_logger", "name": "Logging System", "group": "extra_modules"},
      {"id": "module_flash", "name": "Deployment Tool", "group": "extra_modules"}
    ],
    "allow_add": true,
    "item_groups": {
      "scheduler_mode": {"multi_select": false, "label": "Scheduling Mode"},
      "extra_modules": {"multi_select": true, "label": "Extra Modules"}
    }
  }
}
```

Scheduling mode recommendation rules only affect `selected/meta` and cannot restrict user choice:

| Condition | Recommendation |
|---|---|
| `requirements.network == "wifi"` | `mode_async` |
| Display/LCD/LVGL related devices or `special_requirements` containing lcd/lvgl/display | `mode_async` |
| Other | `mode_timer` |

Module ids map to script `--modules`:

| id | Output |
|---|---|
| `module_logger` | `firmware/lib/logger/*` |
| `module_time_helper` | `firmware/lib/time_helper.py` |
| `module_maintenance` | `firmware/tasks/maintenance.py` |
| `module_flash` | `tools/flash_device.py` |
| `module_log_tools` | `tools/read_device_log.py` + `tools/log_report.py` |

If `module_maintenance` is not selected, `main.py` must not import or call `maintenance_tick`. If the mode is not `timer`, `firmware/lib/scheduler/timer_sched.py` must not be injected.

## script_run: init_scaffold.py

The script is a deterministic renderer that only reads the manifest, outputs JSON to stdout, and does not write to the project directory:

```bash
python -X utf8 <resource_root>/upy-scaffold-plugin/scripts/init_scaffold.py \
  --mode timer \
  --manifest <session_root>/phase_complete.upy_flash_mpy_firmware_plugin.json \
  --modules '["logger","flash_device","log_tools"]' \
  --custom-files '["firmware/lib/my_utils.py"]'
```

Prefer `--manifest <path>` over stdin redirection on Windows. If stdin is used (`--manifest -`), `init_scaffold.py` reads raw stdin bytes as UTF-8-SIG; callers should still run Python with `-X utf8`.

Pass the full `manifest_content` via stdin. Stdout format:

```json
{
  "phase": "scaffold",
  "mode": "timer",
  "scaffold_mode": "timer",
  "directories": ["firmware", "firmware/lib/logger"],
  "files": [
    {"path": "firmware/board.py", "content": "...", "encoding": "utf-8"},
    {"path": "project-manifest.json", "content": "{...}", "encoding": "utf-8"},
    {"path": "docs/.gitkeep", "content": "", "encoding": "utf-8"}
  ],
  "file_operations": [
    {
      "type": "file_operation",
      "payload": {
        "op_id": "scaffold_fo_001",
        "op": "write",
        "path": "firmware/board.py",
        "content": "...",
        "encoding": "utf-8"
      }
    }
  ],
  "status_updates": [
    {"step_id": "scaffold_start", "message": "Generating project skeleton...", "level": "info"}
  ],
  "artifacts": [
    {"type": "file_tree", "title": "Project Structure", "tree": {"firmware": {"board.py": "file"}}},
    {"type": "file_list", "title": "Files to Write", "files": [{"path": "firmware/board.py", "status": "pending"}]}
  ],
  "file_tree": {"firmware": {"board.py": "file"}},
  "manifest_content": {"phase": "scaffold", "scaffold_mode": "timer"},
  "phase_complete_payload": {
    "phase": "scaffold",
    "result": "success",
    "summary": "Generated 18 files, 10 directories",
    "next_phase": "upy-generate-plugin",
    "artifacts": []
  },
  "warnings": []
}
```

The host can use `file_operations[]` directly, or assemble them from `files[]`:

```json
{
  "type": "file_operation",
  "payload": {
    "op_id": "scaffold_fo_001",
    "op": "write",
    "path": "firmware/board.py",
    "content": "...",
    "encoding": "utf-8"
  }
}
```

Paths must remain POSIX-style relative to the project root; do not write absolute paths.

`phase_complete_payload` is a payload draft, not a complete message envelope; the real `msg_id`, `session_id`, `timestamp`, and `idempotency_key` are filled in by the runtime host.

## Incremental Flow

When the user adds a new device in a later phase, skip the approval card and run directly:

```bash
python -X utf8 <resource_root>/upy-scaffold-plugin/scripts/init_scaffold.py \
  --mode incremental \
  --manifest - \
  --new-devices '[{"name":"DHT22","driver":{"source":"upypi"}}]'
```

Only output the new device's `firmware/drivers/<name>_driver/__init__.py` stub and the updated `project-manifest.json`, then `phase_complete(result=success, next_phase=upy-generate-plugin)`. Must not rewrite `firmware/main.py`, `board.py`, or other skeleton files. The incremental payload must include `incremental=true` and `generate_scope="new_devices_only"`.

## Output Constraints

- `board.py` only contains pin constants and query functions; do not instantiate hardware.
- `main.py` only generates hardware instantiation and scheduling framework; leave TODO placeholders for business task registration.
- `timer` mode uses `Scheduler` from `firmware/lib/scheduler/timer_sched.py`; do not rewrite this internal library for port compatibility. The library's default `timer_id=-1` must be preserved because RP2/Pico and Zephyr only support virtual Timers. Port differences must be resolved in the `main.py` entry assembly layer: only RP2/Pico/RP2040/RP2350 and Zephyr may explicitly generate `Scheduler(timer_id=-1, tick_ms=...)`; other MCU/ports default to generating `Scheduler(timer_id=0, tick_ms=...)` or another verified non-negative hardware Timer ID. Do not generate implicit `Scheduler(...)`, `Scheduler(tick_ms=...)`, or `Scheduler(timer_id=-1)`. `async` mode uses `uasyncio` directly; `thread` mode uses `_thread` directly.
- GPIO direction must come from `pinout[].type` and pin semantics: `gpio_out`, `DATA`, `DO`, `OUT`, `GAIN`, `SD` default to `Pin.OUT`; `gpio_in` defaults to `Pin.IN`. Do not generate output pins like WS2812 DATA as `Pin.IN`.
- `main.py` must have a startup fatal guard. After installing the rotating logger, critical startup status must be dual-written via `print + logger`; uncaught startup/assembly exceptions must be printed to the serial port via `sys.print_exception()` and written to `/log/run_*.log` via `logger.exception()`. Do not rely on MicroPython automatically writing top-level tracebacks to file logs.
- Do not generate business `tasks/sensor_task.py`, `display_task.py`, or `network_task.py`.
- `conf.py` must not contain Wi-Fi passwords, API Keys, or other sensitive data.
- `tools/flash_device.py` must implement production deployment filtering: `main.py`, `boot.py`, `conf.py` are always uploaded as `.py`, not compiled to `.mpy`; `firmware/drivers/**/mock.py` belongs to test doubles only and must not be compiled or uploaded, and stale `build/mpy/drivers/**/mock.mpy` must also be skipped. The JSON summary must record `compiled_files`, `uploaded_files`, and `skipped_files` for the deploy-plugin to determine forbidden artifacts.
- `.upy/` only copies schemas and tool scripts that actually exist in the current repository; do not fabricate non-existent downstream tools.
- `project-manifest.json` must be written to the project root as a `file_operation`; `payload.manifest_content` simultaneously retains the object form.
- `docs/.gitkeep` must be preserved as the entry point for project documentation.
- The root `.gitignore` must ignore `__pycache__/`, `*.pyc`, and `build/mpy/` to prevent deploy compilation artifacts from polluting the git state of the next generate/autofix round.
- `.flake8` must be a MicroPython-aware configuration: do not globally ignore `F821/F401`, use `builtins=const` and precise `per-file-ignores`.
- `phase_complete.payload.artifacts` must be an array, containing at least `file_tree` and `file_list`.
- `phase_complete.payload.next_phase` is `upy-generate-plugin` on success; `null` on partial/failure.

## Local Verification

Run:

```bash
python -X utf8 upy-scaffold-plugin/test/smoke_tests.py
python -X utf8 upy-scaffold-plugin/scripts/apply_scaffold.py \
  --session-dir <session_root> \
  --manifest <session_root>/phase_complete.upy_flash_mpy_firmware_plugin.json \
  --mode async \
  --modules logger,flash_device,time_helper,maintenance \
  --write-phase-complete
python -X utf8 upy-scaffold-plugin/scripts/scaffold_manifest.py \
  --input <session_root>/phase_complete.upy_scaffold_plugin.json \
  --validate-phase-complete
```

`test/run_local_actual_project.py` is a compatibility wrapper only; formal local actual tests and Claude Code host-side apply/finalize runs should call `scripts/apply_scaffold.py`.

## Protocol Addendum: project root, idempotency, and final manifest

These rules are normative for the plugin workflow and local Claude Code actual tests:

- `session_root` stores phase state, logs, checkpoint files, and phase_complete JSON.
- Scaffold project files MUST be written under `project_root`.
- If the caller only provides a session directory, use `project_root=<session_root>/project`.
- `file_operations[].payload.path` and `files[].path` stay POSIX-style paths relative to `project_root`; never prefix them with `sessions/<id>/project`.
- `python -m flake8 --jobs=1 firmware tools` MUST run with `cwd=project_root` and MUST return 0 before success can advance to `upy-generate-plugin`.
- Final `phase_complete.payload.runtime_context` MUST include `artifact_root`, `artifact_root_mode`, `session_root`, `project_root`, `file_operation_root`, and `resource_root`.
- `runtime_context.project_root`, `runtime_context.file_operation_root`, `payload.file_manifest.root`, and `payload.lint.cwd` SHOULD be artifact-relative POSIX paths, for example `sessions/<session_id>/project`.
- Final `phase_complete.payload.source` MUST record `source_phase`, `source_phase_complete_path`, `source_manifest_kind`, and `manifest_merge_strategy`. Do not rely on top-level `source_phase` fields only.
- Final `phase_complete.payload.scaffold` MUST record a stage-level summary while preserving `manifest_content.scaffold`, `manifest_content.scaffold_modules`, and `project/project-manifest.json`. Include final `mode`, `modules`, `custom_files`, `project_root`, `file_count`, `directory_count`, `file_status_counts`, `file_manifest_path`, `phase_complete_path`, `lint`, `source`, `approval_id`, `idempotency_key`, `incremental`, `generate_scope`, and `completed_at`.
- Final `phase_complete.payload.approval` MUST record the `scaffold_config` decision, including selected mode/modules/custom files and confirmation time.
- Final `phase_complete.payload.permissions` MUST record approved file writes and the flake8 script run with idempotency keys.
- Final `phase_complete.payload.lint` MUST record flake8 `command`, artifact-relative `cwd`, `config`, `returncode`, `stdout`, `stderr`, and `completed_at`.
- Final `phase_complete` MUST pass `scripts/scaffold_manifest.py --validate-phase-complete`. A success payload with non-empty `structured_errors`, flake8 violations in stdout/stderr, or `lint.returncode != 0` is invalid and must be emitted as `partial` with `next_phase=null`.
- Renderer `file_list.status=pending` is only a draft state. Final `phase_complete` after local/host write MUST use `created`, `updated`, `unchanged`, `skipped`, or `error`.
- Final `phase_complete.payload.file_manifest` MUST include `root`, `generated_at`, and per-file `path`, `status`, `encoding`, `bytes`, `sha256`, `sha256_before`, `sha256_after`, `overwrite`, and optional `reason`.
- Success MUST write `scaffold_file_manifest.json` under `session_root`; it MUST use the same object shape as `payload.file_manifest`: `{root, generated_at, files}`. Do not write a bare file array.
- Final `phase_complete.payload.artifacts` MUST include `file_tree`, `file_list`, and `file_manifest`; `artifacts[type=file_manifest].path` MUST point to `sessions/<session_id>/scaffold_file_manifest.json`, and the final `file_list.title` should be `Scaffold Write Results`.
- Retry/idempotency contract:
  - Missing target file -> `created`.
  - Existing identical file -> `unchanged`.
  - Existing different file without explicit overwrite approval -> `skipped` plus `structured_errors[].code=FILE_CONFLICT`, `result=partial`, `next_phase=null`.
  - Explicit overwrite approval or local `--force` -> `updated`.
- Time fields SHOULD be generated through `upy-project-gen-toolchain-spec/scripts/workflow_time.py --json`; fallback local UTC is only for local tooling degradation.
- Windows JSON reads MUST use UTF-8/UTF-8-SIG, never default GBK.
- For Claude Code local actual tests, use `scripts/apply_scaffold.py --session-dir <session_root> --manifest <phase_complete> --write-phase-complete`. Avoid inline `python -c` finalizers with raw Windows paths because `\U` in paths can trigger Python unicodeescape syntax errors.
- Success manifest MUST set `manifest_content.phase=scaffold`, `manifest_content.domain_phase=scaffold`, and `manifest_content.final_status=scaffolded`. Existing `firmware_flash` facts from flash phase MUST be preserved.
- `payload.warnings` and `structured_errors` should use structured objects. Do not write local machine absolute paths into formal warnings or artifacts.

Coverage:

- Full + timer output JSON, paths, and encoding.
- Async mode does not inject scheduler.
- Incremental only generates new driver stub and `project-manifest.json`, then proceeds to `upy-generate-plugin`.
- `approval_request.scaffold_config` `item_groups` grouping protocol.
- Local actual runner applies `file_operations[]` to a temporary project directory and runs the flake8 gate.

## Final Boundary Addendum

- Treat the explicit `runtime_context.session_root` or user-supplied session path as `workflow_session_root`; diagnostic sessions containing logs are evidence only and must not receive scaffold files.
- Keep `project_root=<workflow_session_root>/project` unless the caller explicitly provides another project root. Do not write scaffold output into a log-only session.
- `approval_request(scaffold_config)` must expose `module_time_helper` as an `extra_modules` option for `firmware/lib/time_helper.py`; this is the time measurement helper used later by generate quality gates and generated task timing wrappers.
- The scaffold scheduler library contract is stable: do not modify `firmware/lib/scheduler/timer_sched.py` for a specific MCU. Its default `timer_id=-1` must remain because RP2/Pico and Zephyr support only virtual timers.
- For timer mode, port-specific timer choice belongs to the generated `main.py` assembly: RP2/Pico/RP2040/RP2350 and Zephyr may use `Scheduler(timer_id=-1, ...)`; ESP32/STM32 and other hardware-timer ports must pass a verified non-negative id such as `Scheduler(timer_id=0, ...)`.
- `tools/flash_device.py` must clear stale `build/mpy` before compile and must refuse wrong-root/cache/test artifacts such as `build/mpy/firmware/**`, `:firmware/**`, `main.mpy`, `boot.mpy`, `conf.mpy`, `__pycache__`, `*.pyc`, and `drivers/**/mock.py|mock.mpy`.
- Scaffold must generate a root `.gitignore` that ignores local build outputs such as `build/mpy/` while keeping source firmware, tests, and manifests trackable.
- Fatal startup guards generated by scaffold must print traceback to REPL and call the project logger when installed. Full traceback capture still depends on deploy REPL capture and device log collection.
