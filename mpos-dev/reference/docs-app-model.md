# MicroPythonOS App Model Reference

This file is generated based on the `docs.micropythonos.com` sitemap/search index re-read on 2026-07-14, and corrected with `/home/leeqingshui/MicroPythonOS/AGENTS.md` and current local repository facts.

## When to Read

Read this file when generating, modifying, reviewing, or analyzing MicroPythonOS Apps. Read `docs-packaging.md` for packaging rules. Read `docs-frameworks.md` for framework API.

## Source Coverage

- `apps/`
- `apps/creating-apps/`
- `apps/app-lifecycle/`
- `apps/appstore/`
- `apps/built-in-apps/`
- `apps/native-apps/`
- `architecture/intents/`

## Local-First Rule: App Directory Structure

The current local repository and `tests/test_apps_manifest.py` use a flat structure as default:

```text
internal_filesystem/apps/<fullname>/
  MANIFEST.JSON
  icon_64x64.png
  assets/<entrypoint>.py
```

The old nested structure is still compatible with the current App loader and installation tests, but should be considered legacy and avoided when generating new Apps:

```text
internal_filesystem/apps/<fullname>/
  META-INF/MANIFEST.JSON
  assets/<entrypoint>.py
  res/mipmap-mdpi/icon_64x64.png
```

`mpos-gen-app` generates root-level `MANIFEST.JSON` and root-level `icon_64x64.png` by default. The entrypoint is placed at `assets/main.py` by default, so that runtime files placed in `assets/` by `mpos-prepare-deps` can be imported directly.

## App Identity and Manifest

Each App is located at `internal_filesystem/apps/<fullname>/`, and `<fullname>` must match the `fullname` field in `MANIFEST.JSON`. For compatibility with legacy Apps, `META-INF/MANIFEST.JSON` can be read, but newly generated Apps should not use the old path.

Manifest fields required by local tests:

- `fullname`: Must match the directory name.
- `name`: Display name.
- `version`: Standard dotted integer version, e.g., `1.0.0`. Do not use `01.0` or `1.0-beta`.
- `activities`: Optional list, but each entry must have an existing `.py` entrypoint, and the source code must contain the corresponding classname.
- `services`: Optional list, with the same entrypoint/classname validation rules.

Activity/Service metadata uses full objects:

```json
{
  "classname": "ExampleActivity",
  "entrypoint": "assets/main.py",
  "intent_filters": [
    {"action": "main", "category": "launcher"}
  ]
}
```

Do not use the string-type `activities` structure found in some storefront seed data.

## Activity Model

An Activity is a single UI screen managed by the activity stack. Lifecycle methods:

- `onCreate()`: Creation state, build or prepare UI.
- `onStart(screen)`: Screen is about to become visible.
- `onResume(screen)`: Enters foreground, interactive.
- `onPause(screen)`: Another activity is about to come to the foreground.
- `onStop(screen)`: No longer visible.
- `onDestroy(screen)`: Clean up resources before removal from the stack.

Minimal pattern:

```python
import lvgl as lv
from mpos import Activity


class ExampleActivity(Activity):
    def __init__(self):
        super().__init__()

    def onCreate(self):
        screen = lv.obj()
        label = lv.label(screen)
        label.set_text("Hello")
        label.center()
        self.setContentView(screen)
```

`self.appFullName` is set by `ActivityNavigator`. Used for App-level preferences, e.g., `SharedPreferences(self.appFullName)`.

## Service Model

A Service is a background component without UI. Suitable for startup tasks and long-running tasks, such as automatic WiFi connection, web server startup, async REPL tasks, periodic checks, notifications, etc.

Lifecycle:

- `onCreate()`: Initialize resources.
- `onStart(intent=None)`: Execute or schedule work.
- `onDestroy()`: Clean up resources.

A Service can subscribe to `boot_completed` in the manifest:

```json
{
  "classname": "ExampleBootService",
  "entrypoint": "assets/service.py",
  "intent_filters": [{"action": "boot_completed"}]
}
```

## Intent and Navigation

Use `Intent` for decoupled Activity communication.

- Explicit intents target a known activity.
- Implicit intents specify an action/category, resolved by the system to a handler.
- Use intents to pass data, receive results, and let Apps handle file/action routing, avoiding direct dependencies on other Apps.

New code should import from the main `mpos` module first:

```python
from mpos import Activity, Intent
```

Existing local code may import from submodules; do not unnecessarily modify unrelated files just to unify imports.

## Built-in Apps and AppStore Context

Built-in Apps are located at `/builtin/apps/` on the device, including launcher, WiFi, AppStore, OSUpdate, Settings, File Manager. User-installed Apps are at `/apps/`.

The AppStore installs `.mpk` packages to `/apps/` and supports multiple backends. Publishing and MPK validation are separate steps; see `docs-packaging.md`.

## Native Modules

Most Apps should be written in MicroPython. Use C/C++ native modules only when pure MicroPython is insufficient, such as for high-frequency game loops, signal processing, calling C libraries, etc.

Native modules increase build complexity and are architecture-dependent. If functionality requires a C module, first enter the dependency preparation and deployment/build phase before committing to device support.

## Local App Code Rules from AGENTS

- Prefer root-level `Makefile` targets when equivalent entry points exist.
- Must pass `make lint` after every code modification.
- Do not modify `AGENTS.md` or `ruff.toml`.
- Use double quotes as per `ruff.toml`.
- Place temporary debug files in the project's `tmp/` directory, not `/tmp`.
- Do not hardcode screen resolution; use `lv.pct(100)`, flex, or align.
- New labels must explicitly call `set_text("")` or set the final text.
- After `lv.style_t()`, must call `init()` before calling setters.
- Activity's `__init__` must call `super().__init__()`.
- Use `machine.reset()` for device reset; soft reset on the current stack has issues.
