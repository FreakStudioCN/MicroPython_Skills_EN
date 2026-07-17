# MicroPythonOS Conversational Skill Decomposition Analysis

Date: 2026-07-14

This document provides analysis and design recommendations only; it does not create or modify skills. The analysis is based on the current contents of local directories
`/home/leeqingshui/MicroPythonOS`, `/home/leeqingshui/MicroPython_Skills`,
`/home/leeqingshui/lvgl_micropython`, and the 2026-07-14 reading of
`https://upystore.io/`, `https://install.micropythonos.com/`,
`https://docs.micropythonos.com/`, `https://web.micropythonos.com/`
including their body text, public APIs, installation manifests, and docs `search_index.json`.

## Conclusion

Conversational development for MicroPythonOS should not be implemented as a single monolithic skill. The correct form is:

1. One user-entry orchestration skill, responsible for conversation, phase switching, and state handover.
2. Multiple phase-specific skills, each handling requirements analysis, driver/dependency preparation, API reference refresh, App generation, testing/simulation, MPK packaging, device installation/flashing, and publishing guidance.
3. One shared base skill/reference layer, consolidating project facts about MicroPythonOS, LVGL, MPK, mpremote, simulators, etc.
4. Highly deterministic, error-prone, and repetitive actions should be implemented as scripts, e.g., API extraction, manifest validation, single-app MPK packaging, MPK structure validation, and device-side installation checks.

The recommended decomposition is 8 user-facing skills + 1 shared base skill:

| Recommended Skill | Type | Primary Responsibility | Current Status |
|---|---|---|---|
| `mpos-dev` | Shared Base | MicroPythonOS/LVGL/API constraints, API extraction script entry point | Existing, reference routing, LVGL independent repo path, API MD/JSON dual format completed |
| `mpos-plan-app` | User Entry/Orchestration | Conversational requirement clarification, phase state, calling downstream skills | Recommended new |
| `mpos-analyze-app` | Phase Skill | Requirements analysis, App type, functional boundaries, manifest draft, hardware/network/storage risks | Recommended new |
| `mpos-prepare-deps` | Phase Skill | Driver/dependency download, information retrieval, missing driver handling, resource preparation | Recommended new |
| `mpos-gen-app` | Phase Skill | Generate or modify MPOS App code, manifest, resources | Existing, but needs enhancement |
| `mpos-test-app` | Phase Skill | Syntax, unit, graphical testing, screenshot/controller verification | Existing, can be retained and extended |
| `mpos-package-app` | Phase Skill | Single App MPK packaging, app_index fragment, MPK format validation | Recommended new |
| `mpos-deploy-app` | Phase Skill | Linux simulation, install App to device, flash firmware if necessary | Recommended new, partial capability in `mpos-debug-app`/mpremote skills |
| `mpos-publish-app` | Phase Skill | Pre-publish checks, upystore upload guidance, post-upload verification suggestions | Recommended new, upload itself provides link |

After this round of re-reading the local repositories, four API references, `upystore.io`, `install.micropythonos.com`, and the entire docs site, this decomposition does not need to change; what needs strengthening is the downstream skills' usage rules for API references, especially that LVGL's `type_aliases[]` can only explain signatures and cannot be used to generate runtime API code.

This decomposition is more suitable than directly reusing the old `upy-*` pipeline. `upy-*` primarily targets "natural language generation of ordinary MicroPython hardware projects," while MicroPythonOS's core objects are Apps, Activities, MANIFEST, MPK, AppStore, desktop simulation, and system images, with different lifecycles and deliverables.

## Key Facts Observed

### Local MicroPythonOS Facts

- `MicroPythonOS/AGENTS.md` describes the project as MicroPythonOS with AppStore, OTA, and built-in apps. The main code is in `internal_filesystem/`, the build is based on `lvgl_micropython/`, and native MicroPython module implementation source code is in `c_mpos/`.
- Recommended command entry points include `make build-mpos-unix`, `make syntax-tests`, `make unittest-tests`, `make tests`, `make lint`, `make lint-fix`.
- The desktop simulation entry point is `scripts/run_desktop.sh`. AGENTS requires using `timeout -s 9 30 ./scripts/run_desktop.sh` for debugging runs.
- The firmware build entry point is `scripts/build_mpos.sh <target>`, supporting `unix`, `macOS`, `esp32`, `esp32-small`, `esp32s3`, `unphone`, `lilygo_t4` and other targets.
- The device App installation entry point is `scripts/install.sh com.micropythonos.<appname>`, which uses `lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py` under the hood.
- The firmware USB flashing entry point is `scripts/flash_over_usb.sh`, which writes to `lvgl_micropython/build/lvgl_micropy_ESP32_GENERIC_S3-SPIRAM_OCT-16.bin` by default.
- The App packaging entry point is `scripts/bundle_apps.sh`, which generates `../apps/app_index.json`, MPK, and icon URLs from `internal_filesystem/apps`.
- The actual App directory structure is typically:

```text
internal_filesystem/apps/com.example.app/
  META-INF/MANIFEST.JSON
  assets/main.py
  res/mipmap-mdpi/icon_64x64.png
```

The existing `com.micropythonos.helloworld/META-INF/MANIFEST.JSON` has `entrypoint` set to `assets/hello.py`, which is closer to the repository reality than the "Activity file in the app root directory" hint in the current `mpos-gen-app`.
- The online docs' `Creating Apps` page currently shows a flat structure example with `MANIFEST.JSON`, `icon_64x64.png`, `hello.py`; however, the local repository `internal_filesystem/apps/*` and `tests/test_apps_manifest.py` currently enforce `META-INF/MANIFEST.JSON`, and entry point files are generally in `assets/*.py`. Therefore, skill generation must adhere to the current repository tests and existing apps, while recording in `mpos-dev` reference that "the docs example is inconsistent with the current repository state."

