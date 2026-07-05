# Board Database Documentation

> Target audience: Plugin engineers, server-side engineers, Skill maintainers
>
> Version: 2.0 / Last updated: 2026-06-16

---

## 1. File Location

```
G:\MicroPython_Skills\upy-analyze\boards\
├── README.md               ← This file
├── _template.json          ← Template for adding new boards (copy and fill)
├── matching-rules.json     ← Board selection scoring rules
├── esp32-devkit-v1.json    ← ESP32 DevKit V1
├── esp32-s3-devkitc.json   ← ESP32-S3-DevKitC-1
├── esp32-c3-devkitm.json   ← ESP32-C3-DevKitM-1
├── raspberry-pi-pico.json  ← Raspberry Pi Pico
├── raspberry-pi-pico-w.json← Raspberry Pi Pico W
├── esp8266-nodemcu.json    ← ESP8266 NodeMCU V3
└── m5stack-core.json       ← M5Stack Core (ESP32)
```

**One JSON file per board.** The filename equals the board id, with a `.json` suffix.

---

## 2. How to Use the Template

Copy `_template.json`, rename it to `{board_id}.json`, and fill in each field according to the instructions below. The template already lists all fields; leave empty fields as-is, do not delete them.

Two sections in the template are optional:
- `onboard_peripherals`: What peripherals the board comes with. For pure GPIO development boards (like ESP32 DevKit), use an empty array `[]`
- `pin_layout.pin_options`: Only fill this for chips where pins and peripherals are fixedly bound, like the RP2040/Pico. Leave as an empty object `{}` for ESP32 series

---

## 3. Field Descriptions

### 3.1 Basic Information (Required for Every Board)

| Field | Type | Description |
|------|------|------|
| `id` | string | Unique identifier. Naming convention: `{chip}-{board_type}`, all lowercase with hyphens. E.g., `esp32-devkit-v1` |
| `display_name` | string | Name displayed in the plugin UI, use the official full name. E.g., `ESP32-S3-DevKitC-1` |
| `mcu` | string | Full MCU model, will appear in the BOM. E.g., `ESP32-S3-WROOM-1` |
| `chip_family` | string | Chip family. Allowed values: `esp32` / `esp32s3` / `esp32c3` / `esp8266` / `rp2` |

### 3.2 Firmware Information (Required for Every Board)

| Field | Type | Description |
|------|------|------|
| `firmware.url` | string | MicroPython official download page URL |
| `firmware.port` | string | MicroPython port: `esp32` / `rp2` / `esp8266` |
| `firmware.board_name` | string | Board target name used during compilation, e.g., `ESP32_GENERIC_S3` |
| `firmware.latest_version` | string | Known latest firmware version number |

Once a board is selected, the firmware URL is determined. `upy-select-hw` will no longer search for firmware online.

### 3.3 Hardware Specifications (Required for Every Board)

There are 13 fields in specs. They are divided into three tiers based on how frequently AI uses them:

**★★★ Affects AI selection and code generation, must be filled accurately:**

| Field | Type | How AI Uses It |
|------|------|---------|
| `flash_mb` | number | Determines if the filesystem can hold all drivers + logs |
| `psram_mb` | number | 0 = no PSRAM; AI will warn about risks when allocating large buffers |
| `gpio` | number | Many peripherals + few GPIOs → AI will warn about insufficient pins |
| `i2c` | number | Many I2C devices but few controllers → suggests software I2C |
| `spi` | number | Many SPI devices but few controllers → suggests sharing the bus |
| `wifi` | boolean | Matches networking requirements vs board capabilities |
| `ble` | boolean | Matches Bluetooth requirements vs board capabilities |

**★★ Referenced during pin allocation, fill as accurately as possible:**

| Field | Type | How AI Uses It |
|------|------|---------|
| `pwm` | number | Servo/LED dimming requires PWM channels |

**★ Displayed on the board details page; filling it is better, but not filling it does not affect AI decisions:**

| Field | Type | Description |
|------|------|------|
| `adc` | number | Number of analog input channels |
| `dac` | number | Number of analog output channels |
| `uart` | number | Number of hardware UARTs |
| `touch` | number | Number of touch pins |
| `usb_otg` | boolean | Whether USB device mode is supported |

### 3.4 Purchase Information (Required for Every Board)

| Field | Type | Description |
|------|------|------|
| `typical_use_cases` | string[] | Typical use cases. Used by the plugin as filter tags, by the LLM for selection matching |
| `beginner_friendly` | boolean | Whether it is suitable for beginners. `upy-analyze` prioritizes recommending these in beginner mode |
| `price_yuan` | number | Reference price (CNY), for display |
| `notes` | string | Notes for the user. **The LLM also reads this section for decision-making.** Write key technical limitations, compatibility warnings, and key differences from other boards |

