# MicroPythonOS Skill Knowledge Index

Generated: 2026-07-14

This file serves as the central entry point for MicroPythonOS / LVGL / mpos-* skill knowledge formed during this conversation. It is not the execution guide for any specific skill, but an index for subsequent maintainers and Codex to quickly locate materials: which directories are sources, which files are reference documents, which APIs are extracted by scripts, and which external sites need re-synchronization.

The body uses Chinese; code, commands, paths, API names, and JSON field names remain in English.

## 1. Overall Conclusions

- `mpos-dev` should continue as the shared base layer for the `mpos-*` skill family, housing MicroPythonOS architecture, LVGL conventions, topic-specific references from official docs, API extraction scripts, and generated reference files.
- The content of `docs.micropythonos.com` has been split into multiple topic-specific reference documents for task-based reading; this is not a verbatim full-text mirror. Whether all pages are covered should be audited against the sitemap/search index in `mpos-dev/reference/docs-site-index.md`.
- `web.micropythonos.com` contains WebAssembly/browser runtime materials and should appear in both `mpos-dev/reference/docs-web-port.md` and the roadmap of the root-level analysis file `mpos-conversational-skills-analysis.md`.
- The source of LVGL API should be the compiled/generated MicroPython stub: `/home/leeqingshui/lvgl_micropython/lvgl.pyi`, not guessed directly from LVGL C header files.
- MicroPythonOS API extraction should only extract MicroPython-visible APIs: the import/call form of native MicroPython modules, module globals, type locals, and the Python API exported by `mpos.__all__`; do not treat internal implementation functions, low-level signatures, or implementation source files as public APIs.
- Machine retrieval should prioritize JSON; human reading and Codex quick scanning require MD. Currently, `mpos_api_summary.json`, `mpos-api-reference.md`, `lvgl_api_summary.json`, and `lvgl-api-reference.md` have been completed.

## 2. Local Root Directories

### `/home/leeqingshui/MicroPython_Skills`

This is the skill repository and the directory where this new index resides.

Key files:

- `mpos-skill-knowledge-index.md`: This file, the overall knowledge index for this conversation.
- `mpos-conversational-skills-analysis.md`: Analysis of the mpos skill family split and conversational capabilities; previously requested to also include `https://web.micropythonos.com/`.
- `README.md`: Repository-level description.

Current skills related to MicroPythonOS:

- `mpos-dev/`: Shared knowledge base, containing API references, docs references, and extraction scripts.
- `mpos-gen-app/`: Used when generating or modifying MicroPythonOS Apps.
- `mpos-debug-app/`: Used when debugging MicroPythonOS Apps.
- `mpos-test-app/`: Used when testing MicroPythonOS Apps.

`mpos-dev/SKILL.md` also mentions these planned/route names: `mpos-package-app`, `mpos-deploy-app`, `mpos-publish-app`. These directories were not found during the top-level directory check. If the packaging, deployment, and publishing workflows are to be completed later, they should be added following this naming convention.

### `/home/leeqingshui/MicroPythonOS`

This is the main MicroPythonOS repository.

Key directories:

- `AGENTS.md`: Highest priority local project constraints and LVGL/MicroPythonOS notes.
- `internal_filesystem/`: Core directory that maps one-to-one to the device filesystem.
- `internal_filesystem/lib/mpos/`: MicroPythonOS Python framework code; `mpos.__all__` is an important source of public Python APIs.
- `internal_filesystem/apps/`: Installed Apps.
- `internal_filesystem/builtin/`: Built-in resources or built-in application related content.
- `c_mpos/src/`: Source code for native MicroPython module implementations, such as `webcam`, `pdm_mic`, `adc_mic`, `qrdecode`, `rvswd`; reference output only shows MPY user-callable interfaces.
- `docs/`: Local docs source files.
- `scripts/`: Scripts for building, running, installing, flashing, deploying, controllers, etc.; previously confirmed this is not the directory for API extraction scripts.
- `tests/`: Syntax tests, unit tests, controller tests, etc.
- `lvgl_micropython/`: LVGL submodule within the MicroPythonOS repository.

Important rule: When official documentation examples conflict with the actual code in the current repository, prioritize the current repository and `AGENTS.md`.

### `/home/leeqingshui/lvgl_micropython`

This is the standalone `lvgl_micropython` repository. Current LVGL API extraction should prioritize using files from here:

- `lvgl.pyi`: The stub for MicroPython LVGL API, the source-of-truth for the current API summary JSON.
- `gen/lvgl_api_gen_mpy.py`
- `gen/stub_gen.py`
- `gen/fixed_gen_json.py`
- `gen/api_gen/*`
- `stubs/*.pyi`
- `build/*.bin`

