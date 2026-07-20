# MicroPythonOS Deployment Target Reference

This file is generated based on a re-read of `docs.micropythonos.com`, `https://install.micropythonos.com/`, `https://web.micropythonos.com/`, and `/home/leeqingshui/MicroPythonOS/AGENTS.md` on 2026-07-14.

## When to Read

Read this file when handling desktop emulation, browser/WebAssembly preview, installing apps to devices, firmware flashing, or target selection. For OS build internals, read `docs-os-development.md`.

## Source Coverage

- `getting-started/running/`
- `getting-started/supported-hardware/`
- `os-development/running-on-desktop/`
- `os-development/installing-on-esp32/`
- `os-development/linux/`
- `os-development/macos/`
- `os-development/windows/`
- `os-development/emulating-esp32-on-desktop/`
- `https://install.micropythonos.com/`
- `https://web.micropythonos.com/`
- Local `AGENTS.md`

## Target Types

MicroPythonOS can run on:

- ESP32 and ESP32-S3 devices.
- Linux/macOS desktops via SDL.
- Raspberry Pi and Linux-like environments such as WSL2.
- Browser/WebAssembly.
- QEMU ESP32 emulator for deeper OS development and CI-style testing.

## Local Desktop Emulation

Local AGENTS rules take precedence:

```bash
make build-mpos-unix
timeout -s 9 30 ./scripts/run_desktop.sh
```

For automated debugging, use `scripts/mpos_controller.py`. `MPOSController()` does not automatically start the process; you need to call `mpos.start()`, wait approximately 8 to 10 seconds, then call `startapp()` or perform REPL operations.

Use `killall` to clean up residual simulator processes, not `pkill -f`:

```bash
killall lvgl_micropy_unix run_desktop.sh
```

Write controller/debug scripts under the repository `tmp/` directory.

## Browser/WebAssembly Preview

`https://web.micropythonos.com/` is a browser runtime, not an installer or app publishing site.

Observed page behavior:

- Loads `micropython.js`.
- Runs with `["-X", "heapsize=16M", "-m", "main"]`.
- Displays a `320x240` LVGL canvas.
- Provides Log and Reset storage controls.
- Simulates NeoPixels, joystick, MENU, START, X/Y/A/B.
- Mounts `/data` and `/apps` via IndexedDB/IDBFS, allowing preferences and user apps to persist across refreshes.

It is suitable for quick user previews and Web port smoke checks. Do not use it as a substitute for Linux SDL emulation or physical device verification when hardware behavior is relevant.

## Installing Apps to Physical Devices

Installing an app is not the same as flashing firmware. For normal Python app iteration, use:

```bash
./scripts/install.sh com.micropythonos.appname
```

Before starting a real-device app installation, confirm that the device has MicroPythonOS installed, the target board model, and the serial port. If the device does not have the OS installed or its status is unknown, first use `https://install.micropythonos.com/` to install/confirm the firmware.

After installation:

```python
from mpos import AppManager
AppManager.refresh_apps()
```

A reboot/reset of the device may also be necessary.

To deploy a single file:

```bash
python3 lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py cp local.py :/remote.py
```

If `mpos_controller.py` / AIOREPL probe fails, but the serial filesystem is accessible, direct app directory copying can be used as a `device-copy` record:

```bash
python3 lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py connect /dev/ttyACM0 fs mkdir :/apps
python3 lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py connect /dev/ttyACM0 fs cp -r internal_filesystem/apps/<fullname> :/apps/
python3 lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py connect /dev/ttyACM0 fs ls :/apps/<fullname>
```

This only proves the file has been copied to the device; for release verification, the MPK install path that calls `AppManager.install_mpk()` should still be preferred.

Then use `machine.reset()` and wait for startup.

## Firmware Installation and Flashing

Only use firmware flashing in the following situations:

- The user explicitly requests firmware flashing.
- The firmware is missing or the version is incorrect.
- Changes have been made to C modules, LVGL bindings, board support, filesystem images, or OS internals.

Current web installer facts:

- `install.micropythonos.com` provides a WebSerial installer.
- Requires USB and a WebSerial-compatible browser, such as Chrome or Edge.
- The page uses `esp-web-install-button`, offering ESP32 and ESP32-S3 targets.
- Currently lists 12 ESP32/ESP32-S3 manifests for `0.10.x`, `0.11.x`, `0.12.x`, `0.13.x`, `0.14.x`, `0.15.x`.
- The latest `0.15.x` manifest corresponds to `0.15.1`, with firmware paths `/firmware_images/esp32/MicroPythonOS_esp32_0.15.1.bin` and `/firmware_images/esp32s3/MicroPythonOS_esp32s3_0.15.1.bin`.
- In the installer manifests read, `new_install_prompt_erase` is `true` for all.

Local flashing path:

```bash
./scripts/build_mpos.sh <target>
./scripts/flash_over_usb.sh
```

Do not perform destructive erase/flash actions without explicit user confirmation.

## QEMU ESP32 Emulation

The docs describe an ESP32 QEMU path for deeper OS testing, capable of emulating WiFi, storage, ULP/deepsleep, GPIO/touch buttons, and ST7789V display. Treat this as OS-development infrastructure, not the default app development path.

## Supported Hardware Description

The docs list multiple ESP32/ESP32-S3 devices as well as browser/desktop targets. When an app depends on sensor, button, camera, display, LED, or radio hardware, enter the dependency preparation phase; ask the user if the target device is unknown.

## Security Rules from AGENTS

- When equivalent entry points exist, prefer `make build-mpos-unix`, `make syntax-tests`, `make unittest-tests`, `make tests`, `make lint`, `make lint-fix`.
- Every code modification must pass `make lint`.
- Use `timeout -s 9 30 ./scripts/run_desktop.sh`.
- Use `killall`, not `pkill -f`.
- Place temporary files in the project `tmp/` directory.
- Do not confuse app installation, MPK installation, and firmware flashing.
