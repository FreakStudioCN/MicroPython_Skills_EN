# MicroPythonOS Conversational Skill Decomposition Analysis

Date: 2026-07-14
Last Updated: 2026-07-17

This document is for analysis and design recommendations only; it does not create or modify skills. The analysis is based on the current contents of local directories
`/home/leeqingshui/MicroPythonOS`, `/home/leeqingshui/MicroPython_Skills`,
`/home/leeqingshui/lvgl_micropython`, and the reading of the main content, public APIs, installation manifests, and docs `search_index.json` from
`https://upystore.io/`, `https://install.micropythonos.com/`,
`https://docs.micropythonos.com/`, `https://web.micropythonos.com/` on 2026-07-14.

## Conclusion

The conversational development of MicroPythonOS should not be implemented as a single, monolithic skill. The correct form is:

1. One user-entry orchestration skill, responsible for conversation, phase switching, and state handover.
2. Multiple phase-specific skills, responsible for requirements analysis, driver/dependency preparation, API reference refresh, App generation, testing/simulation, MPK packaging, device installation/flashing, and publishing guidance.
3. One shared base skill/reference layer, consolidating project facts about MicroPythonOS, LVGL, MPK, mpremote, simulators, etc.
4. Actions with high determinism, high error-proneness, or repetitive execution should be implemented as scripts, e.g., API extraction, manifest validation, single-app MPK packaging, MPK structure validation, device-side installation checks.

The proposed decomposition has been implemented as 8 user-visible phase skills + 1 shared base skill, with `mpos-debug-app` retained as a runtime troubleshooting aid:

| Proposed Skill | Type | Primary Responsibility | Status |
|---|---|---|---|
| `mpos-dev` | Shared Base | MicroPythonOS/LVGL/API constraints, API extraction script entry point | Existing, reference routing, LVGL independent repo path, API MD/JSON dual format completed |
| `mpos-plan-app` | User Entry/Orchestration | Conversational requirement clarification, phase state, invalidation confirmation, recovery, calling downstream skills | Existing, maintains `tmp/mpos-plan-app/<fullname>/plan_state.json` and `activity_log.jsonl` |
| `mpos-analyze-app` | Phase Skill | Requirements analysis, App type, functional boundaries, manifest draft, hardware/network/storage risks | Existing, produces `analysis_result.json` |
| `mpos-prepare-deps` | Phase Skill | Driver/dependency download, data retrieval, async/aio/uasyncio search, runtime file staging | Existing, produces `dependency_handoff.json` |
| `mpos-gen-app` | Phase Skill | Two-phase generation/modification/repair of MPOS App code, manifest, resources, static gating | Existing, default flat layout, produces `generation_result.json` |
| `mpos-test-app` | Phase Skill | Target App runtime smoke using MicroPythonOS built-in tools, optional Web Port check | Existing, does not own lint/manifest/flake8/pylint static gating |
| `mpos-package-app` | Phase Skill | Single App MPK packaging, `app_index_entry` fragment, MPK format validation, optional temporary install verification | Existing, produces `package_result.json` |
| `mpos-deploy-app` | Phase Skill | Desktop/web preview, device copy, MPK real-device install verification, installer/flash guidance | Existing, produces `deploy_result.json` |
| `mpos-publish-app` | Phase Skill | upystore publishing guidance and validation, version comparison, store metadata handover | Existing, produces `publish_result.json`, does not log in or upload |

After this round of re-reading the local repositories, four API references, `upystore.io`, `install.micropythonos.com`, and the entire docs site, this decomposition does not need to change; the focus of subsequent maintenance should be on maintaining responsibility boundaries, handover JSONs, and isolation from the main repository, rather than continuing to increase the number of phases. Downstream skills must still correctly use the API references, especially LVGL's `type_aliases[]` which can only be used to interpret signatures, not to generate code as runtime APIs.

Unified constraints updated on 2026-07-17:

- New Apps default to flat layout: `MANIFEST.JSON`, `icon_64x64.png`, `assets/*.py`; legacy `META-INF/` and `res/` are only read for compatibility with a warning.
- All phases share `<repo-root>/tmp/mpos-plan-app/<fullname>/plan_state.json` and `activity_log.jsonl`; phase artifacts are registered via `update_plan_state.py`.
- The main checkout at `/home/leeqingshui/MicroPythonOS` is not the default workspace for build/simulator/web integration testing. Testing and preview should use isolated clones/worktrees/temporary copies, and should not modify the OS source code except for adding new Apps or with explicit user permission.
- `mpos-gen-app` maintains a mandatory two-phase process; after each file write, it immediately runs and records manifest, syntax, MPY import, `make lint`, flake8, and pylint checks.
- `mpos-test-app` is positioned as runtime smoke; it does not reclaim the static gating from `mpos-gen-app`, nor does it fix external OS/tooling issues.
- `mpos-package-app` may still attempt packaging even if generation/testing is missing or failed, but must synchronize warnings to `package_result.json`.
- `mpos-deploy-app` no longer performs desktop smoke; desktop/manual launch and Web Port are optional preview paths. When no physical board is available, `desktop-preview` or `web-preview` results in `deploy_result.json` can satisfy the publish prerequisite.
- `mpos-publish-app` must read `package_result.json`, `app_test_result.json`, and `deploy_result.json` simultaneously; it is only for upystore publishing guidance, version comparison, and store metadata handover; it does not log in or upload.

This decomposition is more suitable than directly reusing the old `upy-*` pipeline. The `upy-*` pipeline is primarily for "generating general MicroPython hardware projects from natural language," whereas MicroPythonOS's core objects are Apps, Activities, MANIFEST, MPK, AppStore, desktop simulation, and system images, with different lifecycles and deliverables.

## Key Facts Observed

### Local MicroPythonOS Facts

- `MicroPythonOS/AGENTS.md` describes the project as a MicroPythonOS with AppStore, OTA, and built-in Apps. The main code is in `internal_filesystem/`, the build is based on `lvgl_micropython/`, and native MicroPython module implementation source code is in `c_mpos/`.
- Recommended command entry points include `make build-mpos-unix`, `make syntax-tests`, `make unittest-tests`, `make tests`, `make lint`, `make lint-fix`.
- The desktop simulation entry point is `scripts/run_desktop.sh`. AGENTS requires using `timeout -s 9 30 ./scripts/run_desktop.sh` for debugging runtime.
- The firmware build entry point is `scripts/build_mpos.sh <target>`, supporting `unix`, `macOS`, `esp32`, `esp32-small`, `esp32s3`, `unphone`, `lilygo_t4`, and other targets.
- The entry point for installing an App on a device is `scripts/install.sh com.micropythonos.<appname>`, which uses `lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py` under the hood.
- The firmware USB flashing entry point is `scripts/flash_over_usb.sh`, which writes to `lvgl_micropython/build/lvgl_micropy_ESP32_GENERIC_S3-SPIRAM_OCT-16.bin` by default.
- The App packaging entry point is `scripts/bundle_apps.sh`, which generates `../apps/app_index.json`, MPKs, and icon URLs from `internal_filesystem/apps`.
- As of 2026-07-17, the skill's default new App directory structure is the flat layout:

