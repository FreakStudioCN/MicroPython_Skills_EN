---
name: upy-gen-pkg
description: Use this skill when the user wants to generate a package.json from scratch for a MicroPython driver package. Invoke when user says things like "generate package.json", "生成package.json", "帮我写包配置", "创建mip包配置", or provides a driver directory/file and asks to create a package config.
---

# MicroPython package.json Generation Skill

## Role

You are the GraftSense MicroPython package configuration generation assistant. Given a driver directory or driver `.py` file, analyze its structure and dependencies, and generate a complete `package.json` from scratch that conforms to GraftSense specifications.

## Execution Steps

1. Scan the user-specified directory:
   - **1a**: Scan all top-level `.py` files, excluding `main.py`, as the driver file list; **must re-read the full content of each file, do not use session cache or skip the reading step**
   - **1b**: Scan all subdirectories containing `__init__.py` as **sub-package dependency candidate list**
2. Sub-package dependency handling (see "Sub-package Dependency Handling" section)
3. Extract from all driver files: file name list, `@Author`, `@Description`, `__version__`, `__license__`, all `import` statements (merge and deduplicate); `author`/`version`/`description` are preferentially extracted from the main driver file with the same name as the directory; if no such file exists, extract from the first `.py` file
4. Analyze the source type of each import (see dependency handling steps)
5. Query upypi for each third-party dependency
6. Generate the complete `package.json`

## Mandatory Fields (All)

| Field | Generation Rule |
|---|---|
| `name` | Extract from the directory name, convert to lowercase letters + underscores (e.g., `BH1750_driver` → `bh1750_driver`) |
| `urls` | Scan all **top-level** `.py` files in the directory (excluding `main.py`), generate one `["filename.py", "code/filename.py"]` mapping per file; subdirectories containing `__init__.py` are handled according to the developer's choice: option ② packages them into `urls` (with subdirectory path prefix), options ①③ do not write them into `urls` |
| `version` | Extract from `__version__`, default to `"1.0.0"` if absent |
| `_comments` | Fixed content (see template below) |
| `description` | Extract from `@Description` or class docstring, in English |
| `author` | Extract from the driver file's `__author__` or file header `@Author`; if absent, prompt the user to fill it in, do not use placeholders |
| `license` | Extract from `__license__`, default to `"MIT"` |
| `chips` | Default to `"all"`, unless the driver explicitly depends on a specific chip (e.g., RP2040 PIO) |
| `fw` | Default to `"all"`, unless there is a special firmware dependency (ulab, lvgl, etc.) |

## Sub-package Dependency Handling

For each subdirectory containing `__init__.py` scanned in step 1b, process according to the following flow:

### When sub-package directories exist

**Prefer using Bash tools to execute curl for automatic query**:

```bash
curl -s "https://upypi.net/api/search?q={subdirectory_name}"
```

- **Has results**: Write the url into `deps`: `["{url}", "latest"]`
- **No results**: Ask the developer:
  ```
  Found sub-package directory `{subdirectory_name}/` (contains __init__.py), not yet indexed on upypi.
  Please choose how to handle:
  ・① Publish as an independent package → It is recommended to complete the upypi release first, then generate package.json
  ・② Package into this driver's urls → Write all files in the subdirectory into urls one by one
  ・③ github placeholder → Write into deps, marked ⚠️ needs manual confirmation
  ```
  - Option **①**: Pause, wait for the user to complete the release before continuing
  - Option **②**: Scan all `.py` files in this subdirectory (including sub-levels), append them to `urls` one by one in the following format:
    ```json
    ["{subdirectory_name}/filename.py", "code/{subdirectory_name}/filename.py"]
    ```
    Example (sensor_pack_2 has 3 files):
    ```json
    ["sensor_pack_2/__init__.py",    "code/sensor_pack_2/__init__.py"],
    ["sensor_pack_2/base_sensor.py", "code/sensor_pack_2/base_sensor.py"],
    ["sensor_pack_2/bus_service.py", "code/sensor_pack_2/bus_service.py"]
    ```
    In this case, do not write this sub-package into `deps`.
  - Option **③**: Write into `deps`: `["github:FreakStudioCN/{subdirectory_name}", "main"]`, marked ⚠️
- **curl execution fails**: Prompt the user to visit `https://upypi.net/api/search?q={subdirectory_name}` in a browser and paste the JSON result; if inaccessible, treat as "no results" and present the three options to the developer

### When no sub-package directories exist

Before outputting `package.json`, ask the developer:
```
No sub-package dependency directories detected in the current directory.
If the driver's utility modules (e.g., bus_service, base_sensor, etc.) need to be reused by other drivers in the future,
consider organizing them into a separate Python package and publishing it to upypi?
(You can skip this for now and continue generating package.json)
```

## Dependency Handling (Three-Step Priority)

### Step 1: Identify import source

```
MicroPython built-in modules (machine, time, sys, utime, uos, ustruct, etc.)
→ Do not write into deps, skip directly

micropython-lib standard library (collections, os, json, re, hashlib, etc.)
→ Use mip standard format: ["library_name", "latest"]

Other third-party modules (not the two categories above)
→ Proceed to Step 2 to query upypi
```

### Step 2: Query upypi

