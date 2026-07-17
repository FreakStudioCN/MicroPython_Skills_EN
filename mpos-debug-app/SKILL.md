---
name: mpos-debug-app
description: MicroPythonOS App debugger. Helps troubleshoot runtime issues (crashes, UI anomalies, logic errors). Master desktop simulator, mpos_controller automation tool, print diagnostic strategies, and common LVGL pitfalls. Triggered when user needs to debug an App, fix bugs, or analyze runtime behavior.
---

# MicroPythonOS App Debugger

## Role

You are a MicroPythonOS App debugging expert. You use the desktop simulator, mpos_controller, and diagnostic prints to quickly locate and fix runtime issues.

## Unified Project Log

When debugging a single App, also record key diagnostic events in the project state directory for later recovery and review:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-plan-app/scripts/update_plan_state.py record \
  --repo <repo-root> \
  --fullname <fullname> \
  --skill mpos-debug-app \
  --phase debug \
  --result <partial|failed|blocked|success> \
  --artifact app_test_result=<related_result_or_log.json> \
  --next-skill <next-skill-or-null> \
  --event "<short diagnostic summary>"
```

Do not stuff long log content into `activity_log.jsonl`; save long output under `tmp/` and only record the path and summary in the event.

**Prerequisites**: This skill depends on the LVGL programming conventions (root cause of most bugs), C module API, and code architecture provided by `mpos-dev`. Before debugging, ensure you understand from mpos-dev:
- LVGL programming conventions (check each one — most UI bugs are caused by violations)
- C module API usage (especially correct invocation of webcam/pdm_mic)
- Global hard constraints

## Desktop Simulator

Run MicroPythonOS on the desktop for rapid iteration without flashing the device:

```bash
# Run desktop simulator (30-second timeout protection)
timeout -s 9 30 ./scripts/run_desktop.sh

# Manual run (no timeout, exit with Ctrl+C)
./scripts/run_desktop.sh
```

The simulator uses the `lvgl_micropy_unix` binary with 16MB heap and SDL rendering. Behavior closely matches the device.

### Process Management

```bash
# Kill residual processes (when desktop simulator did not exit cleanly)
killall lvgl_micropy_unix run_desktop.sh
```

## mpos_controller.py (Automation Control)

`scripts/mpos_controller.py` (32KB) provides PTY/aioREPL and serial backends for:
- Sending keyboard/touch events to a running simulator
- Taking screenshots (visual regression testing)
- Executing REPL commands

```bash
# PTY mode (desktop simulator)
python3 scripts/mpos_controller.py --backend pty

# Serial mode (physical device)
python3 scripts/mpos_controller.py --backend serial --port /dev/ttyUSB0
```

## Diagnostic Strategy

### print() Diagnostics

MicroPythonOS `print()` output goes to simulator stdout or serial port. When debugging:
1. Add `print(">>> reached X, value =", val)` at suspicious code paths
2. Run the desktop simulator and observe output
3. Remove debug prints after locating the issue

### Temporary Files

Write temporary data to the `tmp/` directory (not `/tmp`):

```python
with open("tmp/debug_log.txt", "w") as f:
    f.write(str(diagnostic_data))
```

## Common LVGL Bug Checklist

Check each item against mpos-dev's LVGL conventions:

| Symptom | Common Cause | Check |
|---------|-------------|-------|
| Device freeze/crash | Forgot to call `init()` after `style_t()` | Search for `lv.style_t()` and check if `.init()` follows on the next line |
| Label shows extra "Text" | New label default text | Immediately call `label.set_text("")` after creation |
| Event callback not firing | Wrong event name | Verify it's `lv.EVENT.VALUE_CHANGED`, not `lv.EVENT_VALUE_CHANGED` |
| Flag operations ineffective | Wrong method name | Use `.add_flag()` / `.remove_flag()`, not `.set_hidden()` / `.clear_flag()` |
| Buttonmatrix value read anomaly | `set_map()` triggers event asynchronously | Add time debounce: `time.ticks_diff(now, last_ts) < 50` |
| Buttonmatrix text cannot be changed | No `set_button_text()` method | Must rebuild the map |
| Animation not working | Wrong API name | Use `lv.anim_t.path_ease_in_out`, not `lv.anim_path_ease_in_out` |
| Attribute assignment error | LVGL objects don't support Python attributes | Use closures/lambdas or parallel lists instead of `obj.myattr = x` |
| Snapshot of hidden object has ghosting | Theme style leak | Place image in a container, snapshot the container instead of the image directly |
| SDL key "hold" not working | SDL_KEYUP ignored | Use timeout mechanism to simulate long-press detection |
| OPA transparency value invalid | Used non-existent enum value | `lv.OPA` only has TRANSP/_10/_20/.../_100/COVER |

## Build System

After fixing code, if you need to recompile C modules or regenerate bindings:

```bash
# Desktop build
make build-mpos-unix

# Equivalent to
./scripts/build_mpos.sh unix
```

If only Python files (`.py` under `internal_filesystem/`) were changed, no rebuild is needed — just run the simulator.

## Device Debugging

```bash
# Install App to device
./scripts/install.sh com.micropythonos.<appname>

# Refresh App registry after installation
# Execute in device REPL:
import mpos.content.app_manager as am
am.AppManager().refresh_apps()

# Deploy a single updated file
python3 lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py \
  cp internal_filesystem/lib/mpos/ui/testing.py :/lib/mpos/ui/testing.py
```

## Hard Constraints

- **Must use killall to kill processes, not pkill -f**
- **Temporary files go in tmp/, not /tmp**
- **Desktop runs must have timeout protection**: `timeout -s 9 30 ./scripts/run_desktop.sh`
- **Do not modify AGENTS.md or ruff.toml**
- **For LVGL bugs, first check against mpos-dev's conventions**
