# MicroPythonOS OS Development Reference

This file is generated based on a re-read of `docs.micropythonos.com` and `/home/leeqingshui/MicroPythonOS/AGENTS.md` as of 2026-07-14.

## When to Read

Read this file when modifying MicroPythonOS internals, C modules, build scripts, board support, test infrastructure, release processes, file format support, or low-level LVGL/MicroPython integration. Normal App generation should use `docs-app-model.md` and `docs-frameworks.md`.

## Source Coverage

- `os-development/`
- `os-development/compiling/`
- `os-development/automated-testing/`
- `os-development/porting-guide/`
- `os-development/linux/`
- `os-development/macos/`
- `os-development/windows/`
- `other/merge-checklist/`
- `other/release-checklist/`
- `other/supported-file-formats/`
- Local `AGENTS.md`

## Repository Structure

The main code is located in `internal_filesystem/`, which corresponds to the filesystem layout on the target device. The OS build is based on `lvgl_micropython`, which contains LVGL, MicroPython, platform ports, and display/input drivers. MicroPython C bindings are located in `c_mpos/`.

## Build Entry Points

When equivalent entry points exist, prefer using the root directory `Makefile`:

```bash
make build-mpos-unix
make syntax-tests
make unittest-tests
make tests
make lint
make lint-fix
```

Low-level build script:

```bash
./scripts/build_mpos.sh <target>
```

Targets mentioned in the docs include `esp32`, `esp32s3`, `unix`, `macOS`, `web`. The local AGENTS also mentions additional targets such as `esp32-small`, `unphone`, `lilygo_t4`.

Important local note: `build_mpos.sh` modifies tracked files when patching build inputs. These changes persist unless explicitly reverted.

## Testing

Local AGENTS rules:

- Syntax tests are run via `./tests/syntax.sh`.
- Unit tests are run via `./tests/unittest.sh [test_file] [--ondevice]`.
- `make tests` runs both syntax and unit tests.
- `mpy-cross` is located at `./lvgl_micropython/lib/micropython/mpy-cross/build/mpy-cross`.
- Tests with `graphical` in the filename are identified as graphical tests.
- Non-graphical tests do not inject LVGL boot/main.
- On-device tests should not manually re-run boot/main; the runner handles this.
- Read `tests/README.md` before adding new tests.

Graphics/UI modifications:

- Use `mpos.ui.testing.GraphicalTestCase`.
- Extend local test helpers; do not write temporary ad hoc helpers.
- Verify using widget tree, visible text, screenshots, and pixel checks.

## Porting

Device-specific Python code should be placed in `internal_filesystem/lib/mpos/board/<boardname>.py`.

Porting workflow:

1. Build or configure `lvgl_micropython` for the device.
2. Confirm that the MicroPython REPL can start.
3. Add or adjust board-specific MicroPythonOS code.
4. Verify display, input, storage, reset, WiFi, and available managers.
5. Add tests or manual verification records.

Do not apply assumptions from the official LVGL binding to this repository's `lvgl_micropython` fork without checking local files.

## Release and Merge Checks

Before merging or releasing, check:

- Does `CHANGELOG.md` need updating?
- Does the modified App need a version bump in `MANIFEST.JSON`?
- Do the docs need updating?
- Does a new board require an update to `MAINTAINERS.md`?
- Have tests been added or extended for behavioral changes?
- Are functional changes separated from bulk formatting/generated diffs?

The release process includes updating the OS version, changelog JSON, GitHub builds, OTA artifacts, installer firmware files, and metadata.

## Supported File Formats

The docs list:

- Images: BMP, PNG, baseline JPEG, some RAW naming patterns.
- Progressive JPEG is not supported.
- Audio: PCM WAV and IMA ADPCM WAV.

Prefer these formats when generating Apps, unless a new decoder is explicitly required.

## Development Rules from AGENTS

- `make lint` must pass after every code change.
- Do not modify `AGENTS.md` or `ruff.toml`.
- Place temporary files in the project's `tmp/` directory.
- Use `killall`, not `pkill -f`.
- Use hard reset (`machine.reset()`), not soft reset.
- Desktop run commands must wrap `./scripts/run_desktop.sh` with `timeout -s 9 30`.
- Use double quotes according to the ruff config.
- Avoid silent `except Exception: pass`, especially in rendering paths.

## When to Upgrade from App Work to OS Work

Unless the task requires the following, remain at the App-level skill:

- C modules or native modules.
- LVGL binding modifications.
- New board/display/touch support.
- Firmware image modifications.
- Filesystem image modifications.
- WebAssembly runtime modifications.
- AppStore backend implementation modifications.

Otherwise, normal App generation, MPK packaging, desktop simulation, and device App installation are sufficient.
