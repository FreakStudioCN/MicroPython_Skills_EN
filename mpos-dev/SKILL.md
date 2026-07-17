---
name: mpos-dev
description: MicroPythonOS basic development knowledge base. Provides code architecture, App/MPK constraints, LVGL programming conventions, MPY API reference, official docs topic reference, and AGENTS local constraints. All mpos-plan-app / mpos-analyze-app / mpos-prepare-deps / mpos-gen-app / mpos-debug-app / mpos-test-app / mpos-package-app / mpos-deploy-app / mpos-publish-app depend on this skill.
---

# MicroPythonOS Basic Development Knowledge Base

## Role

This is the shared foundation layer for the mpos-* skill family. Do not call this skill directly — use `mpos-plan-app` (dialogue orchestration), `mpos-analyze-app` (requirements analysis), `mpos-prepare-deps` (dependency preparation), `mpos-gen-app` (App generation), `mpos-debug-app` (App debugging), `mpos-test-app` (App testing), `mpos-package-app` (packaging), `mpos-deploy-app` (deployment/simulation/installation/flashing), or `mpos-publish-app` (publishing guidance).

## Unified Project Log

First determine the current MicroPythonOS repository root `<repo-root>`:

- When the user explicitly provides a repo path, that path must be used.
- Otherwise, if the current working directory contains `internal_filesystem/apps` and `scripts`, use the current working directory as `<repo-root>`.
- When testing in an isolated clone/worktree/temporary copy, artifacts must never be written back to the `/home/leeqingshui/MicroPythonOS` main repository.
- Build, simulator, desktop-preview, and web-preview should default to executing in an isolated clone/worktree/temporary copy; do not let these processes modify the main MicroPythonOS checkout unless the user explicitly allows it.

All mpos-* skills targeting a single App should maintain the same project state directory to facilitate interruption recovery and AI debugging:

```text
<repo-root>/tmp/mpos-plan-app/<fullname>/
  plan_state.json
  activity_log.jsonl
```

After a phase skill completes, register the artifacts with `mpos-plan-app` instead of scattering results across individual `tmp/mpos-*` directories:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-plan-app/scripts/update_plan_state.py record \
  --repo <repo-root> \
  --fullname <fullname> \
  --skill <mpos-skill-name> \
  --phase <phase> \
  --result <result> \
  --artifact <artifact_key>=<path> \
  --next-skill <next-skill-or-null> \
  --event "<short summary>"
