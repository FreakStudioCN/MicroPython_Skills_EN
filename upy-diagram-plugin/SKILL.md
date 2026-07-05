---
name: upy-diagram-plugin
description: Plugin-based MicroPython software architecture diagram generation phase. Used after receiving the optional_next_phases selection from upy-generate-plugin success, reads the generated firmware and project-manifest.json, generates docs/diagram.json, validates against diagram.schema.json, renders architecture/flowchart/data_flow as md/svg/png/html, and outputs a phase_complete with session/checkpoint/artifact manifest; compatible with plugin protocol invocation and local skill direct testing, does not overwrite the original upy-diagram.
---

# upy-diagram-plugin Plugin Workflow

`upy-diagram-plugin` is an optional software architecture diagram artifact phase in the "one-sentence build hardware" pipeline. It is migrated from the old `G:\MicroPython_Skills\upy-diagram`, but local I/O must be changed to the plugin protocol:

```text
status_update(...)
approval_request(...)
file_operation(read/write/list)
script_run(...)
phase_complete(...)
```

Official chain:

```text
upy-generate-plugin success
  -> optional_next_phases includes upy-diagram-plugin
  -> user selects diagram artifacts
  -> upy-diagram-plugin
```

`upy-diagram-plugin` must not alter the main chain:

```text
upy-generate-plugin -> upy-deploy-plugin
```

That is, the diagram is an optional additional artifact phase, and `phase_complete.payload.next_phase` must default to `null`.

## Boundary Rules

- Do not overwrite the old `G:\MicroPython_Skills\upy-diagram`.
- Do not overwrite or rename `G:\MicroPython_Skills\upy-deploy` or `G:\MicroPython_Skills\upy-deploy-plugin`.
- Do not add hardware, replace MCUs, change pinouts, or modify firmware business code during the diagram phase.
- Do not execute mpremote, flashing, serial debugging, or device-side testing.
- Do not make the diagram a mandatory phase for deployment.
- Do not require the plugin side to understand MicroPython software architecture semantics; the LLM is responsible for understanding, the script is responsible for validation and rendering.

## Data Authority Order

When generating `docs/diagram.json`, facts must be determined using the following priority:

```text
firmware/ actual code > project-manifest.json > LLM inference
```

Where:

- `firmware/main.py` is the highest priority fact for execution flow, DI assembly, scheduler registration, and startup logic.
- `firmware/board.py` represents board-level pin constant mappings, classified under the `board` layer.
- `firmware/lib/**/*.py` represents base libraries, classified under the `lib` layer.
- `firmware/drivers/**/__init__.py` and `mock.py` represent the driver layer, classified under the `driver` layer.
- `firmware/tasks/*.py` represents business tasks, classified under the `task` layer.
- `test/pc/*.py`, `test/device/*.py` represent the test layer; if present, classified under the `test` layer.
- `project-manifest.json` is the design intent and upstream generation record, but in case of conflict with firmware, firmware takes precedence and warnings are generated.
- The Diagram phase is only allowed to supplement/update the `diagrams` field; it must not modify upstream facts like `mcu`, `board`, `devices`, `pinout`, `generate`, or the root-level `updated_at`.

## start_phase Input

The official mode must start from the success phase_complete of `upy-generate-plugin`:

```json
{
  "protocol_version": "1.0",
  "type": "start_phase",
  "phase": "upy-diagram-plugin",
  "session_id": "uuid",
  "msg_id": "uuid",
  "timestamp": "2026-07-02T00:00:00Z",
  "idempotency_key": "upy-diagram-plugin:<session_id>:full:v1",
  "payload": {
    "mode": "full",
    "invocation_mode": "plugin_protocol",
    "local_test": false,
    "source_phase": "upy-generate-plugin",
    "source_phase_complete_path": "sessions/<session_id>/phase_complete.upy_generate_plugin.json",
    "complexity": null,
    "runtime_context": {
      "session_root": "sessions/<session_id>",
      "project_root": "sessions/<session_id>/project",
      "file_operation_root": "sessions/<session_id>/project",
      "resource_root": "upy-diagram-plugin"
    },
    "capabilities": {
      "protocol_versions": ["1.0"],
      "approval_request": true,
      "file_operation": true,
      "script_run": true,
      "checkpoint_resume": true,
      "cancellation": true,
      "retry": true,
      "timeout": true,
      "permission_prompt": true,
      "artifact_manifest": true,
      "network_access": {
        "allowed": true,
        "domains": ["mermaid.ink"]
      }
    },
    "render_policy": {
      "formats": ["json", "md", "html", "svg", "png"],
      "network_rendering": "ask",
      "timeout_ms": 90000
    }
  }
}
```