```text
internal_filesystem/apps/com.example.app/
  MANIFEST.JSON
  icon_64x64.png
  assets/main.py
```

The legacy layout is still read for compatibility but must issue a warning:

```text
internal_filesystem/apps/com.example.app/
  META-INF/MANIFEST.JSON
  assets/main.py
  res/mipmap-mdpi/icon_64x64.png
```

Historical analysis noted that local legacy Apps commonly used `META-INF/` and `res/`; the current skill strategy has changed to flat default, legacy compatibility. The generation, packaging, and publishing pipeline should prioritize root directory `MANIFEST.JSON` and `icon_64x64.png`, and only read legacy paths when processing existing legacy Apps.

### App and MPK Constraints

- `tests/test_apps_manifest.py` validates:
  - The App directory name must equal the `fullname` in `MANIFEST.JSON`.
  - `name` and `version` are required.
  - `version` must be in canonical integer dot notation, e.g., `0.1.6`.
  - The `entrypoint` for each activity/service must end with `.py` and the file must exist.
  - The `classname` must be found in the entrypoint source code.
- `internal_filesystem/lib/mpos/content/streaming_unzip.py` is very strict about MPKs:
  - The first ZIP local header must be the `{fullname}/` directory.
  - There must be only one top-level directory, and all files must be under it.
  - Supports `ZIP_STORED` and `ZIP_DEFLATED`.
  - Data descriptor flag is not allowed.
  - Non-compliance results in a direct `RuntimeError`.
- `tests/test_streaming_unzip.py` covers correct MPKs, missing top-level directory, wrong top-level directory, mixed top-level directories, and insufficient space.

Therefore, `mpos-package-app` cannot simply "zip the directory." It must reliably produce an MPK with `{fullname}/` as the first directory entry and exclude irrelevant files like `__MACOSX/`, `._*`, `.git/`, etc.

### Current Status of API Extraction Scripts

`mpos-dev` already has two key scripts:

- `mpos-dev/scripts/extract_mpos_api.py`
  - Scans native MicroPython modules and outputs the callable interface form for MPY users.
  - Scans Python framework classes, methods, functions, and docstrings in `internal_filesystem/lib/mpos/`.
  - Outputs `reference/mpos-api-reference.md`.
- `mpos-dev/scripts/extract_lvgl_api.py`
  - Reads `lvgl_micropython/lvgl.pyi` by default, only regenerating the stub when necessary.
  - Parses the LVGL Python API and outputs a structured summary.

This indicates that the requirement for "code generation needs scripts to extract MicroPythonOS and lvgl_micropython APIs" already has a foundation and should not be re-implemented within `mpos-gen-app`. It is recommended to make API refreshing a shared script capability of `mpos-dev`, called by `mpos-plan-app` or `mpos-gen-app` when needed.

Review result on 2026-07-14: The default candidates for `extract_lvgl_api.py` already include `~/lvgl_micropython` and `/home/leeqingshui/lvgl_micropython`, and the `source.path` in `reference/lvgl_api_summary.json` already points to `/home/leeqingshui/lvgl_micropython/lvgl.pyi`. This gap has been filled and should no longer be listed as a to-do.

Subsequent update on 2026-07-14: `extract_mpos_api.py` has output `reference/mpos-api-reference.md` and `reference/mpos_api_summary.json`, covering native MicroPython modules, `mpos.__all__` root exports, and a full public API index of all source code in `internal_filesystem/lib/mpos/**/*.py`; native modules only show the MPY import/call form in JSON/MD, without exposing implementation source file paths or underlying signatures. `extract_lvgl_api.py` has output `reference/lvgl-api-reference.md` and `reference/lvgl_api_summary.json`. Non-widget LVGL objects are uniformly marked as `data_classes` / `data_class`, real enum classes/members go into `symbols[]`, and `*_t = int` stub type aliases are placed separately in `type_aliases[]` with mapping to runtime enum classes where possible. Both JSONs contain `generated_at`, `generator`, `counts`, and `symbols[]`, allowing downstream skills to determine if the reference is outdated.

API reference review update on 2026-07-14: `mpos_api_summary.json` is currently generated by `extract_mpos_api.py v3`, with statistics showing 5 native MicroPython modules, 1206 symbols, and `root_export_missing=0`; `lvgl_api_summary.json` is currently generated by `extract_lvgl_api.py v5`, with source pointing to `/home/leeqingshui/lvgl_micropython/lvgl.pyi`, and statistics showing 60 `type_aliases`, 90 enum classes, 873 enum members, 79 data classes, 41 widget classes, 247 standalone functions, and 3715 symbols. It has been verified that neither reference/JSON contains leaks of C/C++ implementation details like `mp_obj_t`, `mp_map_t`, `c_binding`, `C type`, `c_mpos`, etc.; nor misleading runtime API symbols like `lv.display_render_mode_t`, `lv.grad_dir_t`, `lv.event_code_t`, `lv.fs_whence_t`.

### Review of Four API Summary Documents

These four files are the core factual sources for subsequent natural language generation of MicroPythonOS Apps:

- `reference/mpos-api-reference.md`
- `reference/mpos_api_summary.json`
- `reference/lvgl-api-reference.md`
- `reference/lvgl_api_summary.json`

Current assessment: These four files can continue to serve as the API basis for `mpos-gen-app` / `mpos-analyze-app` / `mpos-test-app`, but downstream skills must read them according to the JSON schema, not just perform string searches.

Usage constraints:

- The MPOS side should only be treated as an index of interfaces importable/callable by MicroPython users. Native modules should only use the MPY call forms of `adc_mic`, `pdm_mic`, `qrdecode`, `rvswd`, `webcam`, and should not infer C functions from `c_mpos`.
- The LVGL side should use `symbols[]` as the primary index for generatable APIs, especially `kind == "enum"` / `kind == "enum_member"` / `kind == "widget"` / `kind == "function"`.
- LVGL's `type_aliases[]` should only be used to interpret signature types. `runtime_api: false` means `lv.<alias>` cannot be generated; if `runtime_enum` exists, the corresponding enum class member should be generated, e.g., `event_code_t -> lv.EVENT.CLICKED`, `display_render_mode_t -> lv.DISPLAY_RENDER_MODE.PARTIAL`, `grad_dir_t -> lv.GRAD_DIR.VER`, `fs_whence_t -> lv.FS_SEEK.SET`.
- The continued appearance of `"display_render_mode_t"`, `"event_code_t"`, `"grad_dir_t"` in method signatures is normal type annotation and does not mean these names are runtime APIs. Conversely, `lv.area_t()`, `lv.style_t()`, `lv.anim_t()` and similar `*_t` data classes are real MPY classes/constructors and should not be blanket removed because of the `_t` suffix.
- When `description` is empty, downstream skills should not fabricate semantics; if interpretation is needed, first consult the docs reference, current repository code, or specific source code context.
- Before generating code, prioritize running or checking `generated_at`/`generator`; if the API reference is found to be outdated, call `mpos-dev/scripts/extract_mpos_api.py` and `mpos-dev/scripts/extract_lvgl_api.py --lvgl-micropython-dir /home/leeqingshui/lvgl_micropython`.