```

Do not manually write `plan_state.json` or `activity_log.jsonl`; always call `update_plan_state.py record/discover/invalidate` to ensure `plan_state.json` uses the `mpos-plan-app-v1` schema, and validate it with `validate_plan_state.py` after updates.

Standard artifact keys: `analysis_result`, `dependency_handoff`, `generation_result`, `app_test_result`, `package_result`, `deploy_result`, `publish_result`.

If the user interrupts and later says "continue/resume/next step", first have `mpos-plan-app` read or rebuild `plan_state.json` — do not start from scratch.

## Codebase Architecture

```
MicroPythonOS/
├── c_mpos/src/              ← Native MicroPython module implementation source
│   ├── webcam.c             ← webcam.init(...) returns Webcam handle; module functions operate on that handle
│   ├── pdm_mic.c            ← PDM_Mic(clk, data, rate, bufsize)/start/stop/readinto/deinit
│   ├── adc_mic.c            ← adc_mic.read(...) function
│   ├── quirc_decode.c       ← qrdecode.qrdecode(buffer,width,height) / qrdecode_rgb565(...)
│   └── rvswd_module.c       ← RVSWD(swdio, swclk) debug/flash interface
├── internal_filesystem/
│   ├── lib/mpos/            ← Core framework (Python)
│   │   ├── app/             ← Activity/App/Service base classes
│   │   ├── content/         ← AppManager, Intent, streaming_unzip
│   │   ├── ui/              ← topmenu, keyboard, testing, appearance_manager, input_manager
│   │   ├── audio/           ← AudioManager, stream_wav, stream_rtttl, stream_record_*
│   │   ├── net/             ← wifi_service, download_manager, connectivity_manager
│   │   ├── config.py        ← SharedPreferences (persistent key-value storage)
│   │   ├── camera_manager.py← CameraManager singleton
│   │   ├── battery_manager.py← BatteryManager
│   │   ├── gps_manager.py   ← GPSManager
│   │   ├── activity_navigator.py ← ActivityNavigator (routing)
│   │   └── ...
│   ├── apps/                ← Installed Apps (one directory each)
│   └── main.py              ← System startup entry point
├── lvgl_micropython/        ← LVGL + MicroPython submodule
└── scripts/                 ← Build/flash/deploy scripts
```

## Reference File Routing (Consult as Needed)

Before writing any MicroPythonOS code, **you must first consult** the following API reference files to understand the available APIs:

| Reference File | Content | Generation Method |
|---------|------|---------|
| `reference/mpos-api-reference.md` | Human-readable version of MicroPythonOS user-callable APIs: native MicroPython modules, `mpos.__all__`, full source public API index | `python3 scripts/extract_mpos_api.py` |
| `reference/mpos_api_summary.json` | Machine-readable version of MicroPythonOS user-callable APIs, containing `generated_at`, `counts`, `source_index`, `symbols[]` | `python3 scripts/extract_mpos_api.py` |
| `reference/lvgl-api-reference.md` | Human-readable LVGL MicroPython API parsed from `lvgl_micropython/lvgl.pyi` | `python3 scripts/extract_lvgl_api.py` |
| `reference/lvgl_api_summary.json` | Machine-readable LVGL MicroPython API parsed from `lvgl_micropython/lvgl.pyi`, containing `generated_at`, `counts`, `symbols[]` | `python3 scripts/extract_lvgl_api.py` |

If reference files do not exist or are outdated, run the extraction scripts:

```bash
python3 /home/leeqingshui/MicroPython_Skills/mpos-dev/scripts/extract_mpos_api.py
python3 /home/leeqingshui/MicroPython_Skills/mpos-dev/scripts/extract_lvgl_api.py --lvgl-micropython-dir /home/leeqingshui/lvgl_micropython
```

### API Reference Usage Rules

- The MPOS API reference only represents interfaces that MicroPython users can import/call. Native modules are used only in their MPY call form (`adc_mic`, `pdm_mic`, `qrdecode`, `rvswd`, `webcam`); do not infer C functions from `c_mpos`.
- LVGL code generation relies primarily on `lvgl_api_summary.json`'s `symbols[]`, prioritizing symbols with `kind == "enum"`, `kind == "enum_member"`, `kind == "widget"`, or `kind == "function"`.
- `type_aliases[]` only explains signature types. `runtime_api: false` means `lv.<alias>` cannot be generated; when `runtime_enum` exists, generate the corresponding enum class member, e.g., `event_code_t -> lv.EVENT.CLICKED`, `display_render_mode_t -> lv.DISPLAY_RENDER_MODE.PARTIAL`, `grad_dir_t -> lv.GRAD_DIR.VER`, `fs_whence_t -> lv.FS_SEEK.SET`.
- Occurrences of `"display_render_mode_t"`, `"event_code_t"`, `"grad_dir_t"` in method signatures are type annotations, not runtime APIs. `lv.area_t()`, `lv.style_t()`, `lv.anim_t()` and similar `*_t` data classes/constructors are real MPY APIs; do not exclude them based solely on the suffix.
- When `description` is empty, do not fabricate semantics; if explanation is needed, read the docs reference, current repository code, or specific source context.

Read these docs/reference files on demand to avoid stuffing the entire documentation set into the context:

| Task | Read |
|------|------|
| Generate/modify App, requirements analysis, Activity/Service/Intent | `reference/docs-app-model.md` |
| Use system managers, persistence, downloads, background tasks, notifications, audio, sensors | `reference/docs-frameworks.md` |
| Package `.mpk`, validate manifest, generate app_index, prepare upystore/BadgeHub | `reference/docs-packaging.md` |
| Linux desktop simulation, install App to device, firmware flashing, target device selection | `reference/docs-deploy-targets.md` |
| Modify OS kernel, build system, test infrastructure, board porting, release process | `reference/docs-os-development.md` |
| Browser/WebAssembly runtime, `web.micropythonos.com`, web target | `reference/docs-web-port.md` |
| Audit whether all 61 docs pages are included in the reference routing | `reference/docs-site-index.md` |

These references already incorporate the local rules from `<repo-root>/AGENTS.md`; when official docs examples conflict with local repository tests, the current repository and AGENTS take precedence.

## App and MPK Basic Contract

When creating a new App, use the current repository's new flat structure:

```text
internal_filesystem/apps/<fullname>/
  MANIFEST.JSON
  icon_64x64.png
  assets/<entrypoint>.py