### App and MPK Constraints

- `tests/test_apps_manifest.py` validates:
  - The App directory name must equal the `fullname` in `MANIFEST.JSON`.
  - `name` and `version` are required.
  - `version` must be a canonical integer dot-separated form, e.g., `0.1.6`.
  - Each activity/service `entrypoint` must end with `.py` and the file must exist.
  - The `classname` must be found in the entrypoint source code.
- `internal_filesystem/lib/mpos/content/streaming_unzip.py` is strict about MPK:
  - The first ZIP local header must be the `{fullname}/` directory.
  - There must be only one top-level directory, and all files must be under it.
  - Supports `ZIP_STORED` and `ZIP_DEFLATED`.
  - Data descriptor flag is not allowed.
  - Non-compliance results in a direct `RuntimeError`.
- `tests/test_streaming_unzip.py` covers correct MPK, no top-level directory, wrong top-level directory, mixed top-level directories, insufficient space, etc.

Therefore, `mpos-package-app` cannot simply "zip the directory." It must reliably produce an MPK with `{fullname}/` as the first directory entry and exclude irrelevant files like `__MACOSX/`, `._*`, `.git/`, etc.

### API Extraction Script Current Status

`mpos-dev` already has two key scripts:

- `mpos-dev/scripts/extract_mpos_api.py`
  - Scans native MicroPython modules and outputs the callable interface form for MPY users.
  - Scans Python framework classes, methods, functions, and docstrings in `internal_filesystem/lib/mpos/`.
  - Outputs `reference/mpos-api-reference.md`.
- `mpos-dev/scripts/extract_lvgl_api.py`
  - Reads `lvgl_micropython/lvgl.pyi` by default, only regenerates the stub if necessary.
  - Parses the LVGL Python API and outputs a structured summary.

This indicates that the need for "code generation requires scripts to extract MicroPythonOS and lvgl_micropython APIs" already has a foundation and should not be re-implemented in `mpos-gen-app`. It is recommended to make API refreshing a shared script capability of `mpos-dev`, called by `mpos-plan-app` or `mpos-gen-app` when needed.

2026-07-14 Review Result: `extract_lvgl_api.py`'s current default candidates already include `~/lvgl_micropython` and `/home/leeqingshui/lvgl_micropython`, and `reference/lvgl_api_summary.json`'s `source.path` already points to `/home/leeqingshui/lvgl_micropython/lvgl.pyi`. This gap has been filled and should no longer be listed as a to-do.

2026-07-14 Subsequent Update: `extract_mpos_api.py` has output `reference/mpos-api-reference.md` and `reference/mpos_api_summary.json`, covering native MicroPython modules, `mpos.__all__` root exports, and a full public API index of `internal_filesystem/lib/mpos/**/*.py` source code; native modules only show MPY import/call form in JSON/MD, without exposing implementation source file paths or underlying signatures. `extract_lvgl_api.py` has output `reference/lvgl-api-reference.md` and `reference/lvgl_api_summary.json`. Non-widget LVGL objects are uniformly marked as `data_classes` / `data_class`, real enum class/members go into `symbols[]`, and `*_t = int` stub type aliases are placed separately in `type_aliases[]` with mapping to runtime enum classes where possible. Both JSONs contain `generated_at`, `generator`, `counts`, `symbols[]`, allowing downstream skills to determine if the reference is outdated.

2026-07-14 API Reference Review Update: `mpos_api_summary.json` is currently generated by `extract_mpos_api.py v3`, with statistics showing 5 native MicroPython modules, 1206 symbols, `root_export_missing=0`; `lvgl_api_summary.json` is currently generated by `extract_lvgl_api.py v5`, with source pointing to `/home/leeqingshui/lvgl_micropython/lvgl.pyi`, statistics showing 60 `type_aliases`, 90 enum classes, 873 enum members, 79 data classes, 41 widget classes, 247 standalone functions, 3715 symbols. It has been verified that neither reference/JSON contains leaks of C/C++ implementation details like `mp_obj_t`, `mp_map_t`, `c_binding`, `C type`, `c_mpos`; nor misleading runtime API symbols like `lv.display_render_mode_t`, `lv.grad_dir_t`, `lv.event_code_t`, `lv.fs_whence_t`.

### Review of Four API Summary Documents

These four files are the core factual sources for subsequent natural language generation of MicroPythonOS Apps:

- `reference/mpos-api-reference.md`
- `reference/mpos_api_summary.json`
- `reference/lvgl-api-reference.md`
- `reference/lvgl_api_summary.json`

Current assessment: These four files can continue to serve as the API basis for `mpos-gen-app` / `mpos-analyze-app` / `mpos-test-app`, but downstream skills must read them according to the JSON schema, not just perform string searches.

Usage constraints:

