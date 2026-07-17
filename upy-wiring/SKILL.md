---
name: upy-wiring
description: Wiring diagram generation. Reads all .py source files in firmware/ to extract actual pins/addresses/buses, cross-validates with project-manifest.json, then LLM generates an intermediate JSON, and scripts render a Mermaid wiring diagram (.md code block, CLI natively readable) + SVG + PNG + HTML (double-click in browser to view). Triggered after upy-scaffold or upy-generate completes.
---

# Wiring Diagram Generation Skill

## Role

**Read all `.py` source files in `firmware/` + `project-manifest.json`**, using firmware as the authoritative data source and manifest as the design reference. After cross-validation, the LLM fills in the intermediate JSON, and scripts validate and render the Mermaid wiring diagram + SVG + PNG + HTML + pin cross-reference table. **The LLM is responsible for understanding the data and filling in the JSON; scripts only perform validation and rendering.**

---

## Pre-checks

```bash
python --version
python -c "import jsonschema; print('jsonschema OK')"
```

If missing, prompt to install: `pip install jsonschema`

SVG rendering requires network (mermaid.ink API, zero local dependencies).

---

## Execution Steps

### Step 1: LLM reads Schema → understands structure

Read the intermediate JSON schema:

```
G:/MicroPython_Skills/upy-project-gen-toolchain-spec/wiring.schema.json
```

Understand the 6 required fields: `meta`, `mcu`, `buses`, `standalone`, `power`, `alerts`, and the optional field `canvas`.

### Step 2: LLM reads all .py source files in firmware/ → extracts hardware facts

**This is the core data source for this skill.** Read all `.py` files under `{project_dir}/firmware/` and extract hardware connection facts according to the following priority:

#### 2A: main.py — Hardware initialization (highest priority)

Search for patterns and extract:

| Search Pattern | Extracted Content | Usage |
|----------|----------|------|
| `I2C(id, scl=Pin(n), sda=Pin(n), freq=f)` | Bus ID, SCL/SDA GPIO numbers, frequency | buses[].signals, frequency_hz |
| `SPI(id, sck=Pin(n), mosi=Pin(n), miso=Pin(n))` | SPI bus signal lines | buses[].signals |
| `UART(id, tx=Pin(n), rx=Pin(n), baudrate=b)` | UART bus signal lines | buses[].signals |
| `Pin(n, Pin.OUT)` / `Pin(n, Pin.IN)` | GPIO number, direction | standalone[].type, mcu.pins[].type |
| `Pin(n, Pin.IN, Pin.PULL_UP)` | GPIO number, pull-up | standalone[].type=gpio_in_pullup |
| `create_*(i2c, ...)` / `create_*(pin, ...)` | Device factory call, actual address parameter | buses[].devices / standalone[] |
| Address constants / logs near `i2c.scan()` | Actual I2C address used | buses[].devices[].addr |
| `Pin.OUT` initial value `Pin(n).value(0)` | Initial level | standalone[].active_level |

**Example**: `I2C(0, scl=Pin(5), sda=Pin(4), freq=400000)` → I2C0 bus, SCL=GP5, SDA=GP4, 400kHz.

#### 2B: board.py — Pin mapping table

Extract all pin definitions from the `BOARDS` dictionary:

| Path | Content |
|------|------|
| `BOARDS[name]["FIXED_PINS"]` | Board fixed pins (e.g., Pico onboard LED=GP25), **must be added to mcu.pins[]** |
| `BOARDS[name]["INTERFACES"]["I2C"][id]` | I2C bus pin mapping (SDA/SCL) |
| `BOARDS[name]["INTERFACES"]["SPI"][id]` | SPI bus pin mapping (MOSI/MISO/SCK) |
| `BOARDS[name]["INTERFACES"]["UART"][id]` | UART pin mapping (TX/RX) |
| `BOARDS[name]["DEFAULTS"]` | Default frequencies/baud rates |

**Even if a device is not used in main.py, fixed pins defined in board.py (such as the onboard LED) should appear in mcu.pins[].**

#### 2C: drivers/*/__init__.py — Default I2C address & device information

For each driver's `__init__.py`, look for:

| Pattern | Extraction |
|------|------|
| `_XXXX_DEFAULT_ADDR = 0xNN` | Device default I2C address (**this is the most authoritative address source**) |
| `create_*(i2c, address=...)` | Overridable address parameter, confirms the actual address used |
| Class name / import statements | Device model and driver source |

#### 2D: conf.py — Project identity

Extract `PROJECT_NAME`, `BOARD_NAME` → fill into `meta.project`, `meta.mcu_model`.

#### 2E: tasks/*.py — Supplementary pin usage

Task files may contain additional Pin references (e.g., GPIO operations in alarm tasks). Scan to ensure nothing is missed.

#### 2F: lib/*.py — Third-party drivers

Check driver files under `firmware/lib/` for hardcoded pins or addresses. These are usually consistent with `drivers/*/__init__.py`, but sometimes driver authors hardcode default values.

### Step 3: LLM reads manifest → extracts design intent + cross-validation

Read `{project_dir}/project-manifest.json`, extract `mcu`, `pinout`, `devices`, `bom`.

**Cross-validation rules (firmware is authoritative):**

| Scenario | Handling |
|------|------|
| firmware and manifest agree | Adopt directly |
| firmware has, manifest lacks | Adopt firmware, additionally supplement pins[] / buses[].devices[] |
| manifest has, firmware lacks | Add to alerts[] marked as "planned only, not found in firmware" |
| I2C address mismatch | Use driver `_DEFAULT_ADDR` as authoritative; if main.py explicitly passes an address, use main.py |
| GPIO number mismatch | Use main.py `Pin(x)` as authoritative, add alert for mismatch with manifest |

#### 3A: Field inference rules (when manifest.pinout lacks fields)

**Physical pin number (physical_pin) inference:**

| MCU | Rule |
|-----|------|
| Raspberry Pi Pico | GP0=Pin1, GP1=Pin2, ..., GP28=Pin34. 3V3(OUT)=Pin36. GND=Pin3/8/13/18/23/28/33/38 |
| ESP32 | Consult pinout diagram (WebSearch `ESP32 pinout diagram`) |
| ESP32-S3 | Consult pinout diagram (WebSearch `ESP32-S3 pinout diagram`) |

**Pin electrical type (type) inference (combining firmware Pin initialization mode and manifest pin_name):**

| Basis for Judgment | type value |
|---|---|
| `3V3` / `3.3V` | `power_3v3` |
| `5V` / `VBUS` | `power_5v` |
| `GND` | `gnd` |
| `I2C` + `SDA` / `Data` | `i2c_data` |
| `I2C` + `SCL` / `Clock` | `i2c_clock` |
| `SPI` + `MOSI` / `TX` | `spi_mosi` |
| `SPI` + `MISO` / `RX` | `spi_miso` |
| `SPI` + `SCK` / `CLK` | `spi_sck` |
| `SPI` + `CS` / `SS` | `spi_cs` |
| `UART` + `TX` | `uart_tx` |
| `UART` + `RX` | `uart_rx` |
| `Pin(x, Pin.OUT)` — LED/buzzer/relay | `gpio_out` |
| `Pin(x, Pin.IN)` — button | `gpio_in` |
| `Pin(x, Pin.IN, Pin.PULL_UP)` | `gpio_in_pullup` |
| `ADC` / `Pin(x, Pin.IN)` + analog sensor | `adc` |
| `PWM` / `Pin(x, Pin.OUT)` + servo/dimming | `pwm` |
| I2S | `i2s` |

**Pin side (side) inference:**

| MCU | Rule |
|-----|------|
| Pico (40-pin DIP) | Left side=Pin1~20 (GP0~GP15), Right side=Pin21~40 (GP16~GP28 + power) |
| ESP32 (38-pin) | Left side=Pin1~19, Right side=Pin20~38 |

**Pin position (pos) inference:** Start from 0, increment by physical_pin within the side.

#### 3B: Power pin supplementation

**The manifest usually lacks power pins; the LLM must actively supplement them:**

- 3V3(OUT) pin: VCC for all I2C/SPI sensors, screens
- GND pin: common ground for all devices
- If there are high-power devices (servos/motors), supplement 5V/VBUS pins

#### 3C: Bus classification (based on firmware)