```

- The old `META-INF/MANIFEST.JSON` and `res/mipmap-mdpi/icon_64x64.png` are retained only for compatibility; newly generated Apps do not use the old layout.
- The App directory name must equal the manifest's `fullname`.
- `version` must be a canonical integer dot-separated string, e.g., `1.0.0`.
- The `entrypoint` of an activity/service must end with `.py`, the file must exist, and the source must contain the corresponding `classname`.
- Newly generated activity/service metadata uses full objects: `classname`, `entrypoint`, `intent_filters`; do not use the string-type `activities` from storefront seed data.
- `.mpk` is a ZIP file; the first local header must be the `<fullname>/` directory entry, and all files must reside under that single top-level directory. Read `reference/docs-packaging.md` for packaging details.

## LVGL Programming Conventions (Must Follow Each Rule)

The following rules come from AGENTS.md; **they must be followed one by one when writing LVGL UI code**:

### Imports and Globals
- Use `import lvgl as lv`, access all APIs via `lv.`
- Use `lv.screen_active()`, not `lv.scr_act()`
- Do not hardcode display resolution; use `lv.pct(100)` for responsiveness
- Use `button`, not `btn`; use `image`, not `img`
- `*_t = int` are type aliases in `lvgl.pyi`, not runtime enums; write code with `lv.EVENT.CLICKED`, `lv.COLOR_FORMAT.RGB565`, `lv.DISPLAY_RENDER_MODE.PARTIAL`, `lv.GRAD_DIR.VER`

### Events
- Use `lv.EVENT.VALUE_CHANGED`, not `lv.EVENT_VALUE_CHANGED`
- Event handlers need 3 parameters: `obj.add_event_cb(callback, lv.EVENT.CLICKED, None)`
- Methods used as event callbacks must accept the event parameter: `def callback(self, event)`
- Methods called both directly and as event callbacks need a default value: `def method(self, event=None)`
- Use `event.get_target_obj()`, not `event.get_current_target()`

### Flags and States
- Use `lv.obj.FLAG.CLICKABLE`, not `lv.OBJ_FLAG.CLICKABLE`
- Use `.add_flag(lv.obj.FLAG.HIDDEN)` / `.remove_flag(lv.obj.FLAG.HIDDEN)`, not `.set_hidden()`
- Use `.remove_flag()`, not `.clear_flag()`
- Use `obj.remove_state(...)`, not `obj.clear_state(...)`
- `lv.obj.FLAG.FLOATING` — removes widget from flex layout

### Styles
- Use `style_obj = lv.style_t()` then `style_obj.init()` — **must init before calling setters**, otherwise the device may crash
- LVGL 9.x: style setters only accept values; the selector is in `add_style()`: `obj.add_style(style, lv.PART.ITEMS | lv.STATE.CHECKED)`
- Colors: use `lv.palette_main(lv.PALETTE.RED)` or `lv.color_hex(0xEC048C)`
- The `lv.OPA` enum has only 10 steps: `TRANSP(0)`, `_10`, `_20`, ..., `_100`, `COVER(255)`. Values like `_5` do not exist

### Widget-Specific
- Label: newly created labels display "Text" by default; must explicitly call `label.set_text("")`
- Label: use `label.set_long_mode(lv.label.LONG_MODE.WRAP)`, not `lv.label.LONG.WRAP`
- Msgbox: use `msgbox = lv.msgbox()` then `msgbox.add_title("title")`
- Buttonmatrix: use `lv.buttonmatrix.CTRL.CHECKABLE` / `lv.buttonmatrix.CTRL.CHECKED`
- Buttonmatrix: `set_map()` triggers `LV_EVENT_VALUE_CHANGED` asynchronously; use time debouncing with `time.ticks_diff(now, last_ts) < 50`
- Buttonmatrix: there is no `set_button_text()` / `set_button_ctrl()`; to update text, rebuild the map
- Dropdown: use `lv.dropdown(lst, lv.DROPDOWN.DIR.BOTTOM)` (uppercase DIR)
- Anim: use `lv.anim_t.path_ease_in_out`, not `lv.anim_path_ease_in_out`
- No `get_child_by_type()`; use global variables to store child object references
- LVGL objects do not support arbitrary Python attribute assignment (`btn.idx = 5` raises an error); use closures/lambdas or parallel lists

### Keyboard Input (SDL/Desktop)
- The SDL keyboard driver generates an instantaneous press+release pair for each key
- SDL_KEYUP is completely ignored
- Detecting key release: use a timeout mechanism — set a long deadline (~600ms) on first press, and a short delay (~100ms) for repeat events

### Images and Snapshots
- `lv.snapshot_take()` may still capture non-transparent pixels (theme style leakage) on hidden objects
- To scale an image snapshot: place the image in a container, set the container size, and snapshot the container
- Manually construct an empty image with `lv.image_dsc_t()`:
  ```python
  buf = bytearray(4)
  dsc = lv.image_dsc_t()
  dsc.data = buf
  dsc.header.w = 1
  dsc.header.h = target_height
  dsc.header.cf = lv.COLOR_FORMAT.ARGB8888
  ```

## Native MicroPython Module Quick Reference

### webcam Module
```python
import webcam