During migration, `mode=direct_test` is allowed, but `source_phase=test_only` must be recorded. If the complete firmware or generate phase_complete is missing, an official success cannot be output.

## Protocol Field Semantics

These fields must use the same semantics for both plugin protocol invocation and local skill invocation testing:

| Field | Meaning and Constraints |
|---|---|
| `protocol_version` | Protocol version. Currently only accepts `"1.0"`; if unsupported, output `DIAGRAM_PROTOCOL_UNSUPPORTED` and do not proceed. |
| `type` | Message type, e.g., `start_phase`, `status_update`, `approval_request`, `phase_complete`. The plugin routes based on this. |
| `phase` | Must be uniformly `upy-diagram-plugin`. Must not mix `diagram` or `upy-diagram`. |
| `session_id` | Stable ID for one user workflow. Checkpoint, resume, retry, artifact archiving, and log tracing all depend on it. |
| `msg_id` | Single message ID. On retry, `retry_of` points to the original `msg_id` or original `idempotency_key`. |
| `idempotency_key` | Idempotency key. Retries for the same session/phase/mode/attempt should remain stable to avoid duplicate artifact writes or state progression. |
| `payload.mode` | `full` is the official plugin chain; `direct_test` is the local skill test chain and cannot masquerade as an official success. |
| `payload.invocation_mode` | `plugin_protocol` means files/scripts/confirmations go through protocol tools; `local_skill_test` means local tests can directly read/write the project root. |
| `payload.source_phase` | The official chain must be `upy-generate-plugin`; local tests can be `test_only`. |
| `payload.source_phase_complete_path` | Path to the upstream generate phase_complete, used to prove firmware has been generated and code facts come from the generate output. |
| `payload.source_phase_complete` | Optional inline upstream result. If both path and inline object exist, the result read from the path must be used and consistency checked. |
| `payload.complexity` | `simple`, `medium`, `full`, or null. If null, an `approval_request(approval_id="diagram_complexity")` is required. |
| `runtime_context.session_root` | Root directory for the current session's state, checkpoints, phase_complete, logs, and temporary results. |
| `runtime_context.project_root` | User project root. `project-manifest.json`, `firmware/`, `docs/` should all be here. |
| `runtime_context.file_operation_root` | File boundary the plugin is allowed to read/write. Any file write must fall within this directory. |
| `runtime_context.resource_root` | Plugin resource root, e.g., `upy-diagram-plugin`, used to locate scripts. |
| `capabilities` | Capability negotiation result. In official mode, if `file_operation`, `script_run`, or necessary `approval_request` are missing, do not proceed. |
| `render_policy.formats` | Requested artifact formats. Official success must include `json/md/html/svg/png`. |
| `render_policy.network_rendering` | Network rendering policy: `ask`, `allow`, `deny`. If `deny`, only JSON/MD/HTML or local renderer results are allowed. |
| `render_policy.timeout_ms` | Full rendering timeout, default 90000ms. |
| `checks` | Structured validation results. Each check should include `ok`, `command`, `duration_ms`, `error_code`. |
| `artifacts` | List of artifacts for UI and user display. Records type, path, required, sha256, bytes, generated_at. |
| `file_manifest` | File manifest for recovery, acceptance, and idempotent deduplication. More file-system evidence oriented than artifacts. |
| `errors` | Structured error array. Must not be just natural language strings. |
| `warnings` | Non-blocking warning array. Missing SVG/PNG in official mode is not a warning; it should result in `partial`. |

## Plugin Invocation and Local Skill Testing

`upy-diagram-plugin` must be compatible with both invocation methods but must not split into two sets of business rules:

```text
Plugin protocol invocation:
  All file reads/writes via file_operation
  All script execution via script_run
  User confirmation via approval_request

Local skill invocation testing:
  Allows direct read/write to project_root
  Still generates phase_complete with the same structure
  Still writes session_state/checkpoint
  Still runs schema, artifact, file_manifest validation
```

Local testing can use:

```json
{
  "payload": {
    "mode": "direct_test",
    "invocation_mode": "local_skill_test",
    "local_test": true,
    "source_phase": "test_only",
    "complexity": "medium"
  }
}
```

Files written during local testing are only evidence of direct testing, not official pipeline artifacts. The official completion criteria only apply to `phase_complete.payload.manifest_content.diagrams` where `mode=full`, `invocation_mode=plugin_protocol`, and `source_phase=upy-generate-plugin`.

Local direct testing and official plugin invocation must be strictly isolated:

- When `mode=direct_test`, `invocation_mode=local_skill_test`, or `source_phase=test_only`, `phase_complete.payload.result` must be `partial`, and a `LOCAL_TEST_ONLY` warning must be declared in `warnings`; `success` or non-protocol enum values are not allowed.
- Local direct testing must not write to the official `project-manifest.json.diagrams`, nor masquerade as official diagrams in `phase_complete.payload.manifest_content.diagrams`. Direct test artifacts should only be written to `phase_complete.payload.artifacts`, `file_manifest`, `checkpoints/diagram/`, or the test-specific field `project-manifest.json.test_artifacts.diagram`; if recording in `manifest_content` is needed, only `manifest_content.test_artifacts.diagram.files` can be used.
- Official `success` must come from `mode=full`, `invocation_mode=plugin_protocol`, `source_phase=upy-generate-plugin`, and must have a `source_phase_complete_path` pointing to a successful `phase_complete.upy_generate_plugin.json`.
- If existing `docs/diagram.json`, `docs/architecture.*`, `docs/flowchart.*`, `docs/data_flow.*` from `direct_test/test_only` are found, these artifacts must be cleaned or overwritten before official execution, and `diagram_file_manifest.json` and checkpoints must be rebuilt; old direct test artifacts must not be reused as official success.

## Execution Steps

1. Validate `protocol_version == "1.0"`, `phase == "upy-diagram-plugin"`, `session_id` is not empty.
   - If the target project has `.upy/scripts/render_diagram_local.py`, first verify its SHA256 matches `<resource_root>/scripts/render_diagram_local.py`; if it doesn't match, overwrite the project's `.upy` copy with the script from `<resource_root>`, or directly execute `<resource_root>/scripts/render_diagram_local.py`; do not invoke the old project copy.
2. Send `status_update(stage="start")`, indicating validation of upstream generate output is in progress.
3. Read `source_phase_complete_path` via `file_operation(read)`. The upstream must satisfy:
   - `type == "phase_complete"`
   - `phase == "upy-generate-plugin"`
   - `payload.result == "success"`
   - `payload.manifest_content.phase == "generate"`
4. Read `project-manifest.json`. In official mode, use `file_operation(read)`; local direct testing can read directly.
5. If `complexity` is null, send `approval_request(approval_id="diagram_complexity")`. Do not proceed without confirmation.
6. Enumerate and read:
   - `firmware/**/*.py`
   - `firmware/main.py`
   - `firmware/conf.py`
   - `firmware/board.py`
   - `firmware/lib/**/*.py`
   - `firmware/drivers/**/*.py`
   - `firmware/tasks/**/*.py`
   - `test/pc/**/*.py`, `test/device/**/*.py` (if present)
7. Write checkpoint `diagram:after_source_read`.
8. LLM generates `{project_root}/docs/diagram.json` according to `diagram.schema.json`. Must include `meta`, `architecture`, `flow`, `data_flow`; may include `task_registry`, `diagnostics`.
9. Write `docs/diagram.json` via `file_operation(write)`.
10. Run schema validation:

```text
script_run(
  "python G:/MicroPython_Skills/upy-project-gen-toolchain-spec/scripts/validate_json.py --schema G:/MicroPython_Skills/upy-project-gen-toolchain-spec/diagram.schema.json --json <project_root>/docs/diagram.json"
)
```

11. If schema validation fails, correct `docs/diagram.json` and repeat validation. Maximum 3 rounds; after exceeding, output `phase_complete(result="failed",next_phase=null)`.
12. Write checkpoint `diagram:after_validate_pass`.
13. If `render_policy.network_rendering == "ask"` and SVG/PNG are needed, send a network permission request:

```text
approval_request(approval_id="diagram_network_render")
```

14. Render local artifacts:

```text
script_run(
  "python <resource_root>/scripts/render_diagram_local.py --input <project_root>/docs/diagram.json --output <project_root>/docs/ --format all --json-summary"
)
```

Official success must generate 13 files:

```text
docs/diagram.json
docs/architecture.md
docs/architecture.svg
docs/architecture.png
docs/architecture.html
docs/flowchart.md
docs/flowchart.svg
docs/flowchart.png
docs/flowchart.html
docs/data_flow.md
docs/data_flow.svg
docs/data_flow.png
docs/data_flow.html
```

If the user denies network or network fails:

- If degradation is allowed, keep `diagram.json`, `.md`, `.html`, output `partial` and checkpoint.
- If degradation is not allowed, output `failed` with error code `DIAGRAM_RENDER_NETWORK_FAILED` or `DIAGRAM_IMAGE_RENDER_PERMISSION_DENIED`.