Impact on skill decomposition:

- `mpos-dev` retains the API extraction scripts and four references, and no longer stuffs large API tables into `SKILL.md`.
- `mpos-analyze-app` uses the API JSON to determine if existing managers/frameworks can meet requirements, avoiding premature driver downloads.
- `mpos-gen-app` must read or query the four API references before generation, especially LVGL enums/members, to avoid generating erroneous code like `lv.OBJ_FLAG.*`, `lv.EVENT_VALUE_CHANGED`, `lv.scr_act()`, `lv.display_render_mode_t`.
- `mpos-test-app` and `mpos-debug-app` can use the API JSON for candidate API retrieval during error repair, but must not treat type annotations in the JSON as runtime properties.

### lvgl_micropython Facts

`/home/leeqingshui/lvgl_micropython/README.md` states:

- This is an LVGL binding independent of the old `lv_micropython`, built via `python3 make.py <target> ...`.
- Information from the official binding should not be applied to this binding.
- Manual submodule init commands should not be added.
- Supports unix/macOS SDL2 special drivers and multiple types of display/touch ICs.

MicroPythonOS skills must prioritize using the `lvgl_api_summary.json` generated from this repository, the independent `/home/leeqingshui/lvgl_micropython/lvgl.pyi`, and the AGENTS rules within the project, rather than generalized LVGL/MicroPython memory.

It is not recommended to make `/home/leeqingshui/lvgl_micropython` a separate user-entry skill; it is better suited as an API/reference source for `mpos-dev`, and to be read specifically by `mpos-prepare-deps` or `mpos-deploy-app` when dealing with display/touch drivers, board ports, or binding builds.

### Public Site Access Status

Results from re-reading on 2026-07-14:

- `https://upystore.io/` main page returned `200`; `https://upystore.io/apps` returned `200`, fetched 30373 bytes; `https://upystore.io/developer` showed the Developer Console login/publish page when not logged in, returned `200`, fetched 8267 bytes, and included `submit MPK` text; `https://upystore.io/app_index.json` returned `200`, fetched 5946 bytes; `https://upystore.io/api/v1/apps` returned `200`, fetched 10843 bytes.
- `upystore.io/app_index.json` is currently a list of 10 apps, with fields including `activities`, `category`, `download_url`, `fullname`, `icon_url`, `long_description`, `name`, `publisher`, `short_description`, `version`. `api/v1/apps` currently returns `apps`, `filters`, `pagination`, with pagination showing `total=10`, `total_pages=1`, and additionally includes storefront fields like `slug`, `revision`, `tags`, `hardware_tags`, `min_os_version`, `min_api_level`, `screenshots`, `installs_count`, `downloads_count`, `stars_count`, `released_at`.
- This round also read the public app detail pages for each of the 10 `slug`s from `api/v1/apps`, with the result `UPYSTORE_DETAIL_OK=10/10`; the detail pages contain the corresponding `fullname`, `name`, `version`, which can be used for manual verification after publishing, but should not replace local MPK/manifest validation.
- `https://install.micropythonos.com/` currently returns `200` directly via HTTPS, fetched 17558 bytes. The page uses `esp-web-install-button`, containing information about `WebSerial`, `USB`, `ESP32`, `ESP32-S3`, `0.15.x`, etc.; the previously recorded `SSL_ERROR_SYSCALL` state on this VM is now outdated.
- The Installer currently lists 12 manifests for ESP32-S3 and ESP32 across versions `0.10.x`, `0.11.x`, `0.12.x`, `0.13.x`, `0.14.x`, `0.15.x`. All manifests were read individually, all returned `200` with `new_install_prompt_erase=true`; the latest `0.15.x` corresponds to version `0.15.1`, with firmware paths pointing to `/firmware_images/esp32s3/MicroPythonOS_esp32s3_0.15.1.bin` and `/firmware_images/esp32/MicroPythonOS_esp32_0.15.1.bin`.
- `https://docs.micropythonos.com/` currently returns `200` directly via HTTPS, fetched 39201 bytes; `https://docs.micropythonos.com/sitemap.xml` returned `200`, fetched 10794 bytes; `https://docs.micropythonos.com/search/search_index.json` returned `200`, fetched 775522 bytes. The previously recorded `SSL_ERROR_SYSCALL` state on this VM is now outdated.
- The `lastmod` of pages in the docs sitemap is `2026-07-13`. This round read all 61 pages from the sitemap, with the result `DOCS_FETCH_OK=61/61 failed=0`; the re-fetched `search_index.json` parsed 977 document/section entries, covering `Apps`, `App Lifecycle`, `Bundling Apps`, `AppStore`, `Frameworks`, `Running`, `Supported Hardware`, `DownloadManager`, `TaskManager`, `SharedPreferences`, `AppManager`, `Service`, and other content directly relevant to skill decomposition.
- `https://web.micropythonos.com/` returned `200`, fetched 18555 bytes, with the title `MicroPythonOS Web`; the page loads `micropython.js`, runs MicroPythonOS in the browser, defaults to a `320x240` LVGL canvas, and provides simulated badge peripherals: Log, Reset storage, joystick, MENU/START, X/Y/A/B, NeoPixel simulation, etc.

### docs.micropythonos.com Facts

After reading the docs homepage, sitemap, and `search_index.json` on 2026-07-14, the facts directly relevant to skill decomposition are:

- The docs homepage defines MicroPythonOS as a lightweight operating system built entirely with MicroPython, targeting ESP32, desktop, and browser, featuring an Android-inspired UI, App ecosystem, App Store, and OTA updates.
- `Getting Started / Running` explains that pre-built firmware installation goes through the WebSerial installer at `install.micropythonos.com`; the desktop side can run pre-built binaries, use source checkout for app development, or build from source; the Web side can open `web.micropythonos.com` or the main branch WebAssembly build.
- `Supported Hardware` covers ESP32, ESP32-S3, WebAssembly, Linux/macOS/Windows WSL2/Raspberry Pi, etc., supporting placing "Linux simulation" and "firmware flashing" under the same `mpos-deploy-app` but as different paths.
- `Apps / Creating Apps` states that apps are installed in `/apps/`, manifests declare `activities` and `services`, and services can use `boot_completed` to run automatically after startup; however, the flat directory example on this page is inconsistent with the current local repository, so skills must prioritize local testing.
- `Apps / App Lifecycle` clearly defines the Activity lifecycle including `onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy`, and that an Activity is a UI screen; a Service is a background component without UI. This supports identifying Activity/Service separately during requirements analysis, code generation, and testing.
- `Apps / Bundling Apps` clearly states that `.mpk` is a ZIP archive, the first entry must be the app fullname top-level directory, and there can only be one top-level directory; this is consistent with the local `StreamingUnzip` and `tests/test_streaming_unzip.py`.
- `Apps / AppStore` explains that the AppStore can pull apps from multiple backends and installs apps as `.mpk` files into `/apps/`. This supports `mpos-publish-app` focusing only on preparing and validating packages, without mixing upystore uploads with device-side installation logic.
- `Frameworks` documentation recommends that applications import frameworks directly from the main `mpos` module, e.g., `from mpos import AppManager, DisplayMetrics`, avoiding imports from submodules like `mpos.ui`, `mpos.content`; pages like `DownloadManager`, `TaskManager`, `SharedPreferences`, `AppManager`, `Service` provide API types that need to be consulted during the code generation phase.

Note: This round read all 61 pages of the docs according to the sitemap and used the 977 page/section entries from `search_index.json` for coverage verification; however, the full text of the 61 pages was not transcribed verbatim into offline reference files. For skills, the approach of "stuffing the entire site text directly into `SKILL.md`" is also inappropriate.

The current approach is to split the docs content into topic-specific reference files under `mpos-dev/reference/`, rather than creating a single massive reference:

```text
mpos-dev/reference/
  docs-app-model.md          # apps/creating-apps, app-lifecycle, appstore
  docs-packaging.md          # bundling-apps, MPK, AppStore packaging
  docs-frameworks.md         # framework overview + manager index
  docs-deploy-targets.md     # running, supported-hardware, desktop, ESP32, web port
  docs-os-development.md     # compiling, testing, porting, release checklist
  docs-web-port.md           # web-port/using, web-port/developer, web.micropythonos.com
  docs-site-index.md         # sitemap coverage and reference routing audit
```

`SKILL.md` only retains routing rules for "which reference to read when." This aligns with progressive disclosure: when generating an App, only read the app model and framework summary; when packaging, only read packaging; when deploying/simulating, only read deploy targets and web port.

### web.micropythonos.com Facts

`https://web.micropythonos.com/` is the browser/WebAssembly runtime entry point for MicroPythonOS, not an installation site or app publishing site. Page facts read on 2026-07-14 are:

- The page title is `MicroPythonOS Web`, loads `micropython.js`, and runs MicroPythonOS in the browser via Emscripten.
- The page defaults to a `320x240` LVGL canvas, corresponding to the MicroPythonOS default display profile.
- The page provides simulated badge peripherals: NeoPixel indicator light, joystick, MENU, START, X/Y/A/B buttons; these interact with the Python-side `_webio`/fake WebExpander via `Module.__webio`.
- The page mounts `/data` and `/apps` to IndexedDB/IDBFS, allowing app preferences and user-installed apps to persist across refreshes.
- The page provides a `Reset storage` button that deletes persistent `/data` and `/apps` and reloads the page.
- The runtime parameters are `["-X", "heapsize=16M", "-m", "main"]`, i.e., starting the frozen `main` module with a 16MB heap.
- This entry point is suitable for inclusion as reference material in `mpos-deploy-app`/`mpos-test-app`: for browser-side smoke testing, user quick preview, and Web Port limitation explanations; it should not replace Linux SDL desktop simulation or real device verification.

### upystore.io Facts

The main site, Apps page, Developer login page, `app_index.json`, and `api/v1/apps` read on 2026-07-14 show:

- upystore is an app store for MicroPython/uPyOS hardware. The main site navigation includes Home, Apps, Developer Console, Log in, Sign up, and English/Chinese language switching.
- The homepage emphasizes the three steps of Browse, Install, and Publish, targeting D-Shell/uPyOS devices, supporting app categorization, version history, hardware profiling/matching, install statistics, publisher information, icons, screenshots, release notes, and hardware requirements.
- The Apps page currently displays 10 apps, with categories including IoT, Motor Control, AI, Communication, Games, Media, Education, Development Tools, Utilities, and provides search, category filtering, download count, install count, stars, version number, and other fields.
- The Developer Console, when not logged in, redirects to a login page. The page text clearly states that a developer account is required to submit MPK packages, manage apps, and review app statistics. Therefore, `mpos-publish-app` should not automatically log in or request account credentials; it should only provide upload links and pre/post-publishing verification checklists.
- The `app_index.json` currently includes fields like `name`, `publisher`, `short_description`, `long_description`, `icon_url`, `download_url`, `fullname`, `version`, `category`, `activities`, etc., with a format generally close to what the MicroPythonOS AppStore requires.
- Current upystore data has two forms of `activities`: `Show Battery` and `Danke` use a full object array (`classname`, `entrypoint`, `intent_filters`), while some seed/sample apps use a string array (e.g., `clock`, `timer`). The publishing skill must output the full manifest form when generating metadata and should not copy the string activity form from seed data.
- `api/v1/apps` has more storefront fields than `app_index.json`, including `slug`, `revision`, `tags`, `hardware_tags`, `min_os_version`, `min_api_level`, `screenshots`, `installs_count`, `downloads_count`, `stars_count`, `released_at`, `pagination`; these are suitable for publishing summaries and post-upload verification, but should not replace the local `MANIFEST.JSON`.
- upystore uploads themselves should remain a user action: the skill prepares the MPK, icon, metadata summary, and validation results, then recommends the user visit `https://upystore.io/` or the Developer Console to upload. After upload, if the user provides a download URL or app_index/API return value, the skill can then perform MPK top-level directory and device-side installation verification.
- The old conclusion from the local `MicroPythonOS/docs/upystore-integration-analysis_CN.md` remains critical: the device side should not relax `StreamingUnzip` validation; the packaging/upload side must ensure the first ZIP entry of the MPK is `{fullname}/` and exclude junk files like `__MACOSX/`, `._*`.

## Basic Writing Style for Conversational Skills

Conversational does not mean writing very long explanations; it means each phase has clear inputs, outputs, and next steps. It is recommended that each phase uses a lightweight state object, for example:

```json
{
  "app_fullname": "com.example.weather",
  "app_name": "Weather",
  "phase": "generate",
  "target": "desktop-first",
  "requirements": [],
  "dependencies": [],
  "artifacts": {
    "app_dir": "internal_filesystem/apps/com.example.weather",
    "manifest": "internal_filesystem/apps/com.example.weather/MANIFEST.JSON",
    "mpk": null
  },
  "open_questions": [],
  "verification": []
}
```

Principles:

- The orchestration skill only asks necessary blocking questions, not a long questionnaire all at once.
- Each phase skill receives upstream state and produces handoverable artifacts, not just natural language conclusions.
- When the user explicitly wants "from one sentence to completion," the entry skill calls phase skills sequentially; when the user explicitly wants only testing/packaging/publishing, trigger the corresponding skill directly.
- High-risk actions must be separated: code generation, firmware flashing, file system erasure, and upload/publishing should not be mixed in one skill and executed by default.
- `SKILL.md` should be kept short; detailed rules go into `references/`; deterministic actions go into `scripts/`.

## Proposed Skill Design

### 1. `mpos-dev`: Shared Base Layer

Positioning: The knowledge base for all MicroPythonOS App skills, not a primary user entry point.

Content to retain:

- MicroPythonOS directory structure, Activity/App/Service lifecycle.
- LVGL 9.x/micropython binding constraints.
- Native MicroPython module API quick reference.
- Global constraints: ruff double quotes, temporary files in project `tmp/`, desktop run with timeout, debug process kill with `killall`.
- `extract_mpos_api.py` and `extract_lvgl_api.py`.

Completed or should be maintained:

- The App structure illustration should continue to use the current skill's default new layout: `MANIFEST.JSON`, `icon_64x64.png`, `assets/*.py`.
- The flat default and legacy compatibility strategy is already documented in `reference/docs-app-model.md` / `reference/docs-packaging.md`; generation and packaging should follow the current skill and local tests, with docs serving only as background reference.
- Clarify that `from mpos import Activity` is the recommended unified import method from docs, and it can also be imported from `mpos.app.activity import Activity`; new skills should prioritize the main `mpos` re-export unless the current code has a local convention.
- The default path for `extract_lvgl_api.py` already covers `/home/leeqingshui/lvgl_micropython`; when other binding checkouts are involved, `--lvgl-micropython-dir` can still be passed explicitly.
- MicroPythonOS and LVGL API references now have both MD/JSON formats; JSON includes generation time, statistics, and symbol index, making it easy to determine if outdated.

Current/Proposed resources:

```text
mpos-dev/
  SKILL.md
  reference/docs-app-model.md
  reference/docs-packaging.md
  reference/docs-frameworks.md
  reference/docs-deploy-targets.md
  reference/docs-os-development.md
  reference/docs-web-port.md
  reference/docs-site-index.md
  reference/mpos-api-reference.md
  reference/mpos_api_summary.json
  reference/lvgl-api-reference.md
  reference/lvgl_api_summary.json
  reference/lvgl-rules.md      # Optional: extract if LVGL rules in SKILL.md continue to grow
  scripts/extract_mpos_api.py
  scripts/extract_lvgl_api.py
```

### 2. `mpos-plan-app`: Conversational Entry/Orchestration

Triggered when:

- The user says "help me make a MicroPythonOS App."
- The user provides only a natural language functional description and wants to go from requirements to generation, testing, packaging, and deployment.
- The user asks "what's next," "continue this app," or "resume the last task."
- The user modifies requirements, requiring a determination of which phase artifacts are invalidated.

Responsibilities:

- Acts as the state machine and orchestration entry point, defaulting to running through to the publishing handover of `mpos-publish-app`.
- Maintains `<repo-root>/tmp/mpos-plan-app/<fullname>/plan_state.json` and `activity_log.jsonl`.
- When `fullname` is not provided, can automatically select based on the most recently modified App or recent artifact, and clearly informs the user of the selection basis.
- When the user changes requirements, first lists the invalidation list and waits for confirmation; does not delete, overwrite, or re-run downstream before confirmation.
- Calls downstream skills, but does not write code, download dependencies, test, package, deploy, or publish itself.
- Must not bypass the mandatory two-phase confirmation of `mpos-gen-app`.

Does NOT do:

- Does not directly download unknown drivers.
- Does not directly generate or modify App files.
- Does not directly run tests, packaging, deployment, flashing, or uploading.
- Does not directly flash.
- Does not automatically upload to upystore.

Outputs:

- Current phase.
- Discovered artifact paths.
- List of invalidated artifacts.
- Next skill and reason.
- Project log path.

### 3. `mpos-analyze-app`: Requirements Analysis

Triggered when:

- The user provides an App idea but without clear functional boundaries.
- Requirements need to be translated into manifest, Activity, Service, data persistence, hardware permissions/dependency plans.

Responsibilities:

- Identifies App type: pure UI, tool, game, network, audio, camera, sensor, background service.
- Produces minimum viable features, non-goals, and risk points.
- Suggests `fullname`, `name`, `category`, `activities`, `services`.
- Determines if the following are needed:
  - `SharedPreferences`
  - `TaskManager`
  - `DownloadManager`
  - `CameraManager`
  - `AudioManager`
  - `SensorManager`
  - External MicroPython drivers
  - C modules or firmware recompilation
- Defines testing strategy: regular unittest, GraphicalTestCase, manual hardware testing, device-side verification.
- Produces `analysis_result.json` and registers it in the project state.

Suggested output:

```json
{
  "schema_version": "mpos-analyze-v1",
  "phase": "analyze",
  "manifest_draft": {},
  "feature_slices": [],
  "dependency_plan": [],
  "test_plan": [],
  "blocking_questions": []
}
```

### 4. `mpos-prepare-deps`: Driver Download and Dependency Preparation

Triggered when:

- The App requires external sensor/display/network/service SDKs.
- The user says "download driver," "find library," "this hardware has no driver."
- `mpos-analyze-app` marks a dependency gap.

Responsibilities:

- Prioritizes checking MicroPythonOS built-in capabilities and managers, avoiding redundant introduction of external drivers.
- When a MicroPython driver is needed, reuses the existing data retrieval capabilities of `fetch-doc`, `upy-pkg-guide`, `upy-gen-driver`, but must append async/aio/uasyncio search strategies.
- Can actually download runtime-required pure Python/MPY files to the App's `assets/`; if the App has not been created yet, stages them to `tmp/mpos-deps-cache/<fullname>/staged/assets/`.
- Search results, READMEs, examples, metadata, and candidate evidence are written to `tmp/mpos-deps-cache/<fullname>/`, not to the skill directory.
- Synchronous libraries are allowed to be retained, but must be marked with `sync_needs_adapter=true` in `dependency_handoff.json`, delegating to `mpos-gen-app` to generate a non-blocking wrapper.
- Produces `dependency_handoff.json` and registers it in the project state.

Boundaries:

- This is not `upy-select-hw`. MicroPythonOS Apps typically run on existing devices/system images and should not default to redoing MCU/pin selection.
- Does not modify `lvgl_micropython`, board ports, CMake, native bindings, or firmware configuration.
- Does not accept C extensions, frozen modules, native modules, private binary blobs, or dependencies requiring firmware recompilation; such dependencies go into rejected/warning.
- When peripheral pins and target boards are involved, should ask the user to provide the device/board, or read `internal_filesystem/lib/mpos/board/*.py`.

