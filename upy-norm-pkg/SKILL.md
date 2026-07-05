---
name: upy-norm-pkg
description: Use this skill when the user wants to normalize/standardize an existing validated MicroPython driver package (one or more driver .py files, optional main.py) according to the GraftSense coding spec. Invoke when user says things like "规范化驱动包", "norm pkg", "对整个驱动目录规范化", or provides a driver directory path and asks for full normalization.
---

# MicroPython Driver Package Full Normalization Skill

## Role Definition

You are the GraftSense MicroPython driver package normalization assistant. Given a directory containing validated driver files, normalize all files according to a fixed workflow, generate any missing companion files, and output the standard driver package directory structure.

## Type Determination (Execute immediately after Step 0 scan; all subsequent steps follow the corresponding branch based on type)

| Condition | Type |
|---|---|
| Driver located in `middleware/` subdirectory, or imports `network`/`urequests`/`AsyncWebsocketClient`/`asyncio` with no I2C/SPI/UART hardware bus operations | **Middleware Library** |
| All other cases | **Hardware Driver** |

After determination, output:
```
Package Type: Middleware Library (middleware) / Hardware Driver
Classification Suggestion (middleware): middleware/protocol or middleware/network, etc.
Subsequent steps will use the corresponding rule branch
```

**This Skill acts as the Orchestrator for the following Skills and does not generate content itself; it only calls them in order:**
- `/upy-norm-driver` — Driver file normalization
- `/upy-norm-main` or `/upy-gen-main` — Test file normalization or generation
- `/upy-gen-readme` — README generation
- `/upy-gen-pkg` — package.json generation
- `/upy-pack-driver` — Packaging and directory structure organization

## Execution Steps

### Step 0: Scan Directory

1. Scan all `.py` files and subdirectories under the user-specified directory
2. Classify:
   - **Driver files**: `.py` files in the same directory, excluding `main.py`
   - **Sub-package dependency directories**: Subdirectories containing `__init__.py` (not treated as driver files; will be queried on upypi during gen-pkg step and written to `deps`)
   - **Test file**: `main.py` (if present)
3. Output scan results:
   ```
   Directory: G:/ens160_project/
   Driver files (1): ens160sciosense.py
   Sub-package dependency directories (1): sensor_pack_2/  ← contains __init__.py, will be queried on upypi during gen-pkg step
   Test file: main.py ✓ (exists, will execute norm-main)
   ```
3a. Determine package type:
    Scan driver file imports; if middleware library characteristics are met (see upy-norm-driver type determination rules), output:
    ```
    Package Type: Middleware Library (middleware)
    Classification Suggestion: middleware/protocol or middleware/network, etc.
    Subsequent steps will use middleware library rule branch
    ```
    and pass the type flag (middleware library) when calling each corresponding skill in subsequent steps.
   If no sub-package directories:
   ```
   Directory: G:/bmp280/
   Driver files (2): bmp280_float.py, bmp280_int.py
   Sub-package dependency directories: None
   Test file: main.py ✓ (exists, will execute norm-main)
   ```
   If no `main.py`:
   ```
   Test file: Not found (will execute gen-main, generated based on the first driver file)
   ```
4. If there are more than 1 driver file, list all files and ask:
   ```
   Multiple driver files found. Will execute norm-driver for each file in sequence:
   1. bmp280_float.py
   2. bmp280_int.py
   Confirm to execute all, or select only a specific one?
   ```
5. After user confirmation, proceed to Step 1

### Step 1: norm-driver (per file)

Execute `/upy-norm-driver` for each driver file in sequence:
- After completion, pause and display:
  ```
  [Step 1/5 — norm-driver: bmp280_float.py complete]
  Confirm write and continue to next file? Or need modifications?
  ```
- After user confirms write, if there are more driver files, continue to the next one
- After all driver files are complete, proceed to Step 2

### Step 2: norm-main or gen-main

- **If `main.py` exists**: Execute `/upy-norm-main`
- **If `main.py` does not exist**: Execute `/upy-gen-main` (based on the first driver file in the directory)

After execution, pause:
```
[Step 2/5 — main.py complete]
Confirm write and continue?
```

### Step 3: gen-readme

Execute `/upy-gen-readme` (pass the path of the first driver file in the directory).

After execution, pause:
```
[Step 3/5 — README.md complete]
Confirm write and continue?
```

### Step 4: gen-pkg

Execute `/upy-gen-pkg` (pass the driver directory path).

After execution, pause:
```
[Step 4/5 — package.json complete]
Confirm write and continue?
```

### Step 5: pack-driver

Execute `/upy-pack-driver` (pass the path of the first driver file in the directory).

After execution, output:
```
[Step 5/6 — Packaging complete]
<chip>_driver/
├── code/
│   ├── <chip>.py        ✓
│   ├── main.py          ✓
│   └── <subpkg>/        ✓ (if sub-package exists)
├── package.json         ✓
├── README.md            ✓
└── LICENSE              ✓ (generated)
```

Ask user: "Proceed with device deployment and verification?" After user confirmation, proceed to Step 6.

### Step 6: deploy-test

Execute `/upy-deploy-test` (pass the packaged `code/` directory path).

After execution, output:
```
[Step 6/6 — Device verification complete]
```

## Interruption and Recovery

If the user replies "modify" or "redo" at any step, re-execute the current step without affecting completed steps.

## Output Format

Before each step begins, display progress: `[Step X/6 — skill name: filename]`
After each step completes, pause and wait for user confirmation before proceeding to the next step.

## Context Control

**After each step where the user confirms writing a file, do not retain the full content of that file in the conversation.** Keep only a one-line summary in the format:
```
Written <filename>, total <N> lines, <brief execution summary>
```
For example: `Written bmp280.py, total 312 lines, P0 fully executed, P2 executed bytearray reuse buffer`

Subsequent steps must not reference or re-expand the content of already written files.

## Full Specification Reference

[Full Specification Document](https://github.com/FreakStudioCN/MicroPython_Skills/blob/main/upy_driver_dev_spec_summary.md)

## Introspection and Evolution

After each execution, check whether the following situations occurred:
- Edge cases not covered by the rules
- Output errors or rule deficiencies pointed out by the user
- Newly discovered constraint requirements

If so, immediately execute:
1. Append the new rule to the corresponding section of this file
2. Synchronize the same modification to `G:/MicroPython_Skills/upy-norm-pkg/SKILL.md`
3. Execute in the `G:/MicroPython_Skills/` directory:
   `git add upy-norm-pkg/SKILL.md && git commit -m "skill(upy-norm-pkg): <rule description>"`
