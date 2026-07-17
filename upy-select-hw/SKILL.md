---
name: upy-select-hw
description: Step 2 — MCU selection + firmware verification + pin assignment + BOM generation. Takes the project-manifest.json from upy-analyze as input and outputs a complete hardware plan. Triggered automatically after upy-analyze completes.
---

# Hardware Selection and Pin Assignment Skill

## Role

Given `project-manifest.json` (devices + requirements + mcu_specified), complete MCU selection, firmware verification, pin assignment, and BOM generation. **Do not write code, do not handle drivers.**

## Pre-check

```bash
python --version
```

---

## Execution Steps

### Step 1: MCU Selection + Firmware Verification

#### Case A: User has specified an MCU (`mcu_specified` has a value)

```
1. Verify MicroPython firmware support:

   Known supported models → pass directly (see table below)
   Uncommon models → WebSearch: site:micropython.org/download {model}
   
   No firmware → Stop! Inform the user and suggest alternatives:
     ESP32 (most versatile) / Pico (cost-effective) / ESP32-S3 (AI capability)

2. Output firmware download link:
   URL: https://micropython.org/download/{BOARD_NAME}/
```

#### Case B: User has not specified an MCU → LLM recommends

**Recommendation strategy: Prioritize Pico series and ESP32 series (best MPY support).**

```
Scoring logic:

   Needs WiFi/BLE     → +1 ESP32 series, +1 Pico W
   Needs AI/Voice/Camera → +1 ESP32-S3
   Low power + battery powered → +1 ESP32-C3
   Pure GPIO control       → +1 Pico series
   Extremely low cost           → +1 ESP8266 / Pico
   Beginner friendly           → +1 Pico (USB drag-and-drop flashing) / ESP32

   Final recommendation: Top 1, with a brief reason.

   Alternative: Top 2 (user can switch)
```

**Recommendation output example:**

```
Recommended MCU: Raspberry Pi Pico W
  Reason: Needs WiFi (requirements.network=wifi), RP2040 is cost-effective,
        MPY support is excellent, USB drag-and-drop flashing is beginner-friendly.

Alternative: ESP32 (WiFi + BLE, most complete ecosystem, more interfaces)

Confirm usage? Or specify another model.
```

#### Firmware Download Link Mapping (Known Models)

| MCU | BOARD_NAME | Flashing Method |
|-----|-----------|---------|
| ESP32 | ESP32_GENERIC | esptool.py |
| ESP32-S3 | ESP32_GENERIC_S3 | esptool.py |
| ESP32-C3 | ESP32_GENERIC_C3 | esptool.py |
| ESP32-S2 | ESP32_GENERIC_S2 | esptool.py |
| ESP32-C6 | ESP32_GENERIC_C6 | esptool.py |
| Pico | RPI_PICO | Hold BOOTSEL, drag-and-drop .uf2 |
| Pico W | RPI_PICO_W | Same as above |
| Pico 2 | RPI_PICO2 | Same as above |
| Pico 2 W | RPI_PICO2_W | Same as above |
| ESP8266 | ESP8266_GENERIC | esptool.py |
| STM32F4DISC | STM32F4DISC | dfu-util |
| STM32F7DISC | STM32F7DISC | dfu-util |
| Pyboard | PYBV11 | dfu-util |
| Teensy 4.0 | TEENSY40 | Teensy Loader |
| Teensy 4.1 | TEENSY41 | Teensy Loader |

---

### Step 2: Pin Assignment

#### Step 2A: Get the Pinout Diagram

```
Please upload a pinout diagram of your development board (photo/screenshot/PDF are all acceptable).
Search for "{MCU model} pinout" or "{MCU model} pinout diagram" to find one.
```

If the user says "can't find it" → Use `WebSearch` to search for `{MCU model} pinout diagram`, take the first image and show it to the user for confirmation.

#### Step 2B: LLM Multimodal Recognition

Extract from the pinout diagram:
- List of available GPIO numbers
- Default pins for hardware I2C (e.g., ESP32: I2C0 SCL=22 SDA=21)
- Default pins for hardware SPI
- Default pins for hardware UART (note that UART0 is occupied by REPL)
- Power pin locations (3.3V, 5V, GND)
- Boot/flashing sensitive pins (e.g., ESP32: GPIO0/2/5/12/15)
- Read-only pins (e.g., ESP32: GPIO34-39)
- Pins occupied by Flash/PSRAM (e.g., ESP32: GPIO6-11)

#### Step 2C: Assign Pins

**LLM performs allocation based on the following rules:**

```
Rule 1 — I2C devices:
  ├─ All I2C devices on the same I2C bus (default I2C0)
  ├─ Address conflict → Use a second I2C bus (if available) or Software I2C (any GPIO)
  └─ Each I2C bus uses 2 GPIOs (SCL + SDA)

Rule 2 — SPI devices:
  ├─ Share MOSI/MISO/SCK, each device has its own CS
  ├─ Use hardware SPI default pins
  └─ N SPI devices use 3 + N GPIOs

Rule 3 — UART devices:
  ├─ Prefer UART1/UART2 (UART0 is occupied by REPL)
  └─ Each UART device uses 2 GPIOs (TX + RX)

Rule 4 — Simple GPIO devices (LED/Buzzer/Button/Relay):
  ├─ Prefer pins away from I2C/SPI buses
  ├─ Avoid boot-sensitive pins
  ├─ Avoid read-only pins
  └─ Each device uses 1 GPIO

Rule 5 — ADC devices:
  ├─ Can only use ADC pins (e.g., ESP32: ADC1 from GPIO32-39)
  └─ Note: ESP32 ADC2 conflicts with WiFi

Rule 6 — Conflict detection:
  ├─ The same GPIO cannot be assigned twice
  ├─ Print the pin occupancy table after assignment
  └─ Mark shared pins (e.g., multiple devices on an I2C bus)
```

