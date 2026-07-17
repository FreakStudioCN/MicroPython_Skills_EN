# Pin Assignment Log

Sample select-hw pin assignment log for ESP32-C3-DevKitM-1.

Board definition: `upy-analyze-plugin/boards/esp32-c3-devkitm.json`

## GPIO Usage Summary

- Used GPIOs: GPIO4, GPIO5, GPIO6, GPIO7, GPIO10, GPIO11, GPIO20, GPIO21
- Unused GPIOs: GPIO0, GPIO1, GPIO2, GPIO3, GPIO8, GPIO9, GPIO12, GPIO13, GPIO18, GPIO19
- Conditional/Reserved GPIOs: GPIO2, GPIO8, GPIO9 (strapping boot pins)
- Forbidden GPIOs: (none)

## Pin Assignment Details

| Device | Signal | GPIO | Type | Bus | Source |
|--------|--------|------|------|-----|--------|
| AHT20 | SDA | 5 | i2c_data | i2c0 | default_bus |
| AHT20 | SCL | 6 | i2c_clock | i2c0 | default_bus |
| HC-SR501 | OUT | 4 | gpio_in | - | auto_assigned |
| TTP223 | OUT | 7 | gpio_in | - | auto_assigned |
| INMP441 | BCK | 10 | i2s_bck | i2s0 | auto_assigned |
| INMP441 | WS | 11 | i2s_ws | i2s0 | auto_assigned |
| INMP441 | SD | 20 | i2s_data_in | i2s0 | user_wiring |
| MAX98357 | DIN | 21 | i2s_data_out | i2s0 | user_wiring |
| power | 3V3 | 3V3 | power_3v3 | - | power |
| power | GND | GND | gnd | - | power |

## Risks and Notes

- GPIO4, GPIO5 belong to ADC2/WiFi conflict pins; used only for digital purposes (gpio_in/i2c_data), no functional impact
- GPIO20, GPIO21 are USB serial pins, retained per user wiring; subsequent debugging must avoid occupying USB CDC
- I2S BCK/WS are shared by the microphone and amplifier