### 3.5 Onboard Peripherals `onboard_peripherals` (Optional)

**When to fill:** When the board comes with peripherals like screens, sensors, buttons, etc. (not just indicator LEDs).

**When to use an empty array:** For pure GPIO development boards, e.g., an ESP32 DevKit only has one onboard LED; write one entry or use an empty array directly.

Fields for each peripheral:

| Field | Type | Required | Description |
|------|------|------|------|
| `name` | string | Yes | Peripheral name, e.g., "ILI9342C LCD" |
| `type` | string | Yes | Category. Allowed values: `display` / `sensor` / `imu` / `button` / `led` / `led_rgb` / `speaker` / `storage` / `power_mgmt` / `wifi_module` / `usb_uart` |
| `interface` | string | Yes | I2C / SPI / GPIO / I2S / UART |
| `occupied_pins` | object | Yes | Which pins are occupied. Key = function name, value = GPIO number. -1 means no GPIO is occupied |
| `i2c_addr` | string | Required for I2C devices | I2C address, e.g., "0x68" |
| `driver` | object? | Fill if a known driver exists | Driver information (see table below) |
| `always_used` | boolean | Yes | true = this pin is definitely occupied, skip during pin allocation. false = the user can disable this peripheral, freeing the pin |
| `notes` | string | No | Supplementary notes |

Driver object (fill if a known driver exists):

| Field | Description |
|------|------|
| `source` | Driver source, usually `"upypi"` |
| `package_name` | Package name on upypi |
| `url` | Driver page or API link |
| `install_cmd` | mpremote installation command |

**Note:** Do not fill in uncertain drivers. Only fill in driver information after verification that the driver works with this peripheral on this board. Otherwise, leave it blank and let the AI search for it.

### 3.6 Pin Layout `pin_layout` (Required for Every Board)

Different chips have completely different pin allocation methods. First, look at the `model` field:

| model | Applicable Chips | Meaning |
|-------|---------|------|
| `"flexible"` | ESP32 / ESP32-S3 / ESP32-C3 / ESP8266 | I2C/SPI/UART can be mapped to any free GPIO. Constraints come from "which pins cannot be touched" |
| `"fixed"` | RP2040 / Pico | Each peripheral function can only be selected from a fixed set of pins. Constraints come from "which pins can be used" |

#### flexible model

**Equivalent to "tell the AI to avoid the minefield, the rest is free to allocate."**

`default_bus_pins`: A set of default pins for each bus. The AI prioritizes these, and switches if they don't work. Format:

```json
"default_bus_pins": {
  "i2c0": { "sda": 21, "scl": 22 },
  "spi0": { "mosi": 23, "miso": 19, "clk": 18, "cs": 5 },
  "uart0": { "tx": 1, "rx": 3 }
}
```

`restricted_gpio`: Pins that cannot be touched, categorized by reason:

| Category | Meaning | Example |
|------|------|------|
| `input_only` | Can only be used as input, cannot output or use pull-up/pull-down | ESP32 GPIO 34-39 |
| `strapping` | Determines the operating mode at startup; incorrect connections may prevent the board from booting | ESP32 GPIO 0/2/5/12/15 |
| `flash_psram_occupied` | Internally occupied by Flash/PSRAM, not visible externally | ESP32 GPIO 6-11 |
| `adc2_wifi_conflict` | ADC2 is unreliable when WiFi is on | Most ESP32 ADC2 channels |
| `usb_otg_pins` | Dedicated to USB OTG; repurposing them loses USB functionality | ESP32-S3 GPIO 19/20 |
| `usb_serial_pins` | USB serial pins, used for debugging and flashing | ESP32-C3 GPIO 18/19 |
| `boot_fail_risk` | May not necessarily cause boot failure, but historically people have had issues | ESP8266 GPIO 0/1/2/3/9/10 |
| `wifi_chip_occupied` | Internally occupied by the WiFi module | Pico W GPIO 23/24/25 |
| `onboard_occupied` | Summary: list of all pins occupied by onboard peripherals | M5Stack various peripheral pins |

`pin_options`: Leave as `{}` for the flexible model.

#### fixed model

**Equivalent to "tell the AI each function can only be selected from a specific set of pins."**

`default_bus_pins`: Same as above, just fill in the array.

`restricted_gpio`: Same as above, fill in pins that cannot be occupied.

