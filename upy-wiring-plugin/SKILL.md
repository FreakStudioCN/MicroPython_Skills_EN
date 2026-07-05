---
name: upy-wiring-plugin
description: Plugin-based MicroPython wiring diagram generation phase. Used after receiving the optional_next_phases selection from upy-generate-plugin success, reads the generated firmware and project-manifest.json, generates docs/wiring.json, validates the schema, renders wiring.md/svg/png/html/wiring_pins.md, and outputs phase_complete; this is an optional artifact phase that does not override the main deploy pipeline.
---

# upy-wiring-plugin Plugin Workflow

`upy-wiring-plugin` is the optional wiring diagram artifact phase of the "one-sentence build hardware" pipeline. It is migrated from the old `G:\MicroPython_Skills\upy-wiring`, but must change local I/O to the plugin protocol:

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
  -> optional_next_phases includes upy-wiring-plugin
  -> user selects wiring artifacts
  -> upy-wiring-plugin
```

`upy-wiring-plugin` must not alter the main chain:

```text
upy-generate-plugin -> upy-deploy-plugin
```

That is, wiring is an optional additional artifact phase; `phase_complete.payload.next_phase` must default to `null`.

## Boundary Rules

- Do not overwrite the old `G:\MicroPython_Skills\upy-wiring`.
- Do not overwrite or rename `G:\MicroPython_Skills\upy-deploy` or `G:\MicroPython_Skills\upy-deploy-plugin`.
- Do not add hardware, replace MCU, change pinout, or modify firmware business code during the wiring phase.
- Do not execute mpremote, flashing, serial debugging, or device-side testing.
- Do not make wiring a mandatory phase for deploy.
- Do not require the plugin side to understand MicroPython hardware semantics; the LLM is responsible for understanding, the scripts are responsible for validation and rendering.

## Data Authority Order

When generating `docs/wiring.json`, facts must be determined by the following priority:

```text
firmware/ actual code > project-manifest.json > LLM inference
```

Where:

- `I2C(...)`, `SPI(...)`, `UART(...)`, `Pin(...)` in `firmware/main.py` are the highest priority connection facts.
- Fixed onboard pins and default bus mappings in `firmware/board.py` must be used for completion.
- Default addresses or factory parameters in `firmware/drivers/*/__init__.py` are used to confirm I2C addresses.
- `firmware/conf.py` is used to confirm the project name, board name, and configuration constants.
- `firmware/tasks/*.py` and `firmware/lib/*.py` are used for supplementary checks on extra pins or hardcoded addresses.
- `project-manifest.json` is the design intent and upstream hardware selection record, but in case of conflict with firmware, firmware takes precedence and alerts are generated. The Wiring phase is only allowed to supplement/update the `wiring` field; it must not modify the root-level `updated_at`.

## start_phase Input

The formal mode must start from the success phase_complete of `upy-generate-plugin`:

```json
{
  "protocol_version": "1.0",
  "type": "start_phase",
  "phase": "upy-wiring-plugin",
  "session_id": "uuid",
  "idempotency_key": "upy-wiring-plugin:<session_id>:full:v1",
  "payload": {
    "mode": "full",
    "source_phase": "upy-generate-plugin",
    "source_phase_complete_path": "sessions/<session_id>/phase_complete.upy_generate_plugin.json",
    "runtime_context": {
      "session_root": "sessions/<session_id>",
      "project_root": "sessions/<session_id>/project",
      "file_operation_root": "sessions/<session_id>/project",
      "resource_root": "upy-wiring-plugin"
    },
    "invocation_mode": "plugin_protocol",
    "local_test": false,
    "capabilities": {
      "approval_request": true,
      "file_operation": true,
      "script_run": true,
      "checkpoint_resume": true,
      "cancellation": true,
      "retry": true,
      "timeout": true,
      "permission_prompt": true
    },
    "render_policy": {
      "formats": ["json", "md", "html", "pins", "svg", "png"],
      "network_rendering": "ask",
      "timeout_ms": 30000
    }
  }
}
```

During the migration period, `mode=direct_test` is allowed, but `source=test_only` must be recorded. If the complete firmware or generate phase_complete is missing, a formal success cannot be output.

## Protocol Field Semantics

These fields must use the same semantics for plugin protocol calls and local skill test calls:

| Field | Meaning and Constraints |
|---|---|
| `protocol_version` | Protocol version. Currently only accepts `"1.0"`; if unsupported, output `PROTOCOL_UNSUPPORTED` and do not continue execution. |
| `type` | Message type, e.g., `start_phase`, `status_update`, `approval_request`, `phase_complete`. The plugin routes based on this. |
| `phase` | Must be uniformly `upy-wiring-plugin`. Do not mix with `wiring` or `upy-wiring`. |
| `session_id` | Stable ID for one user workflow. Checkpoint, resume, retry, artifact archiving, and log tracing all depend on it. |
| `idempotency_key` | Idempotency key. Retries for the same session/phase/mode/attempt should remain stable to avoid duplicate artifact writes or state progression. |
| `payload.mode` | `full` is the formal plugin chain; `direct_test` is the local skill test chain and cannot masquerade as a formal success. |
| `payload.invocation_mode` | `plugin_protocol` means file/script/confirmation all go through protocol tools; `local_skill_test` means local tests can directly read/write the project root. |
| `payload.source_phase` | The formal chain must be `upy-generate-plugin`; local tests can be `test_only`. |
| `payload.source_phase_complete_path` | Path to the upstream generate phase_complete, used to prove firmware has been generated and hardware facts come from the generate output. |
| `payload.source_phase_complete` | Optional inline upstream result. If both path and inline object exist, the result read from the path must be used and consistency verified. |
| `runtime_context.session_root` | Root directory for the current session's state, checkpoint, phase_complete, logs, and temporary results. |
| `runtime_context.project_root` | User project root. `project-manifest.json`, `firmware/`, `docs/` should all be here. |
| `runtime_context.file_operation_root` | File boundary the plugin is allowed to read/write. Any file write must fall within this directory. |
| `runtime_context.resource_root` | Plugin resource root, e.g., `upy-wiring-plugin`, used to locate scripts. |
| `capabilities` | Capability negotiation result. The formal mode must not continue if `file_operation`, `script_run`, or `approval_request` are missing. |
| `render_policy.formats` | Requested artifact formats. A formal success must include `json/md/html/pins/svg/png`. |
| `render_policy.network_rendering` | Network rendering strategy: `ask`, `allow`, `deny`. When `deny`, a local renderer must be attempted first. |
| `render_policy.timeout_ms` | Timeout for a single SVG/PNG render, default 30000ms. |
| `checks` | Structured validation results. Each check should include `ok`, `command`, `duration_ms`, `error_code`. |
| `artifacts` | List of artifacts for UI and user display. Records type, path, required, sha256, bytes, generated_at. |
| `file_manifest` | File manifest for recovery, acceptance, and idempotent deduplication. More filesystem-evidence oriented than artifacts. |
| `errors` | Structured error array. Must not only write natural language strings. |
| `warnings` | Non-blocking warning array. Missing SVG/PNG in formal mode is not a warning; it should result in partial. |

## Plugin Calls and Local Skill Testing

`upy-wiring-plugin` must be compatible with both invocation methods, but must not split into two sets of business rules:

```text
Plugin protocol call:
  All file reads/writes via file_operation
  All script execution via script_run
  User confirmation via approval_request

Local skill test call:
  Allows direct read/write of project_root
  Still generates the same structure phase_complete
  Still writes session_state/checkpoint
  Still runs schema, artifact, and file_manifest validation
```

Local tests can use:

```json
{
  "payload": {
    "mode": "direct_test",
    "invocation_mode": "local_skill_test",
    "local_test": true,
    "source_phase": "test_only"
  }
}
```

## Execution Steps

1. Send `status_update(stage="start")`, indicating that the upstream generate output is being validated.
2. Read `source_phase_complete_path` via `file_operation(read)`. The upstream must satisfy:
   - `type == "phase_complete"`
   - `phase == "upy-generate-plugin"`
   - `payload.result == "success"`
   - `payload.manifest_content.phase == "generate"`
3. Read `{project_root}/project-manifest.json` via `file_operation(read)`.
4. Enumerate and read via `file_operation(list)`:
   - `firmware/**/*.py`
   - `firmware/drivers/**/__init__.py`
   - `firmware/conf.py`
   - `firmware/board.py`
   - `firmware/main.py`
5. The LLM generates `{project_root}/docs/wiring.json` based on the rules of the old `upy-wiring`. It must include `meta`, `mcu`, `buses`, `standalone`, `power`, `alerts`.
6. Write `docs/wiring.json` via `file_operation(write)`.
7. Run the deterministic topology derivation script to supplement or overwrite `components[]`, `connections[]`, `buses[]` using `project-manifest.json pinout`:

```text
script_run(
  "python <resource_root>/scripts/derive_wiring_topology.py --wiring <project_root>/docs/wiring.json --manifest <project_root>/project-manifest.json --upstream <session_root>/phase_complete.upy_generate_plugin.json --output <project_root>/docs/wiring.json"
)
```

If the manifest contains I2S/SPI/I2C/UART multi-line interfaces or any non-middleware hardware module, this step is mandatory and must not be skipped. The LLM can generate a draft, but the final `components[]`, `connections[]`, and I2S `buses[]` must be based on `project-manifest.json pinout`.

8. Run schema validation:

```text
script_run(
  "python G:/MicroPython_Skills/upy-project-gen-toolchain-spec/scripts/validate_json.py --schema G:/MicroPython_Skills/upy-project-gen-toolchain-spec/wiring.schema.json --json <project_root>/docs/wiring.json"
)
```

9. If schema validation fails, correct `docs/wiring.json` and repeat the topology derivation and schema validation. If it cannot be corrected, output `phase_complete(result=partial,next_phase=null)`.
10. Render local artifacts:

```text
script_run(
  "python <resource_root>/scripts/render_wiring_local.py --input <project_root>/docs/wiring.json --output <project_root>/docs/ --format all --network-rendering <ask|allow|deny> --timeout-ms <timeout_ms>"
)
```

A formal success must generate `wiring.svg` and `wiring.png`. Recommended rendering degradation chain:

```text
Priority: Local Mermaid CLI / mmdc / available local renderer
  -> On failure: Request user permission for mermaid.ink network rendering
  -> Still fails: phase_complete(result=partial,next_phase=null)