cam = webcam.init(width=320, height=240)          # or webcam.init("/dev/video1", width=640, height=480)
frame = webcam.capture_frame(cam, "grayscale")    # -> memoryview (1 byte/pixel)
frame = webcam.capture_frame(cam, "rgb565")       # -> memoryview (2 bytes/pixel)
webcam.reconfigure(cam, width=640, height=480)    # Switch resolution at runtime
webcam.free_buffer(cam)                           # Free internal buffer
webcam.deinit(cam)                                # Close device
```

### pdm_mic Module
```python
from pdm_mic import PDM_Mic

mic = PDM_Mic(clk=42, data=41, rate=16000, bufsize=4096)
mic.start()
buf = bytearray(1024)
mic.readinto(buf)
mic.stop()
mic.deinit()
```

### adc_mic Module
```python
from adc_mic import read
buf = read(chunk_samples=512, unit_id=1, adc_channel_list=[0,1],
           adc_channel_num=2, sample_rate_hz=16000, atten=3)
```

### qrdecode Module
```python
from qrdecode import qrdecode, qrdecode_rgb565
result = qrdecode(grayscale_buffer, width, height)         # Decode grayscale image
result = qrdecode_rgb565(rgb565_buffer, width, height)     # Decode RGB565 image
# Returns decoded payload bytes; raises an exception if not recognized
```

### rvswd Module
```python
from rvswd import RVSWD

prog = RVSWD(39, 42)                          # RVSWD(swdio, swclk)
prog.reset()
prog.halt()
prog.resume()
vendor = prog.read_vendor_bytes()
prog.read_reg(reg) / prog.write_reg(reg, value)
prog.read_memory(addr) / prog.write_memory(addr, value)
# CH32X03x / CH32V20x series
prog.x03x_program(firmware, progress_callback)  # callback(msg, pct)
prog.v20x_program(firmware, progress_callback)
```

## Strong Constraints

- **Consult API reference files first**: use `reference/mpos-api-reference.md` / `reference/lvgl-api-reference.md` for human reading, and `reference/mpos_api_summary.json` / `reference/lvgl_api_summary.json` for machine retrieval; run extraction scripts if information is insufficient or outdated
- **All LVGL code must follow the LVGL programming conventions above**, checked line by line
- **Activity.__init__ must call super().__init__()**
- **New labels must explicitly call set_text("")**
- **After style_t(), must call init() before setters**
- **Do not hardcode screen resolution**; use `lv.pct(100)`
- **Do not bypass framework APIs**: use SharedPreferences for persistence, do not manipulate JSON files directly
- **Do not modify AGENTS.md or ruff.toml**
- **Place temporary files in tmp/, not /tmp**
- **Kill processes with killall, not pkill -f**
- **Follow the code formatting in ruff.toml** (double quotes)