- The MPOS side should only be used as an index of interfaces that MicroPython users can import/call. Native modules should only use the MPY call form of `adc_mic`, `pdm_mic`, `qrdecode`, `rvswd`, `webcam`, and should not reverse-engineer C functions from `c_mpos`.
- The LVGL side should use `symbols[]` as the primary index for generatable APIs, especially `kind == "enum"` / `kind == "enum_member"` / `kind == "widget"` / `kind == "function"`.
- LVGL's `type_aliases[]` should only be used to explain signature types. `runtime_api: false` means `lv.<alias>` cannot be generated; if `runtime_enum` exists, the corresponding enum class member should be generated, e.g., `event_code_t -> lv.EVENT.CLICKED`, `display_render_mode_t -> lv.DISPLAY_RENDER_MODE.PARTIAL`, `grad_dir_t -> lv.GRAD_DIR.VER`, `fs_whence_t -> lv.FS_SEEK.SET`.
- The continued appearance of `"display_render_mode_t"`, `"event_code_t"`, `"grad_dir_t"` in method signatures is normal type annotation and does not mean these names are runtime APIs. Conversely, `lv.area_t()`, `lv.style_t()`, `lv.anim_t()` and similar `*_t` data classes are real MPY classes/constructors and should not be blanket deleted because of the `_t` suffix.
- When `description` is empty, downstream skills should not fabricate semantics; if explanation is needed, first check the docs reference, current repository code, or specific source code context.
- Before generating code, prioritize running or checking `generated_at`/`generator`. If the API reference is found to be outdated, call `mpos-dev/scripts/extract_mpos_api.py` and `mpos-dev/scripts/extract_lvgl_api.py --lvgl-micropython-dir /home/leeqingshui/lvgl_micropython`.

Impact on skill decomposition:

- `mpos-dev` retains the API extraction scripts and four references, no longer stuffing large API tables into `SKILL.md`.
- `mpos-analyze-app` uses the API JSON to determine if existing managers/frameworks can meet requirements, avoiding premature driver downloads.
- `mpos-gen-app` must read or query the four API references before generation, especially LVGL enum/members, to avoid generating erroneous code like `lv.OBJ_FLAG.*`, `lv.EVENT_VALUE_CHANGED`, `lv.scr_act()`, `lv.display_render_mode_t`.
- `mpos-test-app` and `mpos-debug-app` can use the API JSON for candidate API retrieval during error fixing, but must not treat type annotations in the JSON as runtime properties.

### lvgl_micropython Facts

`/home/leeqingshui/lvgl_micropython/README.md` explains:

- This is an LVGL binding independent of the old `lv_micropython`, built via `python3 make.py <target> ...`.
- Information from the official binding should not be applied to this binding.
- Do not manually add submodule init commands.
- Supports SDL2 special drivers for unix/macOS, as well as multiple display/touch ICs.

MicroPythonOS skills must prioritize using the `lvgl_api_summary.json` generated from this repository, the independent `/home/leeqingshui/lvgl_micropython/lvgl.pyi`, and the AGENTS rules within the project, rather than generalized LVGL/MicroPython memory.

It is not recommended to make `/home/leeqingshui/lvgl_micropython` a separate user-facing skill; it is better suited as an API/reference source for `mpos-dev`, and for targeted reading by `mpos-prepare-deps` or `mpos-deploy-app` when dealing with display/touch drivers, board ports, or binding builds.

### Public Site Access Status

2026-07-14 Re-reading Results:

- `https://upystore.io/` main site returns `200`; `https://upystore.io/apps` returns `200`, fetched 30373 bytes; `https://upystore.io/developer` shows the Developer Console login/publish page when not logged in, returns `200`, fetched 8267 bytes, and includes `submit MPK` text; `https://upystore.io/app_index.json` returns `200`, fetched 5946 bytes; `https://upystore.io/api/v1/apps` returns `200`, fetched 10843 bytes.
- `upystore.io/app_index.json` is currently a list of 10 apps, with fields including `activities`, `category`, `download_url`, `fullname`, `icon_url`, `long_description`, `name`, `publisher`, `short_description`, `version`. `api/v1/apps` currently returns `apps`, `filters`, `pagination`, with pagination showing `total=10`, `total_pages=1`, and additionally includes `slug`, `revision`, `tags`, `hardware_tags`, `min_os_version`, `min_api_level`, `screenshots`, `installs_count`, `downloads_count`, `stars_count`, `released_at` and other storefront fields.
- This round also read the public app detail pages for each of the 10 `slug`s from `api/v1/apps`, resulting in `UPYSTORE_DETAIL_OK=10/10`; the detail pages contain the corresponding `fullname`, `name`, `version`, which can be used for manual verification after publishing but should not replace local MPK/manifest validation.
- `https://install.micropythonos.com/` currently returns `200` directly via HTTPS, fetched 17558 bytes. The page uses `esp-web-install-button`, includes `WebSerial`, `USB`, `ESP32`, `ESP32-S3`, `0.15.x`, etc.; the previously recorded `SSL_ERROR_SYSCALL` status for this VM is now outdated.
- The Installer currently lists 12 manifests for ESP32-S3 and ESP32 across versions `0.10.x`, `0.11.x`, `0.12.x`, `0.13.x`, `0.14.x`, `0.15.x`. All manifests have been read individually, all returning `200` with `new_install_prompt_erase=true`; the latest `0.15.x` corresponds to version `0.15.1`, with firmware paths pointing to `/firmware_images/esp32s3/MicroPythonOS_esp32s3_0.15.1.bin` and `/firmware_images/esp32/MicroPythonOS_esp32_0.15.1.bin` respectively.
- `https://docs.micropythonos.com/` currently returns `200` directly via HTTPS, fetched 39201 bytes; `https://docs.micropythonos.com/sitemap.xml` returns `200`, fetched 10794 bytes; `https://docs.micropythonos.com/search/search_index.json` returns `200`, fetched 775522 bytes. The previously recorded `SSL_ERROR_SYSCALL` status for this VM is now outdated.
- The `lastmod` for pages in the docs sitemap is `2026-07-13`. This round read all 61 pages from the sitemap individually, resulting in `DOCS_FETCH_OK=61/61 failed=0`; the re-fetched `search_index.json` parsed 977 document/section entries, covering `Apps`, `App Lifecycle`, `Bundling Apps`, `AppStore`, `Frameworks`, `Running`, `Supported Hardware`, `DownloadManager`, `TaskManager`, `SharedPreferences`, `AppManager`, `Service` and other content directly related to skill decomposition.
- `https://web.micropythonos.com/` returns `200`, fetched 18555 bytes, title is `MicroPythonOS Web`; the page loads `micropython.js`, runs MicroPythonOS in the browser, with a default LVGL canvas of `320x240`, and provides controls for Log, Reset storage, joystick, MENU/START, X/Y/A/B, NeoPixel simulation, etc.