Note: Both `/home/leeqingshui/MicroPythonOS/lvgl_micropython` and `/home/leeqingshui/lvgl_micropython` exist. It has been clarified in the conversation that the skill's LVGL API extraction should target `/home/leeqingshui/lvgl_micropython/lvgl.pyi` from the standalone repository, as it represents the current compiled/generated MicroPython binding API.

## 3. `mpos-dev` File Structure and Responsibilities

`/home/leeqingshui/MicroPython_Skills/mpos-dev` is the shared base layer.

Key files:

- `SKILL.md`: Entry point for the MicroPythonOS basic development knowledge base, including architecture, LVGL conventions, native MicroPython module quick reference, and reference routing.
- `scripts/extract_lvgl_api.py`: Extracts LVGL MicroPython API from `lvgl.pyi`, currently outputs `reference/lvgl_api_summary.json` and `reference/lvgl-api-reference.md`.
- `scripts/extract_mpos_api.py`: Extracts MicroPython-visible APIs from the MicroPythonOS main repository, currently outputs `reference/mpos_api_summary.json` and `reference/mpos-api-reference.md`.

Current files in `reference/`:

- `docs-site-index.md`: Docs site coverage index, recording sitemap/search index coverage status.
- `docs-app-model.md`: App model, Activity, Service, Intent, local layout override.
- `docs-packaging.md`: `.mpk`, Store, `upystore`, BadgeHub, manifest, app index.
- `docs-frameworks.md`: System managers, framework API, LVGL usage rules.
- `docs-deploy-targets.md`: Linux desktop, browser, device, firmware, QEMU, target devices.
- `docs-os-development.md`: Building, testing, porting, releasing, OS-level development.
- `docs-web-port.md`: WebAssembly/browser runtime, `web.micropythonos.com`, web target.
- `mpos-api-reference.md`: Human-readable reference for MicroPythonOS mpy-visible APIs.
- `mpos_api_summary.json`: Machine-readable index of MicroPythonOS user-callable APIs, containing `native_modules`, root exports, full source public APIs, `source_index`, and `symbols[]`.
- `lvgl-api-reference.md`: Human-readable reference for LVGL MicroPython APIs.
- `lvgl_api_summary.json`: Machine-readable summary of LVGL MicroPython APIs.

## 4. External Sites and Indexes

This conversation involves the following external material entry points:

- `http://docs.micropythonos.com/`: MicroPythonOS official docs main site.
- `http://docs.micropythonos.com/sitemap.xml`: Used for auditing docs page coverage.
- `https://docs.micropythonos.com/search/search_index.json`: Used for auditing search index coverage.
- `https://web.micropythonos.com/`: MicroPythonOS browser/WebAssembly runtime entry point.
- `https://install.micropythonos.com/`: Installation entry point.
- `https://upystore.io/`: App store/package index related entry point.
- `https://upystore.io/app_index.json`: App index JSON.

Suggested maintenance approach:

- Audit results for site directory and search index should be placed in `mpos-dev/reference/docs-site-index.md`.
- Content broken down by topic should be placed in `mpos-dev/reference/docs-*.md`.
- The operation mode, limitations, and deployment targets of `web.micropythonos.com` should be placed in `docs-web-port.md`.
- `upystore.io`, `.mpk`, `app_index.json` should be placed in `docs-packaging.md`.
- Fetch external sites only when re-synchronization is needed; do not mix temporary curl output into skill files.

## 5. Docs Splitting Status

Current docs have been split into `mpos-dev/reference/` by task topic. These files should serve as on-demand loaded references, not crammed entirely into `SKILL.md`.

Reading routes:

- Generate/modify App: Read `docs-app-model.md` first, then `docs-frameworks.md` as needed.
- Use system services, managers, notifications, downloads, audio, sensors: Read `docs-frameworks.md`.
- Packaging, store, manifest, `.mpk`: Read `docs-packaging.md`.
- Linux desktop, device installation, firmware, QEMU, target devices: Read `docs-deploy-targets.md`.
- Modify OS kernel, build system, testing, porting: Read `docs-os-development.md`.
- Browser runtime, WebAssembly, `web.micropythonos.com`: Read `docs-web-port.md`.
- Determine if docs pages are missing: Read `docs-site-index.md`.

Current important numbers:

- `docs-site-index.md` records approximately 61 pages in the sitemap and approximately 977 search items in the search index.
- These references are in Chinese; code, JSON, paths, and API names remain in English.

## 6. API Extraction Status

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

It has been confirmed in the conversation:

- LVGL should extract the MicroPython binding API, i.e., the mpy API.
- `lvgl.pyi` is the most suitable input because it comes from the generation result of the current `lvgl_micropython`.
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
- Python root export: Public API exposed by `mpos.__all__` in `internal_filesystem/lib/mpos/__init__.py`.
- Python full source public API: Non-underscore public classes/functions/constants/variables in `internal_filesystem/lib/mpos/**/*.py`, with `availability`/`aliases` tags for root exports.

