---
name: mpos-analyze-app
description: Analyze MicroPythonOS App ideas directly or when invoked by mpos-plan-app. Use to turn natural-language MPOS App requests into requirements, default app identity, manifest draft, Activity/Service plan, MPOS/LVGL API plan, dependency risk, test/deploy plan, mandatory MicroPythonOS resource links, and a JSON handoff before code generation.
---

# MicroPythonOS App Requirements Analysis

## Role

Analyze one or more user statements about a MicroPythonOS App idea into a stable state that can be handed off to downstream skills. Supports two entry points:

- The user directly requests analysis, planning, or confirmation of an MPOS App.
- `mpos-plan-app` invokes this skill as the analyze phase of a conversational pipeline.

This skill performs analysis and handoff only; it does not write code, download drivers, package, install apps, flash firmware, or upload to upystore.

## Unified Project Log

After completing the analysis and producing `analysis_result.json`, it must be recorded in the project state directory:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-plan-app/scripts/update_plan_state.py record \
  --repo <repo-root> \
  --fullname <fullname> \
  --skill mpos-analyze-app \
  --phase analyze \
  --result <success|partial|failed> \
  --artifact analysis_result=<analysis_result.json> \
  --next-skill <handoff.next_skill-or-null> \
  --event "Analyzed requirements and produced analysis_result.json"
```

If `fullname` is only a suggested value, still create `tmp/mpos-plan-app/<fullname>/plan_state.json` using that suggestion, so it can be tracked during subsequent confirmation or renaming.

## Required Context

First load `mpos-dev`. Read as needed during analysis:

- App/Activity/Service/Intent: `mpos-dev/reference/docs-app-model.md`
- System manager, persistence, networking, audio, camera, sensors, background tasks: `mpos-dev/reference/docs-frameworks.md`
- MPOS API precise index: `mpos-dev/reference/mpos_api_summary.json`
- LVGL API precise index: `mpos-dev/reference/lvgl_api_summary.json`
- Target device, OS installation, desktop emulation, Web runtime: `mpos-dev/reference/docs-deploy-targets.md`
- Browser emulation details: `mpos-dev/reference/docs-web-port.md`
- MPK, AppStore, upystore: `mpos-dev/reference/docs-packaging.md`

API decisions should prioritize the JSON files. LVGL `type_aliases[]` only explains signature types; they are not runtime APIs that can be generated.

## Fixed Resource Entry Points

Every user-visible output must display these entry points; the JSON `resource_links[]` must also include them:

- Official documentation: `https://docs.micropythonos.com/`
- UpyStore: `https://upystore.io/`
- Install MicroPythonOS: `https://install.micropythonos.com/`
- Browser emulation: `https://web.micropythonos.com/`

These links are fixed entry points, not blocking prerequisites. Only ask "Is MicroPythonOS already installed?" when the user wants to run on real hardware, the device status is unknown, or they explicitly state the OS is not installed; recommend the installer if not installed. `web.micropythonos.com` can be used for quick browser emulation/smoke testing, but cannot replace Linux SDL desktop emulation or real hardware verification.

## Workflow

1. Read the user's requirements and any existing context. Preserve the app name, features, target device, hardware, and publishing intent explicitly specified by the user.
2. Generate a default app identity. If `fullname` is missing, suggest `com.micropythonos.<slug>` based on the feature name; if `name`, `category`, or `version` is missing, provide reasonable defaults without blocking.
3. Split feature boundaries: MVP, future features, non-goals, risk points.
4. Determine the app structure: which Activities are needed, whether a Service is required, and whether `boot_completed`, Intents, persistence, or background tasks are needed.
5. Determine if built-in APIs are sufficient: prioritize using MPOS managers/frameworks and LVGL MicroPython APIs; only flag whether external drivers are needed — do not search for or generate driver implementations at this stage.
6. Produce a test plan: syntax, manifest, standard unittest, GraphicalTestCase, desktop emulation, Web smoke, device hardware verification.
7. Produce a deployment/runtime plan: desktop first; Web for preview; confirm OS installation before real device; separate app installation from firmware flashing.
8. Only ask blocking questions. The analysis phase can proceed with default values; only block when code generation is imminent and essential identity, hardware, or target constraints are missing.
9. Output a Markdown summary and mandatory JSON. The JSON should match `templates/analysis_result.json` and be validatable with `scripts/validate_analysis_json.py`.

## Output Requirements

User-visible output in this order:

1. `MicroPythonOS Entry Points`: List the four fixed links.
2. `Analysis Summary`: One sentence describing the app goal and default identity.
3. `Features and Boundaries`: MVP, future, non-goals.
4. `Implementation Plan`: Activity/Service, framework/LVGL/API, dependency assessment.
5. `Testing and Runtime`: Test plan, desktop/Web/device paths.
6. `Items to Confirm`: Only list truly blocking questions; write "No blocking issues" if none.
7. `JSON`: A fenced `json` code block containing the complete analysis object.

## JSON Contract

Use `templates/analysis_result.json` as the field template. Key requirements:

- `schema_version` is fixed to `"mpos-analyze-v1"`.
- `phase` is fixed to `"analyze"`.
- `result` uses `"success"`, `"partial"`, or `"failed"`.
- `resource_links[]` must contain the four fixed URLs.
- `app.fullname` can be a suggested value; still provide a usable default when unknown, do not leave it empty due to lack of user confirmation.
- `manifest_draft.activities[]` and `services[]` use full objects: `classname`, `entrypoint`, `intent_filters`.
- `entrypoint` must include `.py`, recommended to use `assets/main.py` or `assets/service.py`.
- `app_structure.manifest` for new apps defaults to root directory `MANIFEST.JSON`.
- `app_structure.icon` for new apps defaults to root directory `icon_64x64.png`.
- Legacy `META-INF/MANIFEST.JSON` and `res/mipmap-mdpi/icon_64x64.png` are only used as compatibility paths when analyzing an existing legacy app.
- `dependency_plan.builtin_api_sufficient` and `external_driver_required` are core judgments at this stage.
- `blocking_questions[]` only contains questions that block downstream.
- `handoff.next_skill` recommended values: `mpos-gen-app`, `mpos-prepare-deps`, `mpos-test-app`, `mpos-package-app`, `mpos-deploy-app`, `mpos-publish-app`, or `mpos-plan-app`.

Validation command:

```bash
python3 /home/leeqingshui/MicroPython_Skills/mpos-analyze-app/scripts/validate_analysis_json.py \
  /home/leeqingshui/MicroPython_Skills/mpos-analyze-app/templates/analysis_result.json
```

## Downstream Routing

- Built-in APIs sufficient, requirements clear: `handoff.next_skill = "mpos-gen-app"`.
- External Python driver, component documentation, or dependency organization needed: `mpos-prepare-deps`.
- User only wants to verify an existing app: `mpos-test-app`.
- User mentions running, emulation, real device installation, or flashing: `mpos-deploy-app`, and clarify that app installation is not firmware flashing.
- User mentions MPK, AppStore, upystore, or publishing: `mpos-package-app` or `mpos-publish-app`.
- Multi-stage orchestration from requirements to publishing needed: hand back to `mpos-plan-app`.

## Boundaries

- Do not generate or modify `internal_filesystem/apps/<fullname>/`.
- Do not invoke driver downloads, datasheet extraction, or niche driver generation.
- Do not treat string-type `activities` from upystore seed data as the new manifest format.
- Do not require the user to read docs or install the OS first to complete the analysis.
- Do not describe `web.micropythonos.com` as an installer or publishing site.
- Do not describe `install.micropythonos.com` as an app installation tool; it is the OS/firmware installation entry point.