`pin_options`: **This is the key field for the fixed model.** List the optional pins for each peripheral function:

```json
"pin_options": {
  "i2c0_sda": [0, 4, 8, 12, 16, 20],
  "i2c0_scl": [1, 5, 9, 13, 17, 21],
  "spi0_mosi": [3, 7, 11, 15, 19, 23],
  ...
}
```

SDA and SCL must be selected as a pair from the same group. For example, if you choose I2C0 SDA=8, you must use SCL=9; mixing is not allowed.

---

## 4. matching-rules.json Description

Scoring rules used by the LLM for board selection. Each rule contains:

| Field | Description |
|------|------|
| `id` | Rule ID |
| `trigger` | Trigger condition (for human reading; the LLM determines if it triggers) |
| `action` | `boost` (add points) or `exclude` (eliminate) |
| `chip_families` | Which chip_families this applies to |
| `note` | Supplementary notes |

Usage:
1. The LLM reads the user's requirements and determines which rules are triggered
2. For each board: boost rules matching the chip_family → add points; exclude rules matching → eliminate
3. The two boards with the highest scores are recommended
4. An excluded board is not necessarily bad; it may just be unsuitable for the current scenario (e.g., excluding a board with only 1 I2C bus when multiple I2C devices are needed)

---

## 5. How the Plugin Uses This

### 5.1 Getting the Board List

**Local testing phase:** The plugin reads the JSON files directly from this directory.

**Production phase:** The server provides an API, which the plugin calls:
```
GET /v1/boards
→ { version: "2.0", boards: [...] }
```

When the server starts, it scans all `*.json` files in this directory (excluding `_template.json` and `matching-rules.json`), merges them, and returns the result.

### 5.2 Rendering the Board Gallery

The plugin uses these fields when rendering the board selector in the sidebar:

| UI Location | Field |
|---------|------|
| Card title | `display_name` |
| Chip model | `mcu` |
| Key specifications | `specs.wifi`, `specs.ble`, `specs.gpio`, `specs.i2c`, `specs.spi` |
| Beginner badge | `beginner_friendly` → "Beginner Recommended" badge |
| Price tag | `price_yuan` |
| Use case tags | `typical_use_cases` (clickable for filtering) |
| Detail popup | All 13 specs items + `onboard_peripherals` list + `pin_layout.notes` |

### 5.3 After the User Selects a Board

The plugin puts a simplified version of the selected board into the request:

```json
{
  "pre_selected_board": {
    "id": "esp32-devkit-v1",
    "display_name": "ESP32 DevKit V1",
    "mcu": "ESP32-WROOM-32",
    "chip_family": "esp32",
    "firmware_url": "https://micropython.org/download/ESP32_GENERIC/"
  }
}
```

Only these 5 fields are sent. The full specs and pin_layout are read from the server-side complete JSON when the Skill needs them.

---

## 6. Fields That Will Be Updated / May Expire

| Field | Update Frequency | Who Updates |
|------|---------|---------|
| `firmware.latest_version` | When MicroPython releases a new version | Server CI automatically checks and updates |
| `firmware.url` | When MicroPython changes the download URL (very rare) | Manual update |
| `price_yuan` | Market fluctuations | Manual maintenance, check every six months |
| `specs` values | Will not change (determined at chip manufacture) | No update needed |
| `driver` information | When upypi packages are updated or renamed | Update after manual verification |

---

## 7. Steps to Add a New Board

1. Copy `_template.json`, rename it to `{board_id}.json`
2. Fill in basic information (id / display_name / mcu / chip_family)
3. Fill in firmware (go to micropython.org/download to find the corresponding board_name and URL)
4. Fill in specs (consult the chip datasheet, fill in all 13 items according to the actual situation)
5. Fill in typical_use_cases / beginner_friendly / price_yuan / notes
6. Fill in onboard_peripherals:
   - Board only has one LED → write one entry or use an empty array
   - Board has a screen/sensor/button → write occupied_pins for each peripheral, and whether there is a driver link
7. Fill in pin_layout:
   - ESP32/ESP8266 → model: "flexible", fill in default_bus_pins + restricted_gpio, leave pin_options empty
   - RP2040/Pico → model: "fixed", fill in all fields including pin_options
8. Check matching-rules.json: does the new chip_family have corresponding rules? If not, add them
9. Verify: `python -c "import json; json.load(open('{new_file}.json'))"` should not produce errors
10. Commit

Adding a new board does not require modifying the plugin code — the plugin dynamically renders the list after pulling it from the API.
