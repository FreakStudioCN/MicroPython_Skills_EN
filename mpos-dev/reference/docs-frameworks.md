# MicroPythonOS Framework Reference

This file is generated based on the `docs.micropythonos.com` sitemap/search index re-read on 2026-07-14, and corrected with `/home/leeqingshui/MicroPythonOS/AGENTS.md`.

## When to Read

Read this file before generating code that uses system services, persistence, networking, audio, sensors, camera, background tasks, App launch/install flow, UI/system settings.

## Source Coverage

- `architecture/frameworks/`
- `frameworks/app-manager/`
- `frameworks/appearance-manager/`
- `frameworks/audiomanager/`
- `frameworks/battery-manager/`
- `frameworks/build-info/`
- `frameworks/connectivity-manager/`
- `frameworks/device-info/`
- `frameworks/display-metrics/`
- `frameworks/download-manager/`
- `frameworks/file-explorer-activity/`
- `frameworks/focus/`
- `frameworks/font-manager/`
- `frameworks/input-activity/`
- `frameworks/input-manager/`
- `frameworks/lights-manager/`
- `frameworks/notification-manager/`
- `frameworks/number-format/`
- `frameworks/preferences/`
- `frameworks/sensor-manager/`
- `frameworks/service/`
- `frameworks/setting-activity/`
- `frameworks/settings-activity/`
- `frameworks/task-manager/`
- `frameworks/time-zone/`
- `frameworks/webserver/`
- `frameworks/widget-animator/`
- `frameworks/wifi-service/`

## Import Strategy

The docs recommend importing frameworks from the root `mpos` module:

```python
from mpos import AppManager, DownloadManager, TaskManager
```

Or:

```python
import mpos
mpos.AppManager.get_app_list()
```

Avoid adding sub-module imports in new code, unless the file currently being edited already has such a local convention, or a symbol is not re-exported.

## Core Framework

### AppManager

Used for app discovery, app registry, `.mpk` installation, uninstalling user apps, launching apps, version management, intent resolution, restarting the launcher, registering/starting services.

Common usage:

- `AppManager.get_app_list()`
- `AppManager.get("<fullname>")`
- `AppManager.start_app("<fullname>")`
- `AppManager.install_mpk(temp_zip_path, dest_folder)`
- `AppManager.refresh_apps()`

After installing an app to a physical device using a local script, first call `AppManager.refresh_apps()`, then expect `start_app()` to find it.

### DownloadManager

Used for HTTP downloads. Supports downloading to memory, file, stream callback; supports retry, progress callback, range/resume, chunked download.

Apps/network features should use it preferentially, rather than writing temporary socket download logic manually.

### TaskManager

Used for async/background work in apps:

- `TaskManager.create_task(coro)`
- `TaskManager.sleep(seconds)`
- `TaskManager.sleep_ms(milliseconds)`
- `TaskManager.wait_for(coro, timeout=...)`
- event notification helpers

When background tasks affect widgets, use the foreground-safe UI update path provided by Activity.

### SharedPreferences

Used for app-level persistence. In an Activity, prefer `SharedPreferences(self.appFullName)`, do not hardcode the package name.

Do not write custom JSON configuration files directly for normal app preferences, unless there is a clear reason.

### Service

Service is used for long-running tasks without UI, or startup tasks. Service has `onCreate`, `onStart`, `onDestroy`, and can subscribe to `boot_completed`.

## Hardware and System Managers

- `AudioManager`: Playback and recording; coordinates audio priority and hardware output.
- `BatteryManager`: Battery/voltage status.
- `CameraManager`: Camera access; check if the C module is available for camera features.
- `ConnectivityManager`: Network-aware app behavior and reconnection flow.
- `InputManager`, `InputActivity`, focus helpers: Keyboard, touch, buttons, and focus navigation.
- `LightsManager`: LED/NeoPixel-like device lighting.
- `SensorManager`: Sensor access and readings.
- `NotificationManager`: Notifications and status UI.
- `AppearanceManager`, `DisplayMetrics`, `FontManager`, `WidgetAnimator`: UI dimensions, themes, fonts, and animations.
- `WebServer`, `WifiService`: Network services.
- `BuildInfo`, `DeviceInfo`, `TimeZone`, `NumberFormat`: System metadata and utilities.

## UI/LVGL Rules from AGENTS

- `import lvgl as lv`; use API via `lv.`.
- Use `lv.screen_active()`, not `lv.scr_act()`.
- Use names like `button`, `image`, `lv.EVENT.VALUE_CHANGED`, `lv.obj.FLAG.*`, `lv.buttonmatrix.CTRL.*`.
- Event callbacks need an event parameter, and register with `obj.add_event_cb(callback, lv.EVENT.CLICKED, None)`.
- Use `event.get_target_obj()`, not `event.get_current_target()`.
- Do not hardcode screen resolution.
- New labels must not retain the default `"Text"`.
- Before a setter, always `style = lv.style_t(); style.init()`.
- LVGL object wrappers do not accept arbitrary Python attributes; use closures or parallel state structures.
- `lv.buttonmatrix.set_map()` may trigger value-changed asynchronously; debounce by time.
- SDL keyboard has no key-up event; model long press with timeout.

## Compatibility Notes from AGENTS

- The current stack has issues with soft reset; use `machine.reset()`.
- Some builds lack `random.Random` and `random.shuffle`; implement Fisher-Yates or a small local LCG when needed.
- Avoid writing `except Exception: pass` in rendering paths; it hides real errors.

## Test Entry Points

For UI verification, prefer:

- `mpos.ui.testing.GraphicalTestCase`
- `KeyboardTestCase`
- `scripts/mpos_controller.py`
- widget tree and visible text extraction
- screenshots with pixel check for visual regression

When `internal_filesystem/lib/mpos/ui/testing.py` already exists and can be extended, do not write temporary helpers in tests.