- I2C devices → `buses[]` type=`i2c`, signal lines SDA/SCL. I2C address uses driver `_DEFAULT_ADDR` as authoritative; if main.py explicitly passes an address, use the actual passed value
- SPI devices → `buses[]` type=`spi`, signal lines MOSI/MISO/SCK/CS
- UART devices → `buses[]` type=`uart`, signal lines TX/RX
- GPIO devices (no bus, `Pin.OUT`/`Pin.IN`) → `standalone[]`

#### 3D: Automatic alert generation

**Alert messages must be concise, each `msg` ≤60 English characters** (alert boxes in the wiring diagram have a fixed width of ~260px; overly long text will be truncated or crowd the entire layout).

**Hardware alerts:**

| Condition | level | category | msg |
|------|-------|----------|-----|
| I2C address conflict (multiple devices at same address) | `danger` | `conflict` | "{d1} and {d2} both at {addr} — address conflict" |
| I2C no pull-up resistor description | `warning` | `pullup` | "Verify I2C pull-up resistors on SDA/SCL (4.7kΩ to 3.3V)" |
| 5V device connected to 3.3V pin | `danger` | `level_shift` | "{device}: 5V device on 3.3V pin — level shifter needed" |
| 3.3V device connected to 5V pin | `danger` | `level_shift` | "{device}: 3.3V device on 5V pin — risk of damage" |
| Using GP0/GP1 (Pico boot-sensitive) | `warning` | `startup` | "GP0/GP1 used during boot on some boards; verify compatible" |
| Buzzer without current-limiting resistor | `info` | `current_limit` | "Add 220Ω current-limiting resistor in series with buzzer" |
| LED without resistor | `warning` | `current_limit` | "Add 220Ω current-limiting resistor in series with LED" |
| SPI device missing CS pin | `warning` | `general` | "SPI device {name}: missing CS pin assignment" |

**Cross-validation alerts (firmware vs manifest):**

| Condition | level | category | msg |
|------|-------|----------|-----|
| Pin used in firmware but not declared in manifest.pinout | `warning` | `firmware_only` | "GP{n} used in firmware but missing from manifest pinout" |
| Pin declared in manifest.pinout but not used in firmware | `info` | `manifest_only` | "{device}: in manifest but not found in firmware code" |
| I2C address mismatch between firmware and manifest | `danger` | `conflict` | "{device}: firmware uses {addr1}, manifest says {addr2}" |
| GPIO number mismatch between firmware and manifest | `danger` | `conflict` | "{device}: firmware uses GP{n1}, manifest says GP{n2}" |

### Step 4: LLM generates wiring.json

Based on the schema and data extracted/validated in Steps 2/3, generate `{project_dir}/docs/wiring.json`.

**Data priority: firmware > manifest > LLM inference**

**LLM decides autonomously:** `canvas` layout coordinates (can be an empty object), `mcu.orientation`, `mcu.pins[].pos` ordering, alert supplementation.

### Step 5: Validate wiring.json

```bash
python G:/MicroPython_Skills/upy-project-gen-toolchain-spec/scripts/validate_json.py \
  --schema G:/MicroPython_Skills/upy-project-gen-toolchain-spec/wiring.schema.json \
  --json {project_dir}/docs/wiring.json
```

Validation fails → modify wiring.json → re-validate until pass.

### Step 6: Generate Mermaid .md + SVG + PNG + HTML files (combined required output)

**This is the main output of this skill.** The script generates Mermaid wiring diagram .md + SVG + PNG + HTML + pin cross-reference table from wiring.json. The architecture is consistent with `upy-diagram`: JSON → Mermaid code → .md + SVG + PNG + HTML.

```bash
python G:/MicroPython_Skills/upy-wiring/scripts/render_wiring_local.py \
  --input {project_dir}/docs/wiring.json \
  --output {project_dir}/docs/
```

The script defaults to `--format all`, outputting simultaneously:

| File | Content |
|------|------|
| `docs/wiring.md` | Mermaid `graph TB` wiring diagram: MCU pin subgraph + bus subgraph + standalone GPIO + power connections + notes |
| `docs/wiring.svg` | SVG wiring diagram (vector format, clear and not blurry) |
| `docs/wiring.png` | PNG wiring diagram (bitmap format, universally compatible) |
| `docs/wiring.html` | Self-contained HTML page (Mermaid.js CDN dynamic rendering, tab switching between wiring diagram/source code, double-click in browser to view) |
| `docs/wiring_pins.md` | Markdown pin cross-reference table (GPIO → device → type → notes) |