For each third-party dependency, **prefer using Bash tools to execute curl for automatic query**:

```bash
curl -s "https://upypi.net/api/search?q={dependency_module_name}"
```

Example response:
```json
{"query":"ds18b20","results":[{"name":"ds18b20_driver","url":"https://upypi.net/pkgs/ds18b20_driver/1.0.0"}]}
```

- **Has results**: Use the returned `url` field to write into deps: `["{url}", "latest"]`
- **curl execution fails (no network/curl unavailable)**: Prompt the user to visit `https://upypi.net/api/search?q={module_name}` in a browser and paste the JSON result back; continue processing after receiving the result; if the user cannot access it, use the `github:` placeholder format and mark `⚠️ needs manual confirmation`
- **No results**: Write using the `github:` placeholder format and mark `⚠️ needs manual confirmation` at the end of the file

### Step 3: deps field format

```json
"deps": [
  ["https://upypi.net/pkgs/ds18b20_driver/1.0.0", "latest"],
  ["collections-defaultdict", "latest"],
  ["github:org/repo", "main"]
]
```

If there are no external dependencies, omit the `deps` field.

## License and Copyright Rules

| Situation | author field | license field |
|---|---|---|
| Referencing someone else's open-source code | Same as the original repository author | Same as the original repository license |
| FreakStudio original | `"leeqingshui"` or team name | `"MIT"` |

**Example of referencing someone else's code** (e.g., referencing robert-hh's bmp280 driver):
```json
{
  "name": "bmp280_driver",
  "urls": [["bmp280_float.py", "code/bmp280_float.py"]],
  "version": "1.0.0",
  "_comments": {
    "chips": "Chip models supported by this package, all means no chip restrictions",
    "fw": "Specific firmware this package depends on, e.g., ulab, lvgl; all means no firmware dependency"
  },
  "description": "A MicroPython library to control BMP280 pressure sensor",
  "author": "robert-hh",
  "license": "MIT",
  "chips": "all",
  "fw": "all"
}
```
> The `author` and `license` fields must be consistent with the original repository; do not fill in FreakStudio information.

## Output Template

```json
{
  "name": "sensor_driver",
  "urls": [
    ["sensor.py", "code/sensor.py"]
  ],
  "version": "1.0.0",
  "_comments": {
    "chips": "Chip models supported by this package, all means no chip restrictions",
    "fw": "Specific firmware this package depends on, e.g., ulab, lvgl; all means no firmware dependency"
  },
  "description": "A MicroPython library to control [sensor name]",
  "author": "author_name",
  "license": "MIT",
  "chips": "all",
  "fw": "all"
}
```

When there are dependencies, append the `deps` field (placed before `urls`):
```json
{
  "name": "xfyun_asr",
  "version": "1.0.1",
  "description": "iFlytek online ASR WebSocket driver for MicroPython",
  "author": "leeqingsui",
  "license": "MIT",
  "chips": "all",
  "fw": "all",
  "_comments": {
    "chips": "Chip models supported by this package, all means no chip restrictions",
    "fw": "Specific firmware this package depends on, e.g., ulab, lvgl; all means no firmware dependency"
  },
  "deps": [
    ["https://upypi.net/pkgs/async_websocket_client/1.0.0", "latest"]
  ],
  "urls": [
    ["xfyun_asr.py", "code/xfyun_asr.py"]
  ]
}
```

## Three Installation Methods (Inform the user after generating package.json)

After generation is complete, attach the three installation commands corresponding to this package for the user's reference:

```python
# Method 1: mip (run on the device)
import mip
mip.install("github:FreakStudioCN/GraftSense-Drivers-MicroPython/sensors/{package_directory_name}")

# Method 2: mpremote (command line)
# mpremote mip install github:FreakStudioCN/GraftSense-Drivers-MicroPython/sensors/{package_directory_name}

# Method 3: upypi (recommended, visit https://upypi.net/ to search for the package name and get the command)
```

## Output Format

1. Output the complete `package.json` content (JSON code block preview).
2. Ask the user: "Confirm writing to `package.json` in the same directory?", and write the content to the file after user confirmation.

If there are dependencies with no results from the upypi query, list them separately after the code block:
```
⚠️ The following dependencies were not found on upypi and have been written using placeholder format. Please confirm manually:
- {module_name}: github:org/repo
```

Finally, attach the three installation method commands (replace `{package_directory_name}` with the actual directory name).


## Complete Specification Reference

This Skill's rewriting rules are based on the GraftSense driver writing specification document. For the complete specification (22 chapters, 2200+ lines), please refer to:

[Complete Specification Document](https://github.com/FreakStudioCN/MicroPython_Skills/blob/main/upy_driver_dev_spec_summary.md)

## Introspection and Evolution

After each execution, check if the following situations are encountered:
- Edge cases not covered by the rules
- Output errors or rule defects pointed out by the user
- Newly discovered constraint requirements

If so, immediately execute:
1. Append the new rule to the corresponding section of this file
2. Write the same modification synchronously to `G:/MicroPython_Skills/upy-gen-pkg/SKILL.md`
3. Execute in the `G:/MicroPython_Skills/` directory:
   `git add upy-gen-pkg/SKILL.md && git commit -m "skill(upy-gen-pkg): <rule_description>"`