It has been confirmed in the conversation:

- This is an index of MicroPython user-callable APIs; it does not capture private underscore implementations.
- After the current regeneration, both `mpos-api-reference.md` and `mpos_api_summary.json` are from the MPY interface perspective, not containing native implementation source file paths or underlying signatures.
- Currently includes 105 Python files, 106 public classes, 164 public functions, 297 public constants/variables.
- Current root exports include 38 `mpos` classes, 36 `mpos` functions, 1 variable, 11 exported submodules, `missing` is 0.
- Current native MicroPython modules include 5 modules, 3 classes, 23 methods, 8 module-level functions, 4 constants.
- `rvswd` should be represented as `RVSWD` class, methods, and constants.
- The current actual MPY interface for `webcam` is module-level functions plus a `Webcam` type; JSON/MD has been indexed according to this structure: `webcam.init(...)` returns a `Webcam` handle, other module-level functions receive this handle.

Remaining notes:

- `description`, `notes`, `examples` should only be filled from docstrings, comments, official documentation, or manual overrides; keep as `null` or empty arrays if no source is available.

## 7. Recommendations for JSON and Markdown

Both types of artifacts are currently retained:

- JSON: For scripts, agent retrieval, automated validation, and precise field queries.
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

- `source`: Input source, e.g., `lvgl.pyi`, `internal_filesystem/lib/mpos/__init__.py`; implementation source files for native modules are not written into the user-side reference.
- `generated_at`: Generation time, useful for determining if outdated.
- `generator`: Generator script name and version information.
- `counts`: Statistics by module, class, function, enum, method.
- `kind`: `module`, `class`, `method`, `function`, `enum`, `constant`, `data_class`, `widget`, etc.
- `fqname`: Fully qualified name, e.g., `lv.button.set_text` or `mpos.AppManager`.
- `signature`, `params`, `returns`: Fill only when reliably extractable from stubs, AST, or manually maintained native MPY signature tables.
- `description`: API description. Do not fabricate; fill `null` if no source is available.
- `description_source`: `docstring`, `comment`, `official_docs`, `manual_override`, `stub`, etc.
- `notes`: Binding differences, pitfalls, AGENTS rules, compatibility reminders.
- `examples`: Short examples; prioritize from official docs or manual maintenance; do not auto-generate misleading examples.
- `source_path`, `source_line`: Fill when locatable for Python/stub APIs; keep `null` for native module symbols to avoid mixing implementation details into the user-side API reference.

## 8. `AGENTS.md` Basic Rules

Key rules merged from `/home/leeqingshui/MicroPythonOS/AGENTS.md` into the skill:

- Prioritize using root directory `Makefile` targets: `make build-mpos-unix`, `make syntax-tests`, `make unittest-tests`, `make tests`, `make lint`, `make lint-fix`.
- Code changes to MicroPythonOS must pass `make lint`.
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
- After `style_obj = lv.style_t()`, `style_obj.init()` must be called before using setters.
- LVGL 9 style setters only pass the value; the selector is placed in `obj.add_style(style, selector)`.
- Event callbacks require an event parameter; use `event=None` when a method serves as both a callback and a regular method.
- Prefer using `event.get_target_obj()` in callbacks.
- LVGL object wrappers do not support arbitrary Python attribute assignment; use closures/lambdas or parallel lists for associated data.

## 9. Codex/Approval Related Records

This conversation also touched upon the Codex approval mode. This content pertains to runtime security configuration and is not recommended to be written into the default workflow of any MicroPythonOS skill.

Maintenance suggestions:

- Skill documentation should not default to requiring "skip all approvals" or "disable sandbox".
- When network access, writing outside the workspace, or running potentially dangerous commands is needed, confirm separately according to the approval mechanism of the current Codex session.
- If a dedicated Codex usage guide is compiled later, it should be placed in a separate Codex tool documentation file, not mixed with `mpos-dev` development rules.

## 10. Future To-Dos

- Add manually maintained `description`, `notes`, `examples` for commonly used APIs, and mark `description_source: "manual_override"`.
- If adding manual examples for `webcam`, continue processing according to the actual MPY interface structure: module-level functions plus `Webcam` type; examples should avoid writing non-existent methods.
- When re-synchronizing external docs, prioritize updating `docs-site-index.md`, then update the corresponding topic-specific reference.
- If creating `mpos-package-app`, `mpos-deploy-app`, `mpos-publish-app`, they should read from `mpos-dev/reference/` as needed, not copy and paste the entire set of materials.
