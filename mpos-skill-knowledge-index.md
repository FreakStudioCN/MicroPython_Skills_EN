# MicroPythonOS Skill Knowledge Index

Generated: 2026-07-14
Last Updated: 2026-07-20

This file is the master knowledge entry point for MicroPythonOS / LVGL / mpos-* skills formed during this conversation. It is not an execution guide for any specific skill, but an index for subsequent maintainers and Codex to quickly locate materials: which directories are sources, which files are reference documents, which APIs are extracted by scripts, and which external sites need re-synchronization.

The body uses Chinese; code, commands, paths, API names, and JSON field names remain in English.

## 1. Overall Conclusions

- `mpos-dev` should continue as the shared base layer for the `mpos-*` skill family, housing MicroPythonOS architecture, LVGL conventions, official docs topic references, API extraction scripts, and generated reference files.
- The current `mpos-*` main chain has been completed as: `mpos-plan-app` -> `mpos-analyze-app` -> `mpos-prepare-deps` -> `mpos-gen-app` -> `mpos-test-app` -> `mpos-package-app` -> `mpos-deploy-app` -> `mpos-publish-app`, plus the shared `mpos-dev`.
- All stage skills targeting a single App should uniformly maintain `<repo-root>/tmp/mpos-plan-app/<fullname>/plan_state.json` and `activity_log.jsonl`, and register stage artifacts via `mpos-plan-app/scripts/update_plan_state.py`, facilitating interruption, recovery, user requirement changes, and AI debugging.
- New Apps default to a flat layout: `MANIFEST.JSON`, `icon_64x64.png`, `assets/*.py`. The old `META-INF/MANIFEST.JSON` and `res/mipmap-mdpi/icon_64x64.png` are only read for compatibility and must generate a warning.
- The main repository `/home/leeqingshui/MicroPythonOS` should be kept as consistent as possible with upstream. Build, desktop simulator, web preview, and joint debugging tests default to isolated clones/worktrees/temporary copies; except for adding new Apps or explicit user permission, do not modify OS/build source code.
- The content of `docs.micropythonos.com` has been split into multiple topic-specific reference documents for task-based reading; this is not a verbatim full-text mirror. Whether all pages are covered should be audited against the sitemap/search index in `mpos-dev/reference/docs-site-index.md`.
- `web.micropythonos.com` contains WebAssembly/browser runtime information and should appear in both `mpos-dev/reference/docs-web-port.md` and the roadmap of the root-level analysis file `mpos-conversational-skills-analysis.md`.
- The source of LVGL API should be the compiled/generated MicroPython stub: `/home/leeqingshui/lvgl_micropython/lvgl.pyi`, not guessed directly from LVGL C header files.
- MicroPythonOS API extraction should only extract MicroPython-visible APIs: the import/call form of native MicroPython modules, module globals, type locals, and the Python API exported by `mpos.__all__`; do not treat internal implementation functions, low-level signatures, or implementation source files as public APIs.
- Machine retrieval should prioritize JSON; human reading and Codex quick scanning require MD. Currently, `mpos_api_summary.json`, `mpos-api-reference.md`, `lvgl_api_summary.json`, and `lvgl-api-reference.md` have been completed; mpos-related skills must fully read the API summary and cannot omit it based on task difficulty.

## 2. Local Root Directories

### `/home/leeqingshui/MicroPython_Skills`

This is the skill repository and the location of this new index.

Key files:

- `mpos-skill-knowledge-index.md`: The current file, the master knowledge index for this conversation.
- `mpos-conversational-skills-analysis.md`: Analysis of the mpos skill family split and conversational capabilities; previously required `https://web.micropythonos.com/` to be included here.
- `README.md`: Repository-level description.

Current skills related to MicroPythonOS:

- `mpos-dev/`: Shared foundational knowledge base, containing API references, docs references, and extraction scripts.
- `mpos-plan-app/`: Conversational entry point and state machine, responsible for stage orchestration, interruption recovery, invalidation list confirmation, and default handoff to publishing.
- `mpos-analyze-app/`: Translates natural language requirements into App identity, manifest draft, Activity/Service plan, dependency risks, and test/deployment plan.
- `mpos-prepare-deps/`: Prepares application-layer pure Python/MPY dependencies, caches search results, supports async/aio/uasyncio search strategies; synchronous libraries must be marked `sync_needs_adapter=true` for the generation stage to create non-blocking wrappers.
- `mpos-gen-app/`: Two-stage generation, update, and repair of App files; forces output of a plan and waits for confirmation, then immediately runs static gates such as manifest, syntax, MPY import, API usage, `make lint`, flake8, pylint, App-only change checks, and produces `generation_result.json`.
- `mpos-test-app/`: Only performs MPOS runtime smoke / optional Web Port checks for the target App, using MicroPythonOS built-in tools like `run_desktop.sh`, `mpos_controller.py`; does not own static lint/manifest/API gates but must verify they are recorded in `generation_result.json`.
- `mpos-package-app/`: Generates single App `.mpk`, `app_index_entry.json`, and `package_result.json`, defaulting to `stored` compression; can proceed with packaging even if tests are missing or fail, but must issue a warning.
- `mpos-deploy-app/`: Only handles deployment/preview paths, including desktop-preview, web-preview, device-copy, mpk-install, install-site, local-flash; first confirms physical device, serial port, and MicroPythonOS installation status; desktop/manual launch is an optional preview, not a smoke gate.
- `mpos-publish-app/`: Only handles upystore publishing guidance and validation; must read `package_result.json`, `app_test_result.json`, and `deploy_result.json`, and produce `publish_result.json`; does not log in or upload.

### `/home/leeqingshui/MicroPythonOS`

This is the main MicroPythonOS repository.

Key directories:

- `AGENTS.md`: Highest priority local engineering constraints and LVGL/MicroPythonOS notes.
- `internal_filesystem/`: Core directory mapping one-to-one to the device filesystem.
- `internal_filesystem/lib/mpos/`: MicroPythonOS Python framework code; `mpos.__all__` is an important source of public Python APIs.
- `internal_filesystem/apps/`: Installed Apps.
- `internal_filesystem/builtin/`: Built-in resources or built-in application related content.
- `c_mpos/src/`: Native MicroPython module implementation source code, such as `webcam`, `pdm_mic`, `adc_mic`, `qrdecode`, `rvswd`; reference output only shows MPY user-callable interfaces.
- `docs/`: Local docs source files.
- `scripts/`: Build, run, install, flash, deploy, controller, and other scripts; previously confirmed this is not the API extraction script directory.
- `tests/`: Syntax tests, unit tests, controller tests, etc.
- `lvgl_micropython/`: LVGL submodule within the MicroPythonOS repository.

Important rule: When official documentation examples conflict with the actual code in the current repository, prioritize the current repository and `AGENTS.md`.

### `/home/leeqingshui/lvgl_micropython`

This is the independent `lvgl_micropython` repository. Current LVGL API extraction should prioritize using:

- `lvgl.pyi`: The stub for the MicroPython LVGL API, the source-of-truth for the current API summary JSON.
- `gen/lvgl_api_gen_mpy.py`
- `gen/stub_gen.py`
- `gen/fixed_gen_json.py`
- `gen/api_gen/*`
- `stubs/*.pyi`
- `build/*.bin`

Note: Both `/home/leeqingshui/MicroPythonOS/lvgl_micropython` and `/home/leeqingshui/lvgl_micropython` exist. It has been clarified in the conversation that skill LVGL API extraction should target the independent repository's `/home/leeqingshui/lvgl_micropython/lvgl.pyi`, as it represents the current compiled/generated MicroPython binding API.

## 3. `mpos-dev` File Structure and Responsibilities

`/home/leeqingshui/MicroPython_Skills/mpos-dev` is the shared base layer.

Key files:

- `SKILL.md`: Entry point for the MicroPythonOS foundational development knowledge base, containing architecture, LVGL conventions, native MicroPython module quick reference, and reference routing.
- `scripts/extract_lvgl_api.py`: Extracts LVGL MicroPython API from `lvgl.pyi`, currently outputs `reference/lvgl_api_summary.json` and `reference/lvgl-api-reference.md`.
- `scripts/extract_mpos_api.py`: Extracts MicroPython-visible APIs from the MicroPythonOS main repository, currently outputs `reference/mpos_api_summary.json` and `reference/mpos-api-reference.md`.

Current files in `reference/`:

- `docs-site-index.md`: Docs site coverage index, recording sitemap/search index coverage status.
- `docs-app-model.md`: App model, Activity, Service, Intent, local layout override.
- `docs-packaging.md`: `.mpk`, Store, `upystore`, BadgeHub, manifest, app index.
- `docs-frameworks.md`: System managers, framework API, LVGL usage rules.
- `docs-deploy-targets.md`: Linux desktop, browser, device, firmware, QEMU, target devices.
- `docs-os-development.md`: Build, test, port, release, OS-level development.
- `docs-web-port.md`: WebAssembly/browser runtime, `web.micropythonos.com`, web target.
- `mpos-api-reference.md`: Human-readable reference for MicroPythonOS mpy-visible APIs.
- `mpos_api_summary.json`: Machine-readable index of MicroPythonOS user-callable APIs, containing `native_modules`, root exports, full-source public APIs, `source_index`, and `symbols[]`.
- `lvgl-api-reference.md`: Human-readable reference for LVGL MicroPython APIs.
- `lvgl_api_summary.json`: Machine-readable summary of LVGL MicroPython APIs.