### docs.micropythonos.com Facts

After reading the docs homepage, sitemap, and `search_index.json` on 2026-07-14, the facts directly relevant to skill decomposition are:

- The docs homepage defines MicroPythonOS as a lightweight operating system built entirely with MicroPython, targeting ESP32, desktop, and browser, featuring an Android-inspired UI, App ecosystem, App Store, and OTA updates.
- `Getting Started / Running` explains that pre-built firmware installation goes through the WebSerial installer at `install.micropythonos.com`; the desktop side can run pre-built binaries, use source code checkout for app development, or build from source; the Web side can open `web.micropythonos.com` or the main branch WebAssembly build.
- `Supported Hardware` covers ESP32, ESP32-S3, WebAssembly, Linux/macOS/Windows WSL2/Raspberry Pi and other targets, supporting placing "Linux simulation" and "firmware flashing" under the same `mpos-deploy-app` but as different paths.
- `Apps / Creating Apps` states that apps are installed in `/apps/`, the manifest declares `activities` and `services`, and services can use `boot_completed` to run automatically after startup; however, the flat directory example on this page is inconsistent with the current local repository, so skills must prioritize adhering to local tests.
- `Apps / App Lifecycle` clearly defines the Activity lifecycle including `onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy`, where an Activity is a UI screen; a Service is a background component without UI. This supports identifying Activity/Service separately during requirements analysis, code generation, and testing.
- `Apps / Bundling Apps` clearly states that `.mpk` is a ZIP archive, the first entry must be the app fullname top-level directory, and there can only be one top-level directory; this is consistent with the local `StreamingUnzip` and `tests/test_streaming_unzip.py`.
- `Apps / AppStore` explains that the AppStore can pull apps from multiple backends and installs apps as `.mpk` into `/apps/`. This supports `mpos-publish-app` only preparing and validating the package, without mixing upystore upload into the device installation logic.
- `Frameworks` documentation recommends that applications import frameworks directly from the main `mpos` module, e.g., `from mpos import AppManager, DisplayMetrics`, avoiding imports from submodules like `mpos.ui`, `mpos.content`; pages like `DownloadManager`, `TaskManager`, `SharedPreferences`, `AppManager`, `Service` provide API types that need to be consulted during the code generation phase.

Note: This round read all 61 pages of the docs according to the sitemap and simultaneously used the 977 page/section entries from `search_index.json` for coverage validation; however, the full text of the 61 pages was not transcribed verbatim into offline reference files. For skills, the approach of "stuffing the entire site text directly into `SKILL.md`" is also inappropriate.

The docs content has currently been split into topic-specific reference files under `mpos-dev/reference/`, rather than creating one giant reference:

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

`SKILL.md` only retains the routing rules for "when to read which reference." This aligns with progressive disclosure: when generating an App, only read the app model and framework summary; when packaging, only read packaging; when deploying/simulating, only read deploy targets and web port.

### web.micropythonos.com Facts

`https://web.micropythonos.com/` is the browser/WebAssembly runtime entry point for MicroPythonOS, not an installation site or app publishing site. Page facts read on 2026-07-14:

- The page title is `MicroPythonOS Web`, loads `micropython.js`, and runs MicroPythonOS in the browser via Emscripten.
- The page displays a default `320x240` LVGL canvas, corresponding to the MicroPythonOS default display profile.
- The page provides simulated badge peripherals: NeoPixel indicator light, joystick, MENU, START, X/Y/A/B buttons; these interact with the Python-side `_webio`/fake WebExpander via `Module.__webio`.
- The page mounts `/data` and `/apps` to IndexedDB/IDBFS, allowing app preferences and user-installed apps to persist across refreshes.
- The page provides a `Reset storage` button that deletes persistent `/data` and `/apps` and reloads the page.
- The runtime parameters are `["-X", "heapsize=16M", "-m", "main"]`, i.e., starting the frozen `main` module with a 16MB heap.
- This entry point is suitable for inclusion in `mpos-deploy-app`/`mpos-test-app` reference materials: for browser-side smoke tests, user quick previews, and Web port limitation explanations; it should not replace Linux SDL desktop simulation or real device verification.

### upystore.io Facts

The main site, Apps page, Developer login page, `app_index.json`, and `api/v1/apps` read on 2026-07-14 show:

- upystore is an app store for MicroPython/uPyOS hardware. The main site navigation includes Home, Apps, Developer Console, Log in, Sign up, and English/Chinese language switching.
- The homepage emphasizes the three steps of Browse, Install, Publish, targeting D-Shell/uPyOS devices, supporting app categories, version history, hardware profiling/matching, installation statistics, publisher information, icons, screenshots, release notes, and hardware requirements.
- The Apps page currently displays 10 apps, with categories including IoT, Motor Control, AI, Communication, Games, Media, Education, Development Tools, Utilities, and provides search, category filtering, download count, install count, stars, version number, and other fields.
- The Developer Console, when not logged in, goes to a login page. The page text clearly states that a developer account is required to submit MPK packages, manage apps, and review app statistics. Therefore, `mpos-publish-app` should not automatically log in or request account credentials; it should only provide the upload link and pre/post-publish verification checklists.
- `app_index.json` currently includes fields like `name`, `publisher`, `short_description`, `long_description`, `icon_url`, `download_url`, `fullname`, `version`, `category`, `activities`, etc., with a format generally close to what the MicroPythonOS AppStore requires.
- Current upystore data shows two forms of `activities`: `Show Battery` and `Danke` use full object arrays (`classname`, `entrypoint`, `intent_filters`), while some seed/sample apps use string arrays (e.g., `clock`, `timer`). The publishing skill must output the full manifest form when generating metadata and should not copy the string activity form from seed data.
- `api/v1/apps` includes more storefront fields than `app_index.json`, such as `slug`, `revision`, `tags`, `hardware_tags`, `min_os_version`, `min_api_level`, `screenshots`, `installs_count`, `downloads_count`, `stars_count`, `released_at`, `pagination`; these are suitable for publishing summaries and post-upload verification but should not replace the local `MANIFEST.JSON`.
- upystore upload itself should remain a user action: the skill prepares the MPK, icon, metadata summary, and validation results, then recommends the user visit `https://upystore.io/` or the Developer Console to upload. After upload, if the user provides a download URL or app_index/API return value, the skill can then perform MPK top-level directory and device-side installation verification.
- The old conclusion from the local `MicroPythonOS/docs/upystore-integration-analysis_CN.md` remains critical: the device side should not relax `StreamingUnzip` validation; the packaging/upload side must ensure the first ZIP entry of the MPK is `{fullname}/` and exclude junk files like `__MACOSX/`, `._*`.

## Basic Approach to Conversational Skills