**Assignment output format:**

```
Pin Assignment Plan:

  I2C Bus (I2C0):
    SCL = GPIO22, SDA = GPIO21
    Devices: SHT30 (0x44), SSD1306 (0x3C), BMP280 (0x76)
    No address conflict ✓

  GPIO Independent:
    Buzzer = GPIO4
    LED    = GPIO13

  Unused default pins: SPI (no SPI devices)

  Pin occupancy: 6/26 GPIO
  Conflict check: Passed ✓
```

**Pin Electrical Type (type) Enumeration Mapping:**

| Pin Usage | type value |
|---------|---------|
| 3.3V Power Output | `power_3v3` |
| 5V Power Output | `power_5v` |
| GND | `gnd` |
| I2C SDA | `i2c_data` |
| I2C SCL | `i2c_clock` |
| SPI MOSI | `spi_mosi` |
| SPI MISO | `spi_miso` |
| SPI SCK | `spi_sck` |
| SPI CS | `spi_cs` |
| UART TX | `uart_tx` |
| UART RX | `uart_rx` |
| GPIO Output (LED/Buzzer/Relay) | `gpio_out` |
| GPIO Input (Button) | `gpio_in` |
| GPIO Input + Pull-up | `gpio_in_pullup` |
| ADC Input | `adc` |
| PWM Output | `pwm` |
| I2S | `i2s` |

**Physical Pin Number (physical_pin) Retrieval Rules:**
- Pico series: GP0=Pin1, GP1=Pin2, ..., GP28=Pin34; 3V3(OUT)=Pin36; GND=Pin3/8/13/18/23/28/33/38
- ESP32 series: Consult the pinout diagram, note the physical pin number corresponding to the GPIO number
- Other MCUs: Obtain from the pinout diagram / datasheet

#### Step 2D: Power Pin Assignment

**LLM must also write power pins into the pinout:**

```
Power Pin Assignment:
  3V3(OUT) → VCC for all I2C/SPI devices (sensors, screens, etc.)
  5V(VBUS) → High-power devices requiring 5V (servos, motors, etc.)
  GND      → GND for all devices (one wire per device)

Add entries to pinout:
  {device: "Power", pin_name: "3V3(OUT)", gpio: "3V3", physical_pin: 36, type: "power_3v3", side: "right", pos: 16}
  {device: "Power", pin_name: "GND", gpio: "GND", physical_pin: 38, type: "gnd", side: "right", pos: 18}
```

---

### Step 3: BOM Generation

```
Bill of Materials:

  #  Name              Model              Qty  Unit Price  Notes
  1  MCU               {MCU model}        1    ¥{xx}       Includes USB cable
  2  {Device 1}        {Model}            1    ¥{xx}       {Interface}
  3  {Device 2}        {Model}            1    ¥{xx}       {Interface}
  -  Breadboard        830 holes          1    ¥8          Optional
  -  Jumper Wires      Male-Female 20pcs  1    ¥5
  -  USB Data Cable    Micro-USB          1    ¥5          (if not included with MCU)

  Estimated Total: ¥{total}

  vs User Budget: {budget_yuan}
  {Over budget / Within budget}
```

Price source: LLM knowledge + common sense estimation.

---

### Step 4: Update Manifest

Call the script to write to `project-manifest.json`:

```bash
python G:/MicroPython_Skills/upy-select-hw/scripts/update_manifest.py \
  --project-dir {project_dir} \
  --input {llm_output_json}
```

--- Fields to write:
- `phase`: "select-hw"
- `mcu`: {model, board, firmware_url, flash_tool}
- `pinout`: [{device, pin_name, gpio, physical_pin, type, side, pos, notes}]
  - `physical_pin`: Physical pin number (e.g., Pico's GP4 = Pin 6)
  - `type`: Pin electrical type enumeration (see mapping table below)
  - `side`: Which side of the MCU the pin is on (left/right/top/bottom)
  - `pos`: Sequential position on the side (0-based)
- `bom`: [{name, model, quantity, unit_price_yuan, notes}]

---

## Relationship with Other Skills

- ← `upy-analyze`: Input manifest
- → `upy-scaffold`: Passes the complete hardware plan (mcu + pinout + bom)

## Hard Constraints

- **Only recommend Pico series and ESP32 series for the MCU** (unless the user specifies another model)
- **Firmware verification is mandatory** — confirm MPY firmware exists before proceeding
- **Must see the pinout diagram before assigning pins** — do not rely on a built-in database
- **I2C address conflicts must be detected** — cannot place two devices with the same address on the same bus
- **Boot-sensitive pins must be avoided**
