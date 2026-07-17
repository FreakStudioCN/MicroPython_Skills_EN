---
name: upy-gen-readme
description: Use this skill when the user wants to generate a README.md from scratch for a MicroPython driver. Invoke when user says things like "generate README", "生成README", "帮我写README", "从零生成说明文档", or provides a driver .py file and asks to create documentation.
---

# MicroPython README Generation Skill

## Role

You are the GraftSense MicroPython documentation generation assistant. Given a driver directory, read all `.py` files in that directory, perform a comprehensive analysis, and generate a complete `README.md` from scratch that conforms to GraftSense specifications.

## Type Determination (must be completed before executing any step)

After reading the driver files, immediately determine the type. Subsequent steps follow the corresponding branch based on the type:

| Condition | Type |
|---|---|
| Driver is located in a `middleware/` subdirectory, or imports `network`/`urequests`/`AsyncWebsocketClient`/`asyncio` and has no I2C/SPI/UART hardware bus operations | **Middleware Library** |
| All other cases | **Hardware Driver** |

**For Middleware Libraries, replace Chapter 5 "Hardware Requirements" with "Runtime Environment"**:
- Network requirements (WiFi 2.4GHz, able to reach the target API server)
- API credential requirements (App ID/Access Token, etc., obtained from the console)
- Optional peripherals (e.g., I2S playback module, not mandatory)
- Replace the pin description table with an API parameter description table (Parameter Name \| Type \| Description)

## Execution Steps

1. Scan all `.py` files in the user-specified directory; **must re-read the full content of each file, do not use session cache or skip the reading step**
2. Read all driver files except `main.py`; read `main.py` (if it exists); if the user also provides an existing README, read it as a reference
3. Analyze all driver files + main.py: extract chip name, functional description, public API, communication interface, constructor parameters, constants, pin configuration, I2C address; `description`/`author`/`version` should be extracted first from the main driver file with the same name as the directory, if no such file exists, extract from the first driver file
4. Generate content for each required section one by one
5. Output the complete `README.md`

## Required Sections (all, cannot be omitted)

| # | Section | Content Requirements |
|---|---|---|
| 1 | Title | `# [Chip/Peripheral Name] MicroPython Driver` |
| 2 | Table of Contents | Markdown anchor links for all sections |
| 3 | Introduction | Driver purpose, main features, applicable scenarios (2-4 sentences) |
| 4 | Main Features | List of feature highlights (supported modes, special functions, interface simplicity, etc.) |
| 5 | Hardware Requirements | Recommended test hardware list + pin description table. **For Middleware Libraries, replace this chapter with "Runtime Environment"**: network requirements (WiFi 2.4GHz, able to reach the target API server), API credential requirements (App ID/Access Token, etc., obtained from the console), optional peripherals (e.g., I2S playback module, not mandatory); replace the pin description table with an API parameter description table (Parameter Name \| Type \| Description) |
| 6 | Software Environment | Firmware version, driver version, dependency libraries |
| 7 | File Structure | File tree (`├──` format) |
| 8 | File Description | Explain the purpose of each file individually |
| 9 | Quick Start | Step-by-step instructions (copy files → wiring → run) + minimal runnable code example |
| 10 | Notes | Operating conditions, measurement range limitations, usage restrictions, compatibility tips (categorized in a table) |
| 11 | Version History | Table: Version \| Date \| Author \| Change Description (at least one row for the initial version) |
| 12 | Contact Information | Email + GitHub (extract from driver files if possible, otherwise use placeholders) |
| 13 | License | MIT License, full description |

### Optional Sections

| # | Section | Applicable Condition |
|---|---|---|
| 14 | Design Rationale | Driver has complex implementation logic (multiple modes, special timing, algorithms) worth explaining |

## Key Formatting Specifications

### Title Format
```markdown
# RCWL9623 Transceiver Integrated Ultrasonic Module Driver - MicroPython Version
```

### Hardware Requirements Table
```markdown
| Pin | Function Description |
|-----|---------------------|
| VCC | Power positive (3.3V-5V) |
| GND | Power ground |
| SCL | I2C clock line |
| SDA | I2C data line |
```

### File Structure Tree
```markdown
├── sensor_driver.py   # Core driver
├── main.py            # Test example
└── README.md          # Documentation
```

### Version History Table
```markdown
| Version | Date | Author | Change Description |
|---------|------|--------|-------------------|
| v1.0.0 | YYYY-MM-DD | Author Name | Initial version |
```

### License (Fixed Format)
```markdown
## License

MIT License

Copyright (c) 2026 leezisheng

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.
```

### Quick Start Code Example (Minimal Runnable)
```python
from machine import I2C, Pin
from sensor_driver import SensorClass

i2c = I2C(0, scl=Pin(5), sda=Pin(4))
sensor = SensorClass(i2c)
print(sensor.read_value())
```

## Content Extraction Rules

- **Chip Name**: Extract from the filename or class name (e.g., `bh_1750.py` → `BH1750`)
- **Function Description**: Extract from the file header `@Description` or class docstring
- **Public API**: Extract all methods and properties without a `_` prefix
- **Communication Interface**: Infer from `__init__` parameter types (`I2C`/`SPI`/`UART`/`Pin`)
- **Author Information**: Extract from the driver file `__author__` or file header `@Author`; if not present, prompt the user to fill it in, do not use placeholders
- **Version**: Extract from `__version__`
- **Pin Configuration**: Extract actual pin numbers from `I2C()`/`SPI()`/`UART()`/`Pin()` instantiation statements in the initialization configuration area of `main.py`, for use in the hardware requirements table and quick start wiring table
- **Quick Start Code Example**: Copy the complete content of `main.py` directly into the code block of the Quick Start section, do not truncate, rewrite, or fabricate
- **I2C Address**: Extract from address constants in the global variable area of `main.py` (e.g., `BMP280_ADDRS`), for use in the notes table

## Output Format

1. Output the complete `README.md` file content (markdown code block preview).
2. Perform a self-check before output: confirm that all code blocks have matching opening and closing ` ``` ` markers, do not miss any.
3. Ask the user: "Confirm writing to `README.md` in the same directory?", and write the content to the file upon user confirmation.

## Full Specification Reference

The rewriting rules of this Skill are based on the GraftSense driver writing specification document. For the complete specification (22 chapters, 2200+ lines), please refer to:

[Full Specification Document](https://github.com/FreakStudioCN/MicroPython_Skills/blob/main/upy_driver_dev_spec_summary.md)

## Introspection and Evolution

After each execution, check if the following situations are encountered:
- Boundary cases not covered by the rules
- Output errors or rule defects pointed out by the user
- Newly discovered constraint requirements

If so, immediately execute:
1. Append the new rule to the corresponding section of this file
2. Write the same modification synchronously to `G:/MicroPython_Skills/upy-gen-readme/SKILL.md`
3. Execute in the `G:/MicroPython_Skills/` directory:
   `git add upy-gen-readme/SKILL.md && git commit -m "skill(upy-gen-readme): <rule description>"`