Conversational does not mean writing very long explanations; it means each phase has clear input, output, and next steps. It is recommended that each phase uses a lightweight state object, for example:

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
    "manifest": "internal_filesystem/apps/com.example.weather/META-INF/MANIFEST.JSON",
    "mpk": null
  },
  "open_questions": [],
  "verification": []
}
```

Principles:

- The orchestration skill only asks necessary blocking questions, not a long questionnaire all at once.
- Each phase skill receives upstream state and produces handoverable artifacts, not just natural language conclusions.
- When the user explicitly wants "from one sentence to completion," the entry skill calls phase skills sequentially; when the user explicitly only wants testing/packaging/publishing, trigger the corresponding skill directly.
- High-risk actions must be separated: code generation, firmware flashing, file system erasure, and upload publishing cannot be mixed into one skill and executed by default.
- `SKILL.md` should remain short; detailed rules go into `references/`; deterministic actions go into `scripts/`.

## Recommended Skill Design

### 1. `mpos-dev`: Shared Base Layer

Positioning: The knowledge base for all MicroPythonOS App skills, not a primary user entry point.

Content to retain:

- MicroPythonOS directory structure, Activity/App/Service lifecycle.
- LVGL 9.x/micropython binding constraints.
- Native MicroPython module API quick reference.
- Global constraints: ruff double quotes, temporary files in project `tmp/`, desktop run with timeout, debug process kill with `killall`.
- `extract_mpos_api.py` and `extract_lvgl_api.py`.

Completed or should be maintained:

- App structure examples should continue to be based on the current repository's actual structure: `META-INF/MANIFEST.JSON`, `assets/*.py`, `res/mipmap-mdpi/icon_64x64.png`.
- Already recorded in `reference/docs-app-model.md` / `reference/docs-packaging.md` that the online docs' current flat app example is inconsistent with local repository/test constraints; generation and packaging should follow local tests, with docs serving only as background reference.
- Clarify that `from mpos import Activity` is the recommended unified import method from the docs, and it can also be imported from `mpos.app.activity import Activity`; new skills should prioritize using the main `mpos` re-export, unless the current code has local conventions.
- The default path for `extract_lvgl_api.py` already covers `/home/leeqingshui/lvgl_micropython`; when involving other binding checkouts, `--lvgl-micropython-dir` can still be passed explicitly.
- MicroPythonOS and LVGL API references now have both MD/JSON dual formats; JSON includes generation time, statistics, and symbol index, making it easy to determine if outdated.

Current/recommended resources:

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
  reference/lvgl-rules.md      # Optional: split out if LVGL rules in SKILL.md continue to grow
  scripts/extract_mpos_api.py
  scripts/extract_lvgl_api.py
```

### 2. `mpos-plan-app`: Conversational Entry/Orchestration

Trigger:

- User says "help me make a MicroPythonOS App."
- User only provides a natural language functional description and wants to go from requirements to generation, testing, packaging, deployment.
- User asks "what's the next step" or "continue completing this app."

Responsibilities:

- Convert user natural language into phase state.
- Choose whether to enter requirements analysis, dependency preparation, code generation, testing, packaging, deployment, publishing.
- Maintain open questions, but only ask truly blocking ones, e.g., App name/fullname, whether hardware is needed, target device, whether flashing is allowed.
- Call downstream skills, do not implement all details directly.

Does not do:

- Does not directly download unknown drivers.
- Does not directly flash.
- Does not automatically upload to upystore.

Output:

- Current phase.
- Paths to artifacts like App directory/manifest/tests/MPK.
- Next step suggestions and high-risk actions requiring user confirmation.

### 3. `mpos-analyze-app`: Requirements Analysis

Trigger:

- User has an App idea but no clear functional boundaries.
- Need to translate requirements into manifest, Activity, Service, data persistence, hardware permissions/dependency plan.

Responsibilities:

- Identify App type: pure UI, tool, game, network, audio, camera, sensor, background service.
- Produce minimum viable features, non-goals, risk points.
- Suggest `fullname`, `name`, `category`, `activities`, `services`.
- Determine if needed:
  - `SharedPreferences`
  - `TaskManager`
  - `DownloadManager`
  - `CameraManager`
  - `AudioManager`
  - `SensorManager`
  - External MicroPython driver
  - C module or firmware recompilation
- Clarify testing strategy: normal unittest, GraphicalTestCase, manual hardware testing, device-side verification.

Output suggestion:

```json
{
  "manifest_draft": {},
  "feature_slices": [],
  "dependency_plan": [],
  "test_plan": [],
  "blocking_questions": []
}
```

### 4. `mpos-prepare-deps`: Driver Download and Dependency Preparation

Trigger:

- App requires external sensor/display/network/service SDK.
- User says "download driver," "find library," "this hardware has no driver."
- `mpos-analyze-app` marks a dependency gap.

Responsibilities:

- Prioritize checking MicroPythonOS built-in capabilities and managers; do not redundantly introduce external drivers.
- When a MicroPython driver is needed, reuse the existing `fetch-doc`, `upy-pkg-guide`, `upy-gen-driver` capabilities for information retrieval and missing driver handling.
- Download or organize dependencies into the App's `assets/` or shared `lib/`, and record source, version, license.
- Determine if firmware recompilation is necessary: if it's a Python driver, usually not; only if it's a C module, LVGL/display binding, or firmware component, then enter the firmware build/flash chain.

Boundary:

- This is not `upy-select-hw`. MicroPythonOS Apps typically run on existing devices/system images and should not default to redoing MCU/pin selection.
- When peripherals are related to pins and target boards, require the user to provide the device/board, or read `internal_filesystem/lib/mpos/board/*.py`.

Suggested scripts:

- `scripts/vendor_python_module.py`: Place single-file/directory drivers into app assets or lib, and generate source records.
- `scripts/check_dependency_imports.py`: Statically check if the imports in the App entrypoint can be found in the MPOS tree or vendor directory.

### 5. `mpos-gen-app`: Code Generation

Trigger:

- User wants to create/modify a MicroPythonOS App.
- Upstream already has requirements analysis and dependency plan.

Responsibilities:

- Create or modify `internal_filesystem/apps/<fullname>/`.
- Generate `META-INF/MANIFEST.JSON`.
- Generate Activity/Service code in `assets/*.py`.
- Create or complete `res/mipmap-mdpi/icon_64x64.png`.
- Use `mpos-dev`'s API reference and LVGL rules.
- Strictly adhere to this repository's LVGL constraints for UI code.

Generation rules that must be corrected:

- The current directory example in `mpos-gen-app/SKILL.md` still places Activity/Service files in the app root directory, and the manifest example's `entrypoint` lacks `.py`; this would directly generate an app that fails `tests/test_apps_manifest.py` and must be fixed first.
- `entrypoint` should prioritize using `assets/main.py` or `assets/<app>.py`, must include the `.py` suffix, and ensure the file exists.
- The Activity class name must appear in the entrypoint code.
- If customizing `__init__`, must call `super().__init__()`.
- New labels must explicitly call `set_text("")` or set the target text.
- After `style_t()`, must call `init()`.
- Do not hardcode screen resolution; prioritize `lv.pct(100)`, flex, align.
- Use `SharedPreferences(self.appFullName)` for persistence.
- Use `TaskManager` for background/asynchronous tasks; UI updates need to be on the foreground or main thread safe path.

Suggested resources:

```text
mpos-gen-app/
  SKILL.md
  references/app-patterns.md
  references/lifecycle.md
  assets/templates/basic_app/
  assets/templates/service_app/
  scripts/validate_manifest.py
```

### 6. `mpos-test-app`: Testing and Quality Gate

The existing skill's basic direction is correct.

Trigger:

- User wants to verify an App, write tests, reproduce before fixing bugs.
- Automatically enter the testing phase after generating or modifying an App.

Responsibilities:

- Run or guide running:
  - `make lint`
  - `make syntax-tests`
  - `./tests/unittest.sh`
  - Individual test files
  - Graphical tests
- Use `mpos.ui.testing.GraphicalTestCase`, `KeyboardTestCase`.
- Use `scripts/mpos_controller.py` for desktop or serial automation.
- For screenshots, use widget tree/visible text/pixel checks, not just visual description.

Suggested enhancements:

- Add a "minimum test template for new Apps."
- Add a quick path for "how to fix manifest test failures."
- Add a reference to "validate with `test_streaming_unzip` rules after MPK packaging," but actual packaging validation belongs to `mpos-package-app`.

### 7. `mpos-package-app`: App Packaging

Trigger:

- User says "package app," "generate MPK," "prepare for AppStore/upystore upload."
- Enter publishing preparation after code generation and testing are complete.

Responsibilities:

- Read `internal_filesystem/apps/<fullname>/META-INF/MANIFEST.JSON`.
- Validate manifest and file structure.
- Ensure icon exists at path `res/mipmap-mdpi/icon_64x64.png`.
- Generate single App MPK.
- Generate or output app_index entry, including:
  - `icon_url`
  - `download_url`
  - `fullname`
  - `version`
  - `activities`
  - `services`
- Verify that the first ZIP entry of the MPK is the `{fullname}/` directory.

Why an independent skill is needed:

- `scripts/bundle_apps.sh` is a full packaging script with blacklist and app store batch output logic, unsuitable as the sole entry point for "packaging only the current App" in a user conversation.
- MPK specifications are strict; erroneous packages will fail during device-side download. This phase requires deterministic scripts.

Suggested scripts:

```text
mpos-package-app/
  scripts/package_mpos_app.py
  scripts/validate_mpk.py
  scripts/emit_app_index_entry.py
```

`package_mpos_app.py` must ensure:

- The first entry inside the zip is `{fullname}/`.
- File order is stable.
- Modification times can be fixed for reproducible builds.
- Exclude `.git/`, `__pycache__/`, `*.pyc`, `__MACOSX/`, `._*`.
- Optionally use stored or deflated, but must be within `StreamingUnzip` support range.

### 8. `mpos-deploy-app`: Linux Simulation, Device Installation, and Firmware Flashing

Trigger:

- User says "simulate on Linux," "run desktop simulator," "install to device," "flash firmware," "reflash."
- Need real runtime verification before or after packaging.

Responsibility breakdown:

- Desktop simulation:
  - Confirm `lvgl_micropython/build/lvgl_micropy_unix` exists.
  - If not, guide or run `make build-mpos-unix`.
  - Use `timeout -s 9 30 ./scripts/run_desktop.sh <app_fullname>`.
  - If interaction is needed, use `scripts/mpos_controller.py`.
- App installation to device:
  - Use `scripts/install.sh <fullname>` or mpremote single file copy.
  - After installation, remind to execute `AppManager.refresh_apps()` or reboot.
  - Reuse `mpremote-device-interaction`, `mpremote-file-transfer`, `mpremote-live-session`.
- Firmware flashing:
  - Only enter when firmware does not exist, C modules have changed, system image has changed, or the user explicitly requests flashing.
  - Use `scripts/build_mpos.sh <target>` and `scripts/flash_over_usb.sh`.
  - Flashing, erasing, and resetting all require explicit confirmation.

Boundary:

- "Install App" is not "flash firmware." The vast majority of Python App iterations only require copying the app directory or installing an MPK.
- "Linux simulation" is not a PC-side mock project; it runs the MicroPythonOS unix build and SDL LVGL.

### 9. `mpos-publish-app`: upystore Publishing Guidance

Trigger:

- User says "upload to upystore," "publish App," "prepare for AppStore listing."

Responsibilities:

- Confirm that the following have passed:
  - Manifest validation
  - Testing
  - MPK format validation
  - Icon exists
  - Version number incremented
- Output the publishing package path and metadata summary.
- Clarify that the upystore Developer Console requires the user to log in with a developer account; the skill does not request or save account credentials.
- Guide the user to open `https://upystore.io/` or the Developer Console to upload.
- Inform about suggested post-upload verification:
  - Whether the fields in app_index are complete.
  - Whether the downloaded MPK still retains the `{fullname}/` top-level directory.
  - Whether the device-side AppStore can install it.

Does not do:

- Does not automatically upload on behalf of the user.
- Does not save or request upystore account credentials.
- Does not treat upystore as a firmware publishing site. Firmware publishing and `install.micropythonos.com` belong to a different workflow.

## Reuse and Adjustment Suggestions for Existing Skills

### Existing `mpos-*`

The current `MicroPython_Skills` already includes:

- `mpos-dev`
- `mpos-gen-app`
- `mpos-test-app`
- `mpos-debug-app`

These can serve as the foundation for the new system, but are not yet complete:

- Missing conversational entry/orchestration.
- Missing requirements analysis phase.
- Missing driver/dependency preparation phase.
- Missing single App MPK packaging phase.
- Missing a deployment phase that separates "install App" and "flash firmware."
- Missing upystore publishing guidance phase.
- The App structure example in `mpos-gen-app` needs to be aligned with the actual `assets/*.py` pattern, and the manifest example's `entrypoint` must include the `.py` suffix.

### Role of Old `upy-*`

Can be reused but should not be directly used as the main MicroPythonOS chain:

- `fetch-doc`: Can be used for driver information, GitHub, URL content supplementation.
- `upy-pkg-guide`: Can be used for MicroPython driver package usage queries.
- `upy-gen-driver` / `upy-gen-driver-plugin`: Can be used for the missing driver branch.
- `mpremote-*`: Can be used for device connection, file copying, long sessions.
- `upy-deploy`: The concept can be referenced, but MicroPythonOS App installation should prioritize `scripts/install.sh`, MPK, and AppManager workflow.

Parts not recommended for direct reuse as the main MPOS chain:

- `upy-select-hw`: It targets MCU/pin/firmware selection; MicroPythonOS Apps already have a target system by default.
- `upy-scaffold`/`upy-generate`: They generate ordinary MicroPython firmware projects, not equivalent to MPOS App's Activity/manifest/MPK.
- `upy-simulate`: It is a PC CLI/rich simulation, not equivalent to MicroPythonOS unix SDL desktop simulation.

## Recommended Implementation Order

1. First, fix and consolidate `mpos-dev`.
   - Align with App directory facts.
   - Ensure API extraction scripts are runnable and update references.
2. Enhance `mpos-gen-app`.
   - Add manifest validator.
   - Add basic App templates.
   - Generate `assets/*.py`.
3. Add new `mpos-package-app`.
   - This is the most error-prone part of the publishing chain and the most suitable for scripting.
4. Add new `mpos-deploy-app`.
   - Clearly define three paths: desktop simulation, App installation, firmware flashing.
5. Add new `mpos-analyze-app` and `mpos-plan-app`.
   - Only after the preceding phases are stable, create the conversational orchestration entry point, to avoid the entry skill only being able to "verbally plan."
6. Add new `mpos-prepare-deps`.
   - Reuse `fetch-doc`, `upy-pkg-guide`, `upy-gen-driver`.
7. Add new `mpos-publish-app`.
   - Only do pre-publish checks, upload guidance, and post-upload verification; do not automatically upload.

## Template for Each `SKILL.md`

Each skill's `description` should include "what it does + when to trigger," as this is the primary basis for Codex to trigger the skill. The body should only contain necessary workflows and resource navigation.

Example:

```markdown
---
name: mpos-package-app
description: Package and validate MicroPythonOS Apps as MPK files for AppStore/upystore release. Use when Codex needs to create a .mpk, validate MANIFEST.JSON, emit app_index metadata, or prepare an MPOS App for publishing.
---

# MicroPythonOS App Packaging

## Workflow

1. Read `mpos-dev` for MPOS App and MPK constraints.
2. Locate `internal_filesystem/apps/<fullname>/META-INF/MANIFEST.JSON`.
3. Run `scripts/validate_manifest.py`.
4. Run `scripts/package_mpos_app.py --app <fullname>`.
5. Run `scripts/validate_mpk.py <mpk> --fullname <fullname>`.
6. Report MPK path and app_index metadata.

## Constraints

- The first ZIP entry must be `<fullname>/`.
- Exclude `__MACOSX/`, `._*`, `.git/`, `__pycache__/`, `*.pyc`.
- Do not publish or upload automatically.
```

## Minimum Script Checklist

To make skills truly executable, it is recommended to add at least these scripts:

| Script | Belongs to Skill | Purpose |
|---|---|---|
| `scripts/validate_manifest.py` | `mpos-gen-app` or `mpos-package-app` | Replicate single App validation from `test_apps_manifest.py` |
| `scripts/package_mpos_app.py` | `mpos-package-app` | Generate a standard MPK for a single App |
| `scripts/validate_mpk.py` | `mpos-package-app` | Check ZIP entry order, top-level directory, illegal files |
| `scripts/emit_app_index_entry.py` | `mpos-package-app` | Output app_index entry based on manifest and base URL |
| `scripts/check_dependency_imports.py` | `mpos-prepare-deps` | Statically check App import sources |
| `scripts/run_app_desktop.py` | `mpos-deploy-app` | Wrap `run_desktop.sh` + timeout + app launch + log collection |

Existing scripts to keep:

- `mpos-dev/scripts/extract_mpos_api.py`
- `mpos-dev/scripts/extract_lvgl_api.py`

## Point-by-Point Response to the User's Original Idea

> Requirements analysis

Should be an independent `mpos-analyze-app`, called by `mpos-plan-app`. It outputs a manifest draft, feature slices, dependency plan, and test plan, rather than directly writing code.

> Driver download

Should be an independent `mpos-prepare-deps`. However, by default, first check MicroPythonOS built-in managers and `mpos/board`; do not immediately download ordinary MicroPython drivers. When a driver is truly missing, reuse `fetch-doc`, `upy-pkg-guide`, `upy-gen-driver`.

> Code generation, needs scripts to extract MicroPythonOS and lvgl_micropython APIs

Code generation is still done by `mpos-gen-app`. API extraction scripts should be placed in the shared `mpos-dev`; the existing `extract_mpos_api.py` and `extract_lvgl_api.py` are already on the right track. The code generation skill should first read/refresh the references, then generate code.

> `/home/leeqingshui/lvgl_micropython`

It should serve as one of the factual sources for the LVGL binding, but do not directly apply official LVGL binding documentation. Prioritize using the locally generated `lvgl.pyi`/`lvgl_api_summary.json` and the constraints from `MicroPythonOS/AGENTS.md`. It is not recommended as a standalone user-facing skill; it is more reasonable to be managed by `mpos-dev`'s API extraction/reference layer, and when necessary, explicitly pass `--lvgl-micropython-dir /home/leeqingshui/lvgl_micropython` during the dependency preparation or deployment phase.

> App packaging

Should be an independent `mpos-package-app`. This is a key risk point in the publishing chain and must have scripted validation of the MPK top-level directory and manifest.

> Linux simulation and App flashing

It is recommended to merge into `mpos-deploy-app`, but internally it must have three separate paths:

- Linux desktop simulation: `make build-mpos-unix` + `timeout -s 9 30 ./scripts/run_desktop.sh <app>`.
- Install App to device: `scripts/install.sh <fullname>` or mpremote file copy.
- Flash firmware: only go through `build_mpos.sh`/`flash_over_usb.sh` when the user confirms a reflash is needed.

The term "flash App" should be changed to "install App to device" in the skill; "flash" should be reserved for firmware images.

> Upload to upystore

Should be an independent `mpos-publish-app`, but only do pre-publish checks, organize artifact paths, and provide upload link hints. Recommend the user visit `https://upystore.io/` to upload themselves. After upload, if the user provides a download URL or app_index, the skill can perform MPK structure and device-side installation verification.

## Final Recommended User Experience

User says:

> Make a MicroPythonOS weather App that can display temperature and network status, help me package it and upload to upystore.

Ideal workflow:

1. `mpos-plan-app` establishes state, asks for App name/fullname or provides a default suggestion.
2. `mpos-analyze-app` outputs features, manifest draft, dependencies, and test plan.
3. `mpos-prepare-deps` confirms whether a network API SDK is needed or if `DownloadManager` alone is sufficient.
4. `mpos-gen-app` checks or refreshes `mpos_api_summary.json` / `lvgl_api_summary.json` via `mpos-dev` to confirm the API reference is not outdated.
5. `mpos-gen-app` generates `internal_filesystem/apps/<fullname>/...`.
6. `mpos-test-app` runs lint/syntax/graphical tests.
7. `mpos-deploy-app` runs on Linux desktop simulation, and installs to device if necessary.
8. `mpos-package-app` generates and validates `.mpk`.
9. `mpos-publish-app` provides the publishing summary and upload guidance for `https://upystore.io/`.

With this decomposition, each skill is short, triggerable, testable, and aligns with the skill-creator's progressive disclosure principle.
