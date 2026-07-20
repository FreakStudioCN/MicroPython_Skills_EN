# MicroPythonOS Web Port Reference

This file is generated based on a re-read of `docs.micropythonos.com/web-port/*`, `https://web.micropythonos.com/`, and `/home/leeqingshui/MicroPythonOS/AGENTS.md` on 2026-07-14.

## When to Read

Read this file when working with browser/WebAssembly previews, web-port smoke tests, explaining browser limitations, or modifying the `web` build target. For normal desktop emulation, read `docs-deploy-targets.md`.

## Source Coverage

- `web-port/using/`
- `web-port/developer/`
- `https://web.micropythonos.com/`
- Local `AGENTS.md`

## What is the Web Port

The Web port runs MicroPythonOS fully in a modern browser via WebAssembly. It is suitable for quickly trying out the OS without flashing hardware, and for rapid visual smoke checks.

It is not:

- A firmware installer.
- An app store publishing entry point.
- A replacement for Linux SDL emulation when debugging local source changes.
- A substitute for physical device verification when hardware behavior is involved.

## Live Page Facts

`https://web.micropythonos.com/` returned `200` on 2026-07-14.

Page behavior:

- Title: `MicroPythonOS Web`.
- Loads `micropython.js`.
- Uses a `320x240` LVGL canvas.
- Starts MicroPythonOS via `["-X", "heapsize=16M", "-m", "main"]`.
- Provides a toggleable Log panel.
- Provides Reset storage.
- Simulates NeoPixels, joystick, MENU, START, X/Y/A/B.
- Uses `Module.__webio` to simulate badge peripherals.
- Mounts `/data` and `/apps` using IndexedDB/IDBFS.
- Seeds bundled apps into `/apps` on first run.

## Persistence

The browser stores writable `/data` and `/apps` in IndexedDB. This means preferences and installed apps persist across page refreshes.

To force clear state:

- Use the Reset storage button on the page if available.
- Or clear site data/IndexedDB in the browser developer tools.

After reset, bundled demo apps are re-seeded on first run.

## Developer Build Process

The process described in the docs:

```bash
scripts/build_mpos.sh web
scripts/run_web.sh
scripts/run_web.sh --no-build
PORT=9000 scripts/run_web.sh
```

Output is located in `web/`, including `micropython.html`, `micropython.js`, `micropython.wasm`, `micropython.data`, and copied web entry files.

Building requires the Emscripten SDK. If the environment is configured as per the docs, the build can automatically activate a nearby `emsdk` checkout.

If the Web build/link reports low-level symbol errors like `machine_timer_type`, first classify them as Web port, MicroPythonOS, or toolchain issues. The resolution order is to confirm Emscripten/emsdk, submodules, and the current OS Web target state; do not write it off as a missing Python dependency of the target app, and do not modify OS/build source code during normal app generation/fix phases.

## Integration Notes

The Web port is self-contained within the main MicroPythonOS repository. Submodule modifications required by the web target are stored in `scripts/web_port/` and applied automatically during the build. Unless the task explicitly requires modifying these projects, do not leave persistent changes in nested `lvgl_micropython`, `micropython`, or `lvgl` directories.

## Testing Recommendations

Web port checks are suitable for:

- Quick user previews.
- Browser-specific input/storage regression testing.
- Confirming WebAssembly boot and bundled app seeding.
- Checking `/data` and `/apps` persistence behavior.

Local automated app debugging should still use Linux SDL desktop emulation and `mpos_controller.py`. Hardware-specific behavior should be verified with physical devices.

## Local Rules from AGENTS

- For desktop work, prefer `make build-mpos-unix`; only use the web build when the target is browser/WebAssembly.
- Place temporary scripts and debug artifacts in the repository `tmp/` directory.
- Every code modification must pass `make lint`.
- Do not modify `AGENTS.md` or `ruff.toml`.
- When cleaning up local desktop processes, use `killall`, not `pkill -f`.