## 4. mpos Stage Handoff Artifacts

Unified handoff directory:

```text
<repo-root>/tmp/mpos-plan-app/<fullname>/
  plan_state.json
  activity_log.jsonl
```

Standard artifact keys:

- `analysis_result`: Result of `mpos-analyze-app` requirements analysis.
- `dependency_handoff`: Dependency files, cache, sync adaptation requirements from `mpos-prepare-deps`.
- `generation_result`: File writing and static gate records from `mpos-gen-app`.
- `app_test_result`: Runtime smoke/Web Port records from `mpos-test-app`.
- `package_result`: MPK, app_index_entry, and packaging warnings from `mpos-package-app`.
- `deploy_result`: Desktop/web/device/MPK install preview or deployment records from `mpos-deploy-app`.
- `publish_result`: upystore version comparison, store metadata, and publishing handoff from `mpos-publish-app`.

Maintenance rules:

- Do not manually write `plan_state.json` or `activity_log.jsonl`; use `mpos-plan-app/scripts/update_plan_state.py record/discover/invalidate`.
- When the user says "continue/resume/next step" after interruption, first read or rebuild `plan_state.json`; do not start analysis from scratch.
- When the user modifies requirements, `mpos-plan-app` must first list the invalidated artifacts and get user confirmation; the two-stage confirmation of `mpos-gen-app` remains mandatory.
- Without a physical board, `desktop-preview` or `web-preview` in `deploy_result.json` can satisfy the publish prerequisite; with hardware, prioritize `mpk-install` for real-device release verification.

## 5. External Sites and Indexes

This conversation involves the following external resource entry points:

- `http://docs.micropythonos.com/`: MicroPythonOS official docs main site.
- `http://docs.micropythonos.com/sitemap.xml`: Used to audit docs page coverage.
- `https://docs.micropythonos.com/search/search_index.json`: Used to audit search index coverage.
- `https://web.micropythonos.com/`: MicroPythonOS browser/WebAssembly runtime entry point.
- `https://install.micropythonos.com/`: Installation entry point.
- `https://upystore.io/`: App store/package index related entry point.
- `https://upystore.io/app_index.json`: App index JSON.

Recommended maintenance approach:

- Audit results for site directory and search index go into `mpos-dev/reference/docs-site-index.md`.
- Content broken down by topic goes into `mpos-dev/reference/docs-*.md`.
- Operation mode, limitations, and deployment targets for `web.micropythonos.com` go into `docs-web-port.md`.
- `upystore.io`, `.mpk`, `app_index.json` go into `docs-packaging.md`.
- Fetch external sites only when re-synchronization is needed; do not mix temporary curl output into skill files.

## 6. Docs Splitting Status

Current docs have been split by task topic into `mpos-dev/reference/`. These files should be loaded as on-demand references, not crammed entirely into `SKILL.md`.

Reading routing:

- Generate/modify an App: First read `docs-app-model.md`, then read `docs-frameworks.md` as needed.
- Use system services, managers, notifications, downloads, audio, sensors: Read `docs-frameworks.md`.
- Packaging, store, manifest, `.mpk`: Read `docs-packaging.md`.
- Linux desktop, device installation, firmware, QEMU, target devices: Read `docs-deploy-targets.md`.
- Modify OS kernel, build system, testing, porting: Read `docs-os-development.md`.
- Browser runtime, WebAssembly, `web.micropythonos.com`: Read `docs-web-port.md`.
- Determine if docs are missing pages: Read `docs-site-index.md`.

Current important numbers:

- `docs-site-index.md` records approximately 61 pages from the sitemap and approximately 977 search items from the search index.
- These references are in Chinese; code, JSON, paths, and API names remain in English.

## 7. API Extraction Status

### LVGL API

Current outputs:

- `/home/leeqingshui/MicroPython_Skills/mpos-dev/reference/lvgl_api_summary.json`
- `/home/leeqingshui/MicroPython_Skills/mpos-dev/reference/lvgl-api-reference.md`

Current source:

- `/home/leeqingshui/lvgl_micropython/lvgl.pyi`

Current script:

```bash
python3 /home/leeqingshui/MicroPython_Skills/mpos-dev/scripts/extract_lvgl_api.py --lvgl-micropython-dir /home/leeqingshui/lvgl_micropython
```