Suggested scripts:

- `scripts/build_search_plan.py`: Generates and caches base query plans plus async/aio/uasyncio query plans.
- `scripts/stage_runtime_file.py`: Writes runtime files to the real App or staged cache depending on whether the App exists.
- `scripts/validate_dependency_handoff.py`: Validates the handoff schema.

### 5. `mpos-gen-app`: Code Generation

Triggered when:

- The user wants to create/modify a MicroPythonOS App.
- Upstream requirements analysis and dependency plans are available.

Responsibilities:

- Creates or modifies `internal_filesystem/apps/<fullname>/`.
- Generates root directory `MANIFEST.JSON` by default.
- Generates Activity/Service code in `assets/*.py`.
- Creates or completes root directory `icon_64x64.png`, can use `scripts/generate_icon.py` to generate a 64x64 PNG based on the user's functional description.
- Uses `mpos-dev`'s API reference and LVGL rules.
- Strictly adheres to this repository's LVGL constraints for UI code.
- Supports repeated invocation: new creation, user functional modification, re-validation after manual file edits, test failure repair.
- Produces `generation_result.json` and registers it in the project state.

Current generation rules:

- Default is mandatory two-phase: first plan and ask for user confirmation, then create/update/repair only after confirmation.
- Default structure for new Apps:

```text
internal_filesystem/apps/<fullname>/
  MANIFEST.JSON
  icon_64x64.png
  assets/main.py
```

- Legacy `META-INF/MANIFEST.JSON` and `res/mipmap-mdpi/icon_64x64.png` are only read for compatibility when updating legacy Apps, with a warning.
- `entrypoint` should prioritize `assets/main.py` or `assets/<app>.py`, must have the `.py` suffix, and the file must exist.
- Activity class names must appear in the entrypoint code.
- If a custom `__init__` is defined, it must call `super().__init__()`.
- New labels must explicitly call `set_text("")` or set the target text.
- After `style_t()`, `init()` must be called.
- Do not hardcode screen resolution; prioritize `lv.pct(100)`, flex, align.
- Use `SharedPreferences(self.appFullName)` for persistence.
- Use `TaskManager` for background/asynchronous tasks; UI updates need to be on the foreground or main thread safe path.
- If consuming synchronous libraries from `dependency_handoff.json`, an adapter must be generated to avoid blocking directly in `async def` or LVGL event callbacks.
- Test failure repair allows unlimited automatic repairs of its own generated/modified App files; external OS/tooling issues must not cause it to modify OS source code.

Fixed gating after execution:

- `python -m unittest tests/test_apps_manifest.py`
- `scripts/check_app_syntax.py`
- `scripts/check_app_mpy_imports.py`
- `make lint`
- flake8, using `mpos-gen-app/templates/flake8-mpos-app.ini`
- pylint, using `mpos-gen-app/templates/pylintrc-mpos-app`
- Clean up `__pycache__/` and `.pyc`

### 6. `mpos-test-app`: Testing and Quality Gating

Current positioning is runtime testing, not static gating.

Triggered when:

- The user wants to verify the target App in the MPOS runtime.
- Static gating from `mpos-gen-app` passes, entering runtime smoke.
- The user requests Web Port verification or desktop controller automation.

Responsibilities:

- Reviews that the static gating from `generation_result.json` has been recorded.
- Defaults to first trying `<repo-root>/scripts/run_desktop.sh <fullname>` as a built-in desktop runner probe.
- Uses `scripts/mpos_controller.py` and `AppManager.start_app("<fullname>")` for structured smoke testing.
- Collects visible text, widget tree, traceback, and optional screenshot.
- Optionally runs target App-specific `GraphicalTestCase`/`KeyboardTestCase`, but does not run the full OS regression suite.
- Optionally performs Web Port check: if `web/` artifacts exist, serve + HTTP check; if Emscripten, Chrome, or web artifacts are missing, records skipped/warning.
- For screenshots, uses widget tree/visible text/pixel checks, not just visual description.
- Produces `app_test_result.json` and registers it in the project state.

Boundaries:

- Does not run `make lint`, flake8, pylint, or manifest validation; these belong to `mpos-gen-app`.
- Does not default to fixing OS/tooling issues like `_webrepl`, desktop binary, libffi/libv4l; only marks them as external/tooling blocked. When explicitly requested by the user, helper scripts can be used to prepare local desktop tooling, but OS source code is not edited.
- Does not reference `mpos-debug-app` as a default prerequisite.

### 7. `mpos-package-app`: App Packaging

Triggered when:

- The user says "package app," "generate MPK," "prepare for AppStore/upystore release."
- Code generation and testing are complete, entering release preparation.

Responsibilities:

- Reads `internal_filesystem/apps/<fullname>/MANIFEST.JSON`; legacy `META-INF/MANIFEST.JSON` is only read for compatibility with a warning.
- Validates manifest and file structure.
- Ensures icon exists, default root directory `icon_64x64.png`; legacy `res/mipmap-mdpi/icon_64x64.png` is only read for compatibility with a warning.
- Generates a single App MPK.
- Generates a single `app_index_entry.json` by default, does not merge the full `app_index.json`.
- Generates or outputs the app_index entry, including:
  - `icon_url`
  - `download_url`
  - `fullname`
  - `version`
  - `activities`
  - `services`
- Validates that the first ZIP entry of the MPK is the `{fullname}/` directory.
- Default compression method is `stored`.
- If `generation_result.json` or `app_test_result.json` is missing or failed, packaging is still allowed but must issue a warning and set the result to partial.
- Optional temporary install verification only extracts to `tmp/mpos-package-app/<fullname>/install-check/`, does not write to the real App directory.
- Produces `package_result.json` and registers it in the project state.

Why a separate skill is needed:

- `scripts/bundle_apps.sh` is a full packaging script with blacklist and app store batch output logic, unsuitable as the sole entry point for "packaging only the current App" in a user conversation.
- MPK specifications are strict; incorrect packages will fail during device-side download. This phase requires deterministic scripts.

Suggested scripts:

```text
mpos-package-app/
  scripts/package_mpos_app.py
  scripts/validate_mpk.py
  scripts/emit_app_index_entry.py
```

`package_mpos_app.py` must ensure:

- The first entry in the zip is `{fullname}/`.
- File order is stable.
- Modification times can be fixed for reproducible builds.
- Excludes `.git/`, `__pycache__/`, `*.pyc`, `__MACOSX/`, `._*`.
- Default is stored; deflated can also be used explicitly, but must be within the `StreamingUnzip` support range.

### 8. `mpos-deploy-app`: Deployment, Preview, and Installation Paths

Triggered when:

- The user says "preview," "run desktop simulator," "web preview," "install to device," "MPK real-device verification," "flash/burn firmware."
- After packaging, deployment/preview records are needed before publishing.