15. Collect actually generated files via `script_run` or `file_operation(list)`.
16. Update the `diagrams` field in `project-manifest.json` without changing upstream facts; if recording diagram generation time is needed, only write to `diagrams.generated_at` or the phase_complete `timestamp`.
17. Write session state:

```text
<session_root>/checkpoints/diagram/session_state.upy_diagram_plugin.json
<session_root>/diagram_file_manifest.json
```

18. Run this plugin's validation:

```text
script_run(
  "python <resource_root>/scripts/diagram_manifest.py --validate-phase-complete --input <session_root>/phase_complete.upy_diagram_plugin.json --artifact-root <project_root> --session-root <session_root>"
)
```

`--artifact-root` must point to the project root directory, used to verify that files declared in `file_manifest.files[]` actually exist, `bytes` match, and `sha256` match. If `--artifact-root` is missing, the script only performs protocol structure validation and cannot be used as the final success acceptance criterion.

19. Output `phase_complete`.

## complexity Constraints

`complexity` controls the upper limit of diagram readability:

| Parameter | simple | medium default | full |
|---|---:|---:|---:|
| `architecture` total modules | 6 | 10 | 16 |
| Modules per layer | 2 | 4 | 6 |
| `cross_layer_deps` total edges | 6 | 12 | 20 |
| `flow[]` total steps | 5 | 8 | 14 |
| `data_flow[]` total edges | 2 | 4 | 8 |

The LLM must actively merge similar modules, steps, or data flows to avoid exceeding constraints. `diagnostics` must record the actual module count, dependency edge count, maximum depth, circular dependencies, orphan modules, and direct machine access.

`data_flow[].data` and `data_flow[].rate` must keep short labels; do not write full expressions, long sentences, or Mermaid-sensitive delimiters. Avoid directly using ASCII `()`, `/`, `@`, `|`, `==`, arrow symbols, or code snippets in edge labels; prefer short text like `button press`, `poll 50ms`, `30ms chunk`. The rendering script must still perform final escaping, cleaning, and truncation to prevent mermaid.ink from returning 400 for SVG/PNG.

`diagnostics.total_modules` must equal the actual number of modules in `architecture.layers[].modules`; if modules are merged for readability, count based on the merged graph's modules and explain the merge rationale in the module `role` or `diagnostics`.

## checkpoint / resume

Each resumable boundary must define a checkpoint:

| checkpoint | Generation Timing | resume Behavior |
|---|---|---|
| `input_loaded` | After reading upstream manifest and runtime_context | Continue reading source code |
| `source_read` | After firmware source code reading is complete | Reuse source summary, continue generating diagram.json |
| `diagram_json_written` | After `docs/diagram.json` is written | Start from validation |
| `diagram_json_validated` | After schema validation passes | Start from rendering |
| `artifacts_rendered_partial` | After partial rendering success | Only re-run for missing files |
| `manifest_updated` | After manifest is written back | Directly output phase_complete |
| `phase_completed` | After phase_complete is output | Idempotently return existing result |

`partial`, `failed`, and `cancelled` results must include `payload.checkpoint_info`:

```json
{
  "checkpoint": {
    "checkpoint_id": "diagram_json_written",
    "resume_phase": "upy-diagram-plugin",
    "resume_step": "validate",
    "resume_label": "Continue validating and rendering architecture diagram",
    "resume_from": {
      "diagram_json": "docs/diagram.json",
      "project_root": "sessions/<session_id>/project"
    }
  }
}
```

On resume, the artifact pointed to by the checkpoint must be used first; do not re-infer from natural language or old conversation.

## cancellation / retry / timeout

- On user cancellation, output `phase_complete(result="cancelled", next_phase=null)`.
- If cancellation occurs after source code reading or JSON generation, keep written artifacts, return `partial + checkpoint`.
- Repeated `file_operation(write)` with the same `idempotency_key` should overwrite the same target file, not generate random new files.
- Repeated `script_run(render)` with the same `idempotency_key` should recognize existing files and report actual file status via `--json-summary`.
- Validation repair is limited to 3 rounds; after exceeding, return a structured error.
- Network rendering failure is retried a maximum of 2 times; if still failing, return `partial` or `failed`.

Suggested timeouts:

| Action | timeout |
|---|---:|
| `file_operation(read firmware list)` | 10000 ms |
| `file_operation(read single file)` | 5000 ms |
| `file_operation(write diagram.json)` | 5000 ms |
| `script_run(validate_json.py)` | 15000 ms |
| `script_run(render_diagram_local.py --format all)` | 90000 ms |
| `file_operation(write project-manifest.json)` | 5000 ms |