Confirmed in conversation:

- LVGL extraction should target the MicroPython binding API, i.e., the mpy API.
- `lvgl.pyi` is the most suitable input because it comes from the current `lvgl_micropython` generation result.
- The current JSON structure includes `source`, `generated_at`, `generator`, `counts`, `type_aliases`, `enums`, `data_classes`, `widgets`, `functions`, `symbols[]`.
- Current statistics: 60 type aliases, 90 enum classes, 873 enum members, 79 data classes, 41 widgets, 247 standalone functions, 1016 widget methods, 1369 data class methods, 3715 symbols.
- Stub type aliases like `*_t = int` are no longer written into `symbols[]`, only placed in `type_aliases[]`, and `runtime_enum` mappings are provided where possible, e.g., `display_render_mode_t -> lv.DISPLAY_RENDER_MODE`, `grad_dir_t -> lv.GRAD_DIR`, `event_code_t -> lv.EVENT`, `fs_whence_t -> lv.FS_SEEK`. AI-generated code should use runtime enum members like `lv.DISPLAY_RENDER_MODE.PARTIAL`, `lv.GRAD_DIR.VER`, `lv.EVENT.CLICKED`.

Remaining notes:

- JSON is suitable for machine retrieval; MD is suitable for human and Codex quick scanning; both are generated by the same script.
- The `description` field cannot be hardcoded by the script; only descriptions from docstrings, comments, official documentation, or manual overrides should be written.

### MicroPythonOS API

Current outputs:

- `/home/leeqingshui/MicroPython_Skills/mpos-dev/reference/mpos-api-reference.md`
- `/home/leeqingshui/MicroPython_Skills/mpos-dev/reference/mpos_api_summary.json`

Current script:

```bash
python3 /home/leeqingshui/MicroPython_Skills/mpos-dev/scripts/extract_mpos_api.py --mpos-dir /home/leeqingshui/MicroPythonOS
```

Current extraction scope:

- Native MicroPython modules: `MP_REGISTER_MODULE`, module globals, type locals; final output only retains the Python form that users can import/call.
- Python root exports: Public APIs exposed by `mpos.__all__` in `internal_filesystem/lib/mpos/__init__.py`.
- Python full-source public APIs: Non-underscore public classes/functions/constants/variables in `internal_filesystem/lib/mpos/**/*.py`, with `availability`/`aliases` markers for root exports.

Confirmed in conversation:

- This is an index of MicroPython user-callable APIs; it does not capture private underscore implementations.
- After the current regeneration, `mpos-api-reference.md` and `mpos_api_summary.json` are both from the MPY interface perspective, not containing native implementation source file paths or low-level signatures.
- Currently includes 105 Python files, 106 public classes, 164 public functions, 297 public constants/variables.
- Current root exports include 38 `mpos` classes, 36 `mpos` functions, 1 variable, 11 exported submodules, with `missing` being 0.
- Current native MicroPython modules include 5 modules, 3 classes, 23 methods, 8 module-level functions, 4 constants.
- `rvswd` should be represented as the `RVSWD` class, its methods, and constants.
- The current actual MPY interface for `webcam` is module-level functions plus the `Webcam` type; JSON/MD have been indexed according to this structure: `webcam.init(...)` returns a `Webcam` handle, other module-level functions receive this handle.

Remaining notes:

- `description`, `notes`, `examples` should only be filled from docstrings, comments, official documentation, or manual overrides; keep as `null` or empty arrays when no source is available.

## 8. Recommendations for JSON and Markdown

Both types of artifacts are currently retained:

- JSON: For scripts, agent retrieval, automatic validation, and precise field queries.
- Markdown: For human reading, Codex quick scanning, and manual confirmation before tasks.

Both are located in:

- `/home/leeqingshui/MicroPython_Skills/mpos-dev/reference/`

Completed files:

- `reference/mpos_api_summary.json`
- `reference/lvgl-api-reference.md`

Current JSON top-level fields:

```json
{
  "source": {},
  "generated_at": "",
  "generator": "",
  "counts": {},
  "symbols": []
}
```

Recommended `symbols[]` fields:

```json
{
  "kind": "",
  "name": "",
  "fqname": "",
  "module": "",
  "parent": null,
  "signature": "",
  "params": [],
  "returns": null,
  "description": null,
  "description_source": null,
  "notes": [],
  "examples": [],
  "source_path": "",
  "source_line": null,
  "availability": null,
  "aliases": [],
  "deprecated": false
}
```

Field descriptions:

- `source`: Input source, e.g., `lvgl.pyi`, `internal_filesystem/lib/mpos/__init__.py`; implementation source files of native modules are not written into the user-side reference.
- `generated_at`: Generation time, useful for determining if outdated.
- `generator`: Generator script name and version information.
- `counts`: Statistics by module, class, function, enum, method.
- `kind`: `module`, `class`, `method`, `function`, `enum`, `constant`, `data_class`, `widget`, etc.
- `fqname`: Fully qualified name, e.g., `lv.button.set_text` or `mpos.AppManager`.
- `signature`, `params`, `returns`: Fill only when reliably extractable from stubs, AST, or manually maintained native MPY signature tables.
- `description`: API description. Do not fabricate; set to `null` if no source exists.
- `description_source`: `docstring`, `comment`, `official_docs`, `manual_override`, `stub`, etc.
- `notes`: Binding differences, pitfalls, AGENTS rules, compatibility reminders.
- `examples`: Short examples; prioritize from official docs or manual maintenance; do not auto-generate misleading examples.
- `source_path`, `source_line`: Fill when Python/stub API is locatable; keep `null` for native module symbols to avoid mixing implementation details into the user-side API reference.

## 9. `AGENTS.md` Basic Rules

Key rules merged from `/home/leeqingshui/MicroPythonOS/AGENTS.md` into the skill:

- Prioritize using root `Makefile` targets: `make build-mpos-unix`, `make syntax-tests`, `make unittest-tests`, `make tests`, `make lint`, `make lint-fix`.
- MicroPythonOS code changes must pass `make lint`.
- When running the desktop version, use `timeout -s 9 30 ./scripts/run_desktop.sh`.
- Write temporary debugging scripts to the repository root `tmp/`, not `/tmp`.
- Kill processes using `killall <name>`, not `pkill -f <pattern>`.
- Python formatting follows `ruff.toml`; current quote style is double quotes.
- After installing an App to the device, `AppManager.refresh_apps()` must be called before `start_app()`.
- `self.appFullName` is automatically set by `ActivityNavigator`; prioritize using it for in-App persistence and similar scenarios; do not hardcode package names.
- `Soft reset` is unreliable in the current `lvgl_micropython` / MicroPythonOS combination; use `machine.reset()`.

Key LVGL rules:

- `import lvgl as lv`.
- Use `lv.screen_active()`, not `lv.scr_act()`.
- Do not hardcode resolution; use adaptive methods like `lv.pct(100)`.
- Use `button`, `image`; do not use old names `btn`, `img`.
- `lv.EVENT.VALUE_CHANGED`, not `lv.EVENT_VALUE_CHANGED`.
- `lv.obj.FLAG.CLICKABLE`, not `lv.OBJ_FLAG.CLICKABLE`.
- Hide/show using `.add_flag(lv.obj.FLAG.HIDDEN)` / `.remove_flag(lv.obj.FLAG.HIDDEN)`.
- New labels must explicitly call `label.set_text("")`, otherwise they default to displaying `"Text"`.
- After `style_obj = lv.style_t()`, must call `style_obj.init()` before using setters.
- LVGL 9 style setters only pass the value; the selector is placed in `obj.add_style(style, selector)`.
- Event callbacks require an event parameter; use `event=None` when a method serves as both a callback and a regular method.
- In callbacks, prefer using `event.get_target_obj()`.
- LVGL object wrappers do not support arbitrary Python attribute assignment; use closures/lambdas or parallel lists for associated data.

## 10. Codex/Approval Related Records

This conversation also touched on the Codex approval mode. This content belongs to runtime security configuration and is not recommended to be written into the default flow of a MicroPythonOS skill.

Maintenance recommendations:

- Skill documents should not default to requiring "skip all approvals" or "disable sandbox".
- When network access, writing outside the workspace, or running potentially dangerous commands is needed, confirm separately according to the approval mechanism of the current Codex session.
- If Codex usage instructions are specifically organized later, they should be placed in a separate Codex tool documentation file, not mixed with `mpos-dev` development rules.

## 11. Future To-Do Items

- Add manually maintained `description`, `notes`, and `examples` for commonly used APIs, and mark `description_source: "manual_override"`.
- If adding manual examples for `webcam`, continue processing according to the actual MPY interface structure: module-level functions plus the `Webcam` type; examples should avoid writing non-existent methods.
- When re-synchronizing external docs, first update `docs-site-index.md`, then update the corresponding topic-specific reference.
- When maintaining `mpos-*` skills, continue to adhere to progressive disclosure: `SKILL.md` only retains stage flows and resource routing; details go into `reference/`; deterministic actions go into `scripts/`; do not copy and paste entire documents.