Responsibility breakdown:

- `desktop-preview`: Manual/human preview path, does not perform smoke assertions, does not replace `mpos-test-app`.
- `web-preview`: Defaults to only serving existing `web/` artifacts; if artifacts are missing, prompts the user to confirm a build, does not auto-build.
- `device-copy`: After explicit confirmation of target board type and serial port, uses the project's mpremote or `mpos_controller.py installapp` for controlled copying.
- `mpk-install`: Recommended real-device release verification path; uploads the MPK and then calls the device-side AppManager for installation.
- `install-site`: Firmware installation, erasure, and board identification default to guiding the user to `https://install.micropythonos.com/`.
- `local-flash`: Disabled by default; only allowed to call `scripts/flash_over_usb.sh` after the user explicitly says "allow local flash execution."
- Each time, writes `deploy_result.json`, recording the device, port, installation mode, command, result, warnings, and next steps.

Boundaries:

- "Installing an App" is not "flashing firmware." The vast majority of Python App iterations only require copying the app directory or installing an MPK.
- "Linux simulation" is not the runtime smoke of `mpos-test-app`; in deploy, it is only a preview/pre-publishing handover record.
- Does not run static lint, does not package, does not log in or upload.
- Does not modify MicroPythonOS OS/build source code.

### 9. `mpos-publish-app`: upystore Publishing Guidance

Triggered when:

- The user says "upload to upystore," "publish App," "prepare for AppStore listing," "generate publishing handover."

Responsibilities:

- Must read `package_result.json`, `app_test_result.json`, and `deploy_result.json` simultaneously.
- Checks if the upstream result schema, phase, and `fullname` match; `failed` is a blocker, `partial` can proceed with a warning.
- Reads the upystore public list and compares the published version of the same `fullname`.
- Maintains upystore store metadata handover: `short_description`, `long_description`, `hardware_tags`, `release_notes`, `screenshots`, and optionally `tags`, `category`, `min_os_version`, `min_api_level`.
- Outputs `publish_result.json`, recording version status, release readiness, warnings, and local artifact paths required for upload.
- Clearly states that the upystore Developer Console requires the user to log in with a developer account; the skill does not request or save account credentials.
- Guides the user to open `https://upystore.io/developer` to upload.
- Informs the user of recommended post-upload verification:
  - Whether the fields in app_index are complete.
  - Whether the downloaded MPK still retains the `{fullname}/` top-level directory.
  - Whether the device-side AppStore can install it.

Does NOT do:

- Does not automatically upload on behalf of the user.
- Does not save or request upystore account credentials.
- Does not generate MPKs, run tests, deploy to devices, or fix code.
- Does not treat upystore as a firmware publishing site. Firmware publishing and `install.micropythonos.com` belong to a different process.

## Recommendations for Reusing and Adjusting Existing Skills

### Existing `mpos-*`

The current `MicroPython_Skills` already has a complete main chain:

- `mpos-dev`
- `mpos-plan-app`
- `mpos-analyze-app`
- `mpos-prepare-deps`
- `mpos-gen-app`
- `mpos-test-app`
- `mpos-package-app`
- `mpos-deploy-app`
- `mpos-publish-app`
- `mpos-debug-app`

Maintenance priorities:

- All phases must register in a unified project log; state must not be scattered across individual skills.
- The main repository `/home/leeqingshui/MicroPythonOS` is not the default workspace for build/simulator/web integration testing; testing and preview use isolated clones/worktrees/temporary copies.
- Flat layout is the default for new Apps; legacy layout is only compatible with a warning.
- The two-phase confirmation of `mpos-gen-app` must not be bypassed.
- `mpos-test-app` does not reclaim static gating, `mpos-deploy-app` does not reclaim runtime smoke, `mpos-publish-app` does not reclaim packaging/testing/deployment.
- No skill should modify MicroPythonOS OS/build source code to "fix the environment on the side"; when local patches are needed, use skill helper scripts or an isolated workspace.

### Role of Old `upy-*`

Can be reused but should not be directly used as the MicroPythonOS main chain:

- `fetch-doc`: Can be used for driver data, GitHub, URL content supplementation.
- `upy-pkg-guide`: Can be used for MicroPython driver package usage queries.
- `upy-gen-driver` / `upy-gen-driver-plugin`: Can be used for missing driver branches.
- `mpremote-*`: Can be used for device connection, file copying, long sessions.
- `upy-deploy`: The concept can be referenced, but MicroPythonOS App installation should prioritize the `scripts/install.sh`, MPK, and AppManager flow.

Parts not recommended for direct reuse as the MPOS main chain:

- `upy-select-hw`: It targets MCU/pin/firmware selection; MicroPythonOS Apps already have a target system by default.
- `upy-scaffold`/`upy-generate`: They generate general MicroPython firmware projects, not equivalent to MPOS App Activity/manifest/MPK.
- `upy-simulate`: It is a PC CLI/rich simulation, not equivalent to MicroPythonOS unix SDL desktop simulation.

## Recommended Maintenance Order

1. First check `mpos-plan-app` and `mpos-dev` to confirm that the global log, isolated workspace, flat/legacy layout, and API reference routing have not drifted.
2. Then check the modified phase skill to confirm its input artifacts, output JSON, helper scripts, and templates are consistent.
3. When code generation is involved, focus on verifying the two-phase confirmation of `mpos-gen-app`, the fixed flake8/pylint templates, `generation_result.json`, and repair boundaries.
4. When testing/deployment is involved, focus on verifying responsibility separation: runtime smoke belongs to `mpos-test-app`, preview/device installation belongs to `mpos-deploy-app`.
5. When publishing is involved, focus on verifying that the three handover results from `mpos-package-app`, `mpos-deploy-app`, and `mpos-publish-app` are read simultaneously.
6. After each modification, run the relevant skill's `quick_validate.py` or its respective validator; if there is no unified quick validation, at least run `py_compile` on the modified script and the template JSON validator.

## Template for Each SKILL.md

The `description` of each skill must include "what it does + when it is triggered," as this is the primary basis for Codex to trigger the skill. The body should only contain the necessary process and resource navigation.

Example:

```markdown
---
name: mpos-package-app
description: Package and validate MicroPythonOS Apps as MPK files for AppStore/upystore release. Use when Codex needs to create a .mpk, validate MANIFEST.JSON, emit app_index metadata, or prepare an MPOS App for publishing.
---

# MicroPythonOS App Packaging

## Workflow

1. Read `mpos-dev` for MPOS App and MPK constraints.
2. Locate `internal_filesystem/apps/<fullname>/MANIFEST.JSON`; treat legacy `META-INF/MANIFEST.JSON` as compatibility with warning.
3. Run `scripts/validate_mpos_app.py --repo <repo-root> --app-fullname <fullname>`.
4. Run `scripts/package_mpos_app.py --repo <repo-root> --app-fullname <fullname> --compression stored`.
5. Run `scripts/validate_package_result.py <package_result.json>`.
6. Report MPK path, `app_index_entry.json`, warnings, and next handoff.

## Constraints

- The first ZIP entry must be `<fullname>/`.
- Exclude `__MACOSX/`, `._*`, `.git/`, `__pycache__/`, `*.pyc`.
- Emit one `app_index_entry.json`; do not merge full `app_index.json`.
- Do not publish or upload automatically.
```