## permission prompts

The Diagram phase typically requires:

| Permission | Required | Description |
|---|---|---|
| File Read | Yes | Read manifest and firmware source code |
| File Write | Yes | Write docs and project-manifest.json |
| Script Execution | Yes | validate and render |
| Network Access | Conditional | mermaid.ink rendering SVG/PNG |
| Device Access | No | The diagram phase must not send `device_command` |

If the host requires explicit authorization, `permission_request` or equivalent `approval_request` must be used, and the scope must be clearly stated:

```text
Read: project-manifest.json, firmware/**/*.py
Write: docs/diagram.json, docs/architecture.*, docs/flowchart.*, docs/data_flow.*, project-manifest.json
Script: scripts/diagram_manifest.py, scripts/render_diagram_local.py, validate_json.py
Network: mermaid.ink
```

## structured errors

`errors` must be an array of objects:

```json
{
  "code": "DIAGRAM_RENDER_TIMEOUT",
  "message": "Mermaid diagram rendering timed out",
  "step_id": "render",
  "severity": "error",
  "retryable": true,
  "recoverable": true,
  "idempotency_key": "upy-diagram-plugin:<session_id>:render:v1",
  "artifact_refs": ["docs/diagram.json"],
  "suggested_actions": ["Retry rendering", "Keep only Markdown/HTML output", "Continue from checkpoint later"]
}
```

These error codes must be covered:

```text
DIAGRAM_INPUT_MISSING
DIAGRAM_PROTOCOL_UNSUPPORTED
DIAGRAM_CAPABILITY_MISSING
DIAGRAM_PERMISSION_DENIED
DIAGRAM_SOURCE_READ_FAILED
DIAGRAM_JSON_INVALID
DIAGRAM_VALIDATE_FAILED
DIAGRAM_RENDER_TIMEOUT
DIAGRAM_RENDER_NETWORK_FAILED
DIAGRAM_MANIFEST_UPDATE_FAILED
DIAGRAM_CANCELLED
DIAGRAM_IDEMPOTENCY_CONFLICT
```

## phase_complete Success Form

A successful output must satisfy:

- `type == "phase_complete"`
- `phase == "upy-diagram-plugin"`
- `payload.phase == "upy-diagram-plugin"`
- `payload.result == "success"`
- `payload.mode == "full"`
- `payload.invocation_mode == "plugin_protocol"`
- `payload.local_test != true`
- `payload.source_phase == "upy-generate-plugin"`
- `payload.source_phase_complete_path` points to a successful `phase_complete.upy_generate_plugin.json`
- `payload.next_phase == null`
- `payload.manifest_content.diagrams` contains 13 file paths and `generated_at`
- `payload.artifacts` contains 13 file artifacts
- `payload.file_manifest.files[]` declares 13 files, including `path`, `type`, `required`, `bytes`, `sha256`
- `payload.checks.diagram_schema.ok == true`
- `payload.checks.render_diagram.ok == true`
- `payload.session_state.checkpoint == "phase_completed"`
- `payload.errors == []`

## Script Resources

This skill comes with:

```text
scripts/render_diagram_local.py
scripts/diagram_manifest.py
```

`render_diagram_local.py` is copied from the old `upy-diagram` and supplemented with `--json-summary`. Running `--format md` or `--format html` does not require network; running `--format svg/png/all` may access mermaid.ink.

`diagram_manifest.py` is used for local testing and plugin host validation:

```text
python scripts/diagram_manifest.py --validate-phase-complete --input <phase_complete.json> --artifact-root <project_root> --session-root <session_root>
python scripts/diagram_manifest.py --build-file-manifest --artifact-root <project_root> --output <session_root>/diagram_file_manifest.json
```

## Local Acceptance

After implementing or modifying this skill, at least run:

```text
python test/smoke_tests.py
```

Tests must cover:

- `SKILL.md` frontmatter and key protocol text.
- `.codex-plugin/plugin.json` is verifiable.
- Sample JSON structure for phase, result, checkpoint, errors, file_manifest.
- `diagram.sample.json` passes `diagram.schema.json`.
- `render_diagram_local.py --format md --json-summary` can generate Markdown and output JSON summary.
- `diagram_manifest.py` can validate success, partial, cancelled, timeout, permission_denied, capability_unavailable samples.
- `diagram_manifest.py` must reject direct test success, direct test missing `LOCAL_TEST_ONLY`, direct test using official `manifest_content.diagrams`, non-success result missing `checkpoint_info`, and inconsistency between sidecar manifest and `payload.file_manifest`.
