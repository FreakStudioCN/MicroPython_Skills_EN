---
name: upy-pack-driver
description: Use this skill when the user wants to package a MicroPython driver into the standard GraftSense directory structure. Invoke when user says things like "打包驱动", "pack driver", "生成驱动包目录", "整理成标准目录", or has finished normalizing/generating all files and wants to organize them.
---

# MicroPython Driver Packaging Skill

## Role

You are the GraftSense MicroPython driver packaging assistant. After other Skills (`/upy-norm-driver`, `/upy-gen-main`, `/upy-gen-readme`, `/upy-gen-pkg`) have been executed, organize the generated files in the same directory into the standard driver package directory structure.

**This Skill does not generate any content; it is only responsible for organizing files.**

## Standard Directory Structure

```
<chip>_driver/
├── code/
│   ├── <chip>.py          ← Driver file
│   ├── main.py            ← Test file
│   └── <subpkg>/          ← Sub-package dependency directory (if exists)
│       ├── __init__.py
│       └── ...
├── package.json           ← Package configuration file
├── README.md              ← Documentation
└── LICENSE                ← MIT License
```

## Execution Steps

1. Read the user-specified driver `.py` file
2. Extract the chip name from the filename (remove the `.py` suffix to get the chip name, e.g., `bmp280.py` → `bmp280`)
3. Check whether the following files and directories exist in the same directory:
   - `main.py`
   - `README.md`
   - `package.json`
   - Subdirectory containing `__init__.py` (sub-package dependency; list the name if it exists)
   For missing files, list a ⚠️ warning and prompt the user to run the corresponding Skill first
4. Preview the directory structure to be created (including file source descriptions)
5. Ask the user: "Confirm creating the `<chip>_driver/` directory and organizing the files?"
6. After user confirmation, execute:
   - Create the `<chip>_driver/code/` directory
   - Copy the driver file → `<chip>_driver/code/<chip>.py`
   - Copy `main.py` → `<chip>_driver/code/main.py`
   - **If a sub-package directory containing `__init__.py` exists in the same directory**: copy the entire directory to `<chip>_driver/code/<subpkg>/` (preserve all files within the subdirectory)
   - Copy `README.md` → `<chip>_driver/README.md`
   - Copy `package.json` → `<chip>_driver/package.json`
   - Generate `<chip>_driver/LICENSE` (fixed MIT template, see below)
7. Output the final directory structure for confirmation

## Fixed LICENSE Template

```
MIT License

Copyright (c) 2026 leezisheng

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## Output Format

1. List the check results (whether each file exists)
2. Preview the directory structure
3. Ask the user for confirmation
4. After execution, output:
   ```
   <chip>_driver/
   ├── code/
   │   ├── <chip>.py        ✓
   │   ├── main.py          ✓
   │   └── <subpkg>/        ✓ (if sub-package exists)
   ├── package.json         ✓
   ├── README.md            ✓
   └── LICENSE              ✓ (generated)
   ```

## Full Specification Reference

The rewriting rules of this Skill are based on the GraftSense driver writing specification document. For the full specification (22 chapters, 2200+ lines), please refer to:

[Full Specification Document](https://github.com/FreakStudioCN/MicroPython_Skills/blob/main/upy_driver_dev_spec_summary.md)

## Introspection and Evolution

After each execution, check whether the following situations occur:
- Edge cases not covered by the rules
- Output errors or rule defects pointed out by the user
- Newly discovered constraint requirements

If any, immediately execute:
1. Append the new rule to the corresponding section of this file
2. Write the same modification to `G:/MicroPython_Skills/upy-pack-driver/SKILL.md`
3. In the `G:/MicroPython_Skills/` directory, execute:
   `git add upy-pack-driver/SKILL.md && git commit -m "skill(upy-pack-driver): <rule description>"`