## Minimum Script Inventory

The current main chain already has these scripts. During subsequent maintenance, priority should be given to fixing/reusing them rather than rewriting them in conversation:

| Script | Belongs to Skill | Purpose |
|---|---|---|
| `scripts/update_plan_state.py` | `mpos-plan-app` | Unified registration, discovery, and invalidation marking of phase artifacts |
| `scripts/validate_plan_state.py` | `mpos-plan-app` | Validates `plan_state.json` |
| `scripts/check_app_syntax.py` | `mpos-gen-app` | CPython/MPY syntax risk validation |
| `scripts/check_app_mpy_imports.py` | `mpos-gen-app` | MicroPython import risk validation |
| `scripts/generate_icon.py` | `mpos-gen-app` | Generates `icon_64x64.png` using standard library |
| `scripts/build_search_plan.py` | `mpos-prepare-deps` | Dependency search plan, including async/aio/uasyncio |
| `scripts/stage_runtime_file.py` | `mpos-prepare-deps` | Downloads runtime files to App or staged cache |
| `scripts/package_mpos_app.py` | `mpos-package-app` | Generates a standard MPK for a single App |
| `scripts/validate_mpk.py` | `mpos-package-app` | Checks ZIP entry order, top-level directory, illegal files |
| `scripts/emit_app_index_entry.py` | `mpos-package-app` | Outputs app_index entry based on manifest and base URL |
| `scripts/run_app_smoke.py` | `mpos-test-app` | Runs runtime smoke using built-in desktop/controller tools |
| `scripts/launch_desktop_preview.py` | `mpos-deploy-app` | Desktop manual preview |
| `scripts/serve_web_preview.py` | `mpos-deploy-app` | Serves existing Web Port artifacts |
| `scripts/deploy_app_copy.py` | `mpos-deploy-app` | Device App file copy |
| `scripts/deploy_mpk_install.py` | `mpos-deploy-app` | Real-device MPK installation verification |
| `scripts/prepare_publish.py` | `mpos-publish-app` | Generates upystore publishing handover |

Existing scripts to keep:

- `mpos-dev/scripts/extract_mpos_api.py`
- `mpos-dev/scripts/extract_lvgl_api.py`

## Point-by-Point Response to the User's Original Ideas

> Requirements analysis

Already separated into `mpos-analyze-app`, called by `mpos-plan-app`. It outputs `analysis_result.json` and does not write code directly.

> Driver download

Already separated into `mpos-prepare-deps`. It first checks MicroPythonOS built-in managers and API references, and does not immediately download generic MicroPython drivers. When a driver is genuinely needed, it can actually download runtime files to the App or staged cache, and must cache search results and append async/aio/uasyncio strategies.

> Code generation, needs scripts to extract MicroPythonOS and lvgl_micropython APIs

Code generation is still done by `mpos-gen-app`. API extraction scripts should be placed in the shared `mpos-dev`; the existing `extract_mpos_api.py` and `extract_lvgl_api.py` are already on the right track. The code generation skill should first read/refresh the reference, then generate code.

> `/home/leeqingshui/lvgl_micropython`

It should serve as one of the factual sources for the LVGL binding, but official LVGL binding documentation should not be directly applied. Prioritize using the locally generated `lvgl.pyi`/`lvgl_api_summary.json` and the constraints from `MicroPythonOS/AGENTS.md`. It is not recommended as a standalone user-entry skill; it is more reasonable to be managed by `mpos-dev`'s API extraction/reference layer, and when necessary, explicitly pass `--lvgl-micropython-dir /home/leeqingshui/lvgl_micropython` during dependency preparation or deployment phases.

> App packaging

Already separated into `mpos-package-app`. This is a key risk point in the publishing pipeline and must be scripted to validate the MPK top-level directory, manifest, icon, and `app_index_entry.json`. Packaging is still allowed when testing is missing/failed, but must issue a warning.

> Linux simulation and App flashing

Already placed in `mpos-deploy-app`, but must be internally divided into independent modes:

- `desktop-preview`: Preview only, no smoke gate.
- `web-preview`: Defaults to serving existing artifacts; if artifacts are missing, asks the user if they want to build.
- `device-copy`: Controlled copy using the project's mpremote, must confirm board type and serial port.
- `mpk-install`: Recommended real-device release verification path.
- `install-site`: Firmware installation/erasure defaults to guiding the user to `https://install.micropythonos.com/`.
- `local-flash`: Disabled by default, unless the user explicitly says "allow local flash execution."

The phrase "flash App" is recommended to be changed to "install App to device" in the skill; "flash" is reserved for firmware images.

> Upload to upystore

Already separated into `mpos-publish-app`, but it only does publishing guidance + validation. It must read `package_result.json`, `app_test_result.json`, and `deploy_result.json` simultaneously, compare with the published version on upystore, and include store fields like screenshots, short/long description, hardware tags, release notes in `publish_result.json`. It recommends the user visit `https://upystore.io/developer` to upload themselves.

## Final Recommended User Experience

User says:

> Make a MicroPythonOS weather App that can display temperature and network status, help me package it and upload it to upystore.

Ideal flow:

1. `mpos-plan-app` establishes state, asks for the App name/fullname or gives a default suggestion.
2. `mpos-analyze-app` outputs functionality, manifest draft, dependencies, and test plan.
3. `mpos-prepare-deps` confirms whether a network API SDK is needed or if `DownloadManager` can be used.
4. `mpos-gen-app` checks or refreshes `mpos_api_summary.json` / `lvgl_api_summary.json` via `mpos-dev` to confirm the API reference is not outdated.
5. `mpos-gen-app` generates `internal_filesystem/apps/<fullname>/...`.
6. `mpos-gen-app` immediately runs manifest, syntax, MPY import, `make lint`, flake8, pylint checks and records `generation_result.json`.
7. `mpos-test-app` uses MicroPythonOS built-in desktop/controller tools for target App runtime smoke.
8. `mpos-package-app` generates and validates `.mpk`.
9. `mpos-deploy-app` writes `deploy_result.json` for desktop/web preview or real-device install; when no physical board is available, preview results can be used.
10. `mpos-publish-app` provides a publishing summary, store metadata, and upload guidance to `https://upystore.io/developer`.

With this decomposition, each skill is short, triggerable, testable, and conforms to the progressive disclosure principle of skill-creator.