### Step 7: SVG rendering (required, already included in Step 6's --format all)

The script defaults to using the mermaid.ink API for SVG rendering (zero local dependencies, requires network):

```bash
# SVG only (skip .md rewrite):
python G:/MicroPython_Skills/upy-wiring/scripts/render_wiring_local.py \
  --input {project_dir}/docs/wiring.json \
  --output {project_dir}/docs/ \
  --format svg
```

Principle: Mermaid code Base64 encoded → GET `https://mermaid.ink/img/{base64}?type=svg` → save SVG.

HTML uses Mermaid.js CDN to render directly in the browser, independent of mermaid.ink.

### Step 8: Update manifest

```bash
cd {project_dir} && python -c "
import json, os
from datetime import datetime, timezone
path = 'project-manifest.json'
with open(path, 'r', encoding='utf-8') as f:
    m = json.load(f)
m['wiring'] = m.get('wiring', {})
m['wiring']['json'] = 'docs/wiring.json'
m['wiring']['svg'] = 'docs/wiring.svg'
m['wiring']['png'] = 'docs/wiring.png'
m['wiring']['html'] = 'docs/wiring.html'
m['wiring']['md'] = 'docs/wiring.md'
m['wiring']['generated_at'] = datetime.now(timezone.utc).isoformat()
with open(path, 'w', encoding='utf-8') as f:
    json.dump(m, f, ensure_ascii=False, indent=2)
print('[OK] manifest wiring updated')
"
```

---

## Relationship with Other Skills

- ← `upy-scaffold` / `upy-generate`: Input firmware/ source code + manifest (including pinout/mcu/devices/bom)
- Parallel with `upy-diagram`: Can be generated simultaneously, sharing the mermaid.ink SVG rendering pipeline
- → VS Code extension WebView: Display Mermaid diagram (Markdown preview) or PNG

---

## Strong Constraints

- **Firmware is the authoritative data source, manifest is the design reference**: When they conflict, firmware takes precedence
- **Must read all .py source files in firmware/**: This step cannot be skipped
- **board.py's FIXED_PINS must be added to mcu.pins[]**: e.g., Pico onboard LED=GP25
- **LLM generates JSON, scripts only perform validation + rendering**: Consistent with `upy-generate` / `upy-diagram` pattern
- **Schema is the sole contract**: wiring.json must pass validation by `validate_json.py`
- **LLM must infer missing fields**: When manifest.pinout data is incomplete, prioritize completion from firmware, then based on Pico/ESP32 pinout diagram knowledge
- **LLM must supplement power pins**: 3V3, GND must always be added to mcu.pins[]
- **Pin type enum must match**: `mcu.pins[].type` must be an enum value defined in the schema
- **I2C devices must have `addr`**: Format `0x00`, regex `^0x[0-9a-fA-F]{2}$`. Address uses driver `_DEFAULT_ADDR` as authoritative
- **SPI devices must have `cs_gpio`**: Chip select pin
- **Alerts are judged by the LLM according to rules and written into alerts[]**
- **SVG + PNG + HTML are required outputs**: The script defaults to `--format all`, generating .md, .svg, .png, and .html simultaneously; only `--format md` can skip image rendering
- **canvas can be an empty object**: The renderer auto-layouts, does not require the LLM to calculate coordinates
- **Rendering script reads defensively**: Missing fields will not cause a crash, but will output warnings to stderr
- **Shares mermaid.ink pipeline with upy-diagram**: Both use the same PNG rendering method
- **Readability constraints (ensuring PNG is clearly readable at ~1200px width)**:

  | Field | Upper Limit | Description |
  |------|------|------|
  | `alerts[].msg` | ≤60 English characters | Alert box width ~260px; overly long text is truncated or crowds the layout |
  | `standalone[].external_components` | ≤20 characters | Device accessory description; overly long text expands the standalone device box |
  | `buses[].devices[].notes` | ≤20 characters | Device notes; keep concise |