```

If `render_policy.network_rendering=deny`, do not immediately abandon image artifacts; first attempt a local renderer. If there is no local renderer and the user refuses network, output `partial` with error code `WIRING_IMAGE_RENDER_PERMISSION_DENIED`.

11. Collect the actually generated files via `script_run` or `file_operation(list)`.
12. Update the `wiring` field in `project-manifest.json`, without changing upstream facts like `mcu`, `board`, `devices`, `pinout`, `generate`, root-level `updated_at`; if recording the wiring generation time, only write to `wiring.generated_at` or the phase_complete `timestamp`.
13. Run this plugin's validation:

```text
script_run(
  "python <resource_root>/scripts/wiring_manifest.py --validate-phase-complete --input <session_root>/phase_complete.upy_wiring_plugin.json --artifact-root <project_root> --session-root <session_root>"
)
```

`--artifact-root` must point to the project root directory to verify that the files declared in `file_manifest.files[]` actually exist, `bytes` match, and `sha256` match. If `--artifact-root` is missing, the script only performs protocol structure validation and cannot be used as the final acceptance criterion for success.

14. Output `phase_complete`.

## wiring.json Generation Rules

`docs/wiring.json` must conform to:

```text
G:/MicroPython_Skills/upy-project-gen-toolchain-spec/wiring.schema.json
```

The wiring diagram must be expressed as a "component-level, pin-annotated electrical wiring topology diagram":

- Actual hardware modules such as MCU, audio amplifier, microphone, LED, button, sensor must be rendered as independent component boxes.
- Every real signal line, power line, and ground line must be displayed as an independent connection edge.
- Connection edges must be annotated with MCU GPIO, MCU-side signal role, peripheral-side pin name, and necessary direction, e.g., `GPIO14 / I2S1 BCK -> BCLK`.
- The main diagram must not place long text directly on Mermaid edge labels; use intermediate `net_*` label nodes to express short labels like `GPIO14 I2S1 BCK -> MAX98357.BCLK` to avoid text overlap on multiple edges.
- The main diagram must not render `alerts_sg` or notes subgraphs; notes should only be placed in `wiring_pins.md` or the HTML description area, and must not crowd the wiring body of `wiring.svg/png`.
- SVG/PNG must use a white or light background; do not rely on transparent backgrounds, otherwise lines and text will be unreadable in dark viewers.
- Multi-line interfaces like I2S, SPI, I2C, UART can be visually grouped as buses, but the pin-to-pin mapping for each line must be preserved.
- Multi-pin peripherals must not be compressed into comma-separated strings like `standalone.pin="14,32,33"`; they must use `components[]` + `connections[]`, or provide device-side `pins[]` mapping in `buses[]`.

Key fields:

- `meta.project`: Project name.
- `meta.generated_at`: Real ISO 8601 timestamp; do not use sample placeholder times.
- `meta.source_phase`: Use `generate` for the formal chain.
- `mcu.pins[]`: Includes actual used GPIO, power pins, ground pins, and fixed onboard pins.
- `components[]`: Component-level nodes. Recommended to include MCU, peripheral modules, LEDs, buttons, power-related modules.
- `connections[]`: Real electrical connections. Each item represents one wire and must include `from.component/from.pin` and `to.component/to.pin`.
- `buses[]`: I2C/SPI/UART/OneWire/CAN/I2S buses. Used for protocol grouping and compatibility with old renderers; must not replace the pin-to-pin facts in `connections[]`.
- When `project-manifest.json pinout` contains `i2s_*` records, success must include the corresponding I2S component, I2S bus, and the MCU GPIO to peripheral pin connection for each `i2s_bck/i2s_ws/i2s_data_in/i2s_data_out`; otherwise, it must be corrected or output as partial.
- `standalone[]`: Single-pin independent GPIO devices like LEDs, buzzers, buttons, relays. Only single pins are allowed; cannot be used for multi-pin components like I2S audio modules.
- `power[]`: 3.3V, 5V, Vin, GND power supply relationships.
- `alerts[]`: Conflict, safety, power, pull-up, current limit prompts.

Alert `msg` must be concise, recommended not to exceed 60 English characters, to avoid expanding the wiring diagram layout.

## phase_complete Output

On success, output:

```json
{
  "type": "phase_complete",
  "phase": "upy-wiring-plugin",
  "payload": {
    "phase": "upy-wiring-plugin",
    "result": "success",
    "next_phase": null,
    "source_phase": "upy-generate-plugin",
    "source_phase_complete_path": "sessions/<session_id>/phase_complete.upy_generate_plugin.json",
    "manifest_content": {
      "phase": "wiring",
      "wiring": {
        "json": "docs/wiring.json",
        "md": "docs/wiring.md",
        "html": "docs/wiring.html",
        "pins": "docs/wiring_pins.md",
        "svg": "docs/wiring.svg",
        "png": "docs/wiring.png"
      }
    },
    "artifacts": [
      {"type": "wiring_json", "path": "docs/wiring.json"},
      {"type": "wiring_markdown", "path": "docs/wiring.md"},
      {"type": "wiring_html", "path": "docs/wiring.html"},
      {"type": "wiring_pins", "path": "docs/wiring_pins.md"}
    ],
    "checks": {
      "wiring_schema": {"ok": true},
      "render_wiring": {"ok": true},
      "manifest_update": {"ok": true}
    },
    "render_result": {
      "json": {"ok": true, "path": "docs/wiring.json"},
      "md": {"ok": true, "path": "docs/wiring.md"},
      "html": {"ok": true, "path": "docs/wiring.html"},
      "pins": {"ok": true, "path": "docs/wiring_pins.md"},
      "svg": {"ok": true, "path": "docs/wiring.svg", "backend": "local_mermaid"},
      "png": {"ok": true, "path": "docs/wiring.png", "backend": "local_mermaid"}
    },
    "file_manifest": {
      "path": "sessions/<session_id>/wiring_file_manifest.json",
      "files": []
    },
    "session_state": {
      "path": "sessions/<session_id>/session_state.upy_wiring_plugin.json",
      "checkpoint": "phase_completed"
    },
    "warnings": [],
    "errors": []
  }
}
```

Hard requirements for `result=success`:

- `docs/wiring.json` exists and passes schema validation.
- `docs/wiring.md` exists.
- `docs/wiring.html` exists.
- `docs/wiring_pins.md` exists.
- `docs/wiring.svg` exists.
- `docs/wiring.png` exists.
- `payload.next_phase == null`.
- `payload.manifest_content.wiring` records the generated artifacts.
- `payload.artifacts[]` covers the successfully generated wiring outputs.
- `payload.file_manifest.files[]` covers all required artifacts and records `required=true`, `sha256`, `bytes`, `source`, `checkpoint`.
- `payload.session_state.checkpoint == "phase_completed"`.

Common cases for `result=partial`:

- Upstream generate phase_complete is missing or not success.
- Firmware source code is incomplete.
- `wiring.json` cannot pass schema.
- Blocking conflict between firmware and manifest requires user confirmation.
- SVG/PNG rendering fails, times out, or is denied.

Partial/failed must set `next_phase=null` and write to `errors` or `warnings`.

## User Confirmation Points

If network rendering of SVG/PNG is needed, first send:

```text
approval_request(approval_id="wiring_network_render")
```

User options:

- `render_all`: Allow mermaid.ink/CDN, generate md/html/pins/svg/png.
- `local_only`: Only allow local renderer; if local cannot generate SVG/PNG, output partial.
- `cancel`: Stop wiring, output partial.

If there are GPIO, address, or power conflicts between firmware and manifest, send:

```text
approval_request(approval_id="wiring_conflict_review")
```

Do not hide conflicts as success before user confirmation.

## Scripts

- `scripts/render_wiring_local.py`: Renderer migrated from the old `upy-wiring`. Input `docs/wiring.json`, output wiring Markdown, HTML, SVG, PNG, and pin table.
- `scripts/derive_wiring_topology.py`: Derives component-level topology from `project-manifest.json pinout`, forcibly generates `components[]`, `connections[]`, and I2S/I2C/SPI/UART bus mappings.
- `scripts/wiring_manifest.py`: Validates start_phase, upstream generate phase_complete, wiring phase_complete, and artifact path contracts.

## Session, Checkpoint, Retry, Cancel, and Timeout

Each run should write:

```text
<session_root>/session_state.upy_wiring_plugin.json
<session_root>/wiring_file_manifest.json
```

Recommended checkpoints:

| checkpoint | Meaning | Resumable Action |
|---|---|---|
| `started` | start_phase received | Re-validate input |
| `upstream_validated` | generate phase_complete validated | Continue reading project files |
| `inputs_read` | manifest and firmware read | Re-derive wiring.json |
| `wiring_json_written` | wiring.json written | Continue schema validation |
| `wiring_json_validated` | schema passed | Continue rendering |
| `artifacts_rendered` | wiring artifacts generated | Continue manifest update and file manifest |
| `manifest_updated` | project-manifest wiring field updated | Continue phase_complete |
| `phase_completed` | phase_complete output | Idempotently return existing result |
| `cancelled` | User cancelled | Do not automatically continue |
| `failed` | Blocking failure | On retry, resume from last_ok_artifact |

Recovery rules:

- When `checkpoint=phase_completed` and phase_complete validation passes, retry directly returns the existing result.
- When `docs/wiring.json` exists and schema passes, resume from the rendering phase, do not re-parse firmware.
- When SVG exists but PNG is missing, only retry PNG rendering.
- Retry must reuse the original `session_id` and stable `idempotency_key`, and increment `attempt`.
- Timeout must write to `last_error` and the current checkpoint.
- Cancellation must output `CANCELLED_BY_USER`, `next_phase=null`.

## Capability Negotiation and Permission Prompts

The formal plugin mode must have:

```json
{
  "capabilities": {
    "approval_request": true,
    "file_operation": true,
    "script_run": true,
    "checkpoint_resume": true,
    "cancellation": true,
    "retry": true,
    "timeout": true,
    "permission_prompt": true
  }
}
```

When capabilities are missing:

| Missing Capability | Handling |
|---|---|
| `file_operation` | Cannot execute in formal mode, output `CAPABILITY_UNAVAILABLE` |
| `script_run` | Cannot validate or render, output `CAPABILITY_UNAVAILABLE` |
| `approval_request` | Cannot request network rendering or conflict confirmation, default to conservative partial |
| `checkpoint_resume` | Can direct_test, but formal mode success is not recommended |
| `cancellation` | Must inform that it is not cancellable; long tasks still need timeout |
| `permission_prompt` | Cannot perform write file, script, or network operations requiring authorization |

Permission prompt scope:

- `file_operation(read)`: Read `project-manifest.json`, `firmware/**/*.py`, upstream phase_complete.
- `file_operation(write)`: Write `docs/wiring.*`, the wiring field of `project-manifest.json`, session_state, file_manifest; must not rewrite the root-level `updated_at` of `project-manifest.json` due to the wiring phase.
- `script_run`: Only allow whitelisted scripts `validate_json.py`, `derive_wiring_topology.py`, `render_wiring_local.py`, `wiring_manifest.py`.
- `network_rendering`: Must explicitly confirm whether to access mermaid.ink or CDN before generating SVG/PNG.
- `device_command`: Not needed for this phase; if a device command request is received, it should be rejected.

## Structured Errors

Error objects uniformly use:

```json
{
  "code": "WIRING_SCHEMA_INVALID",
  "severity": "blocking",
  "retryable": false,
  "message": "docs/wiring.json failed schema validation",
  "details": {
    "path": "docs/wiring.json",
    "validator": "wiring.schema.json"
  },
  "checkpoint": "wiring_json_written",
  "next_action": "fix wiring.json and rerun schema validation"
}
```

Common error codes:

| Error Code | Scenario |
|---|---|
| `PROTOCOL_UNSUPPORTED` | `protocol_version` not supported |
| `CAPABILITY_UNAVAILABLE` | Missing necessary protocol capability |
| `UPSTREAM_PHASE_MISSING` | generate phase_complete missing |
| `UPSTREAM_PHASE_INVALID` | Upstream not success or manifest incomplete |
| `PROJECT_MANIFEST_MISSING` | Project manifest missing |
| `FIRMWARE_NOT_FOUND` | Firmware source code missing |
| `WIRING_SCHEMA_INVALID` | wiring.json schema failed |
| `WIRING_CONFLICT_REQUIRES_REVIEW` | Firmware and manifest conflict requires user confirmation |
| `WIRING_IMAGE_RENDER_PERMISSION_DENIED` | User refused network and no local renderer |
| `WIRING_IMAGE_RENDER_TIMEOUT` | SVG/PNG rendering timeout |
| `WIRING_IMAGE_RENDER_FAILED` | SVG/PNG rendering failed |
| `FILE_PERMISSION_DENIED` | File read/write permission denied |
| `SCRIPT_PERMISSION_DENIED` | Script execution permission denied |
| `CANCELLED_BY_USER` | User cancelled |
| `IDEMPOTENCY_CONFLICT` | Input for the same idempotency key has changed |

## Final Checks

Before outputting the final response, at least confirm:

- The old `upy-wiring` has not been overwritten.
- Deploy-related directories have not been overwritten.
- `phase_complete.payload.next_phase` is `null`.
- All declared artifacts can be interpreted as wiring artifacts.
- If success declares SVG/PNG, they were actually rendered successfully or there is clear evidence.
