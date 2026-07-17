---
name: upy-pkg-guide
description: Use this skill when the user mentions a device/chip name and wants to know how to use its MicroPython driver from upypi. Invoke when user says things like "how to use BMP280", "how to call DS18B20", "tell me how to use MPR121", "check how to use the XX driver on upypi", or mentions any chip/sensor name and asks for usage guidance.
---

# MicroPython Driver Package Usage Extraction Skill

## Role

Given a device name, search for drivers from **upypi → awesome-micropython** in priority order, then output usage guidance after comprehensive analysis.

---

## Execution Steps

### Step 1: Search upypi

```bash
curl -s "https://upypi.net/api/search?q={device_name}"
```

- Results found → proceed to **upypi path** (Step 2A)
- No results → proceed to **awesome-micropython fallback path** (Step 2B)

---

### Step 2A: upypi Path

#### Get package.json

```bash
curl -s "{package_url}/package.json"
```

Extract: `urls` (driver file list), `version`, `author`, `description`, `deps`

#### Download files in parallel

base_url = `https://upypi.net/pkgs/{name}/{version}/`

| File | URL |
|---|---|
| Driver .py (from urls) | `{base_url}{source_path}` |
| main.py | `{base_url}code/main.py` |
| README.md | `{base_url}README.md` |

Skip on 404, no error.

→ Jump to **Step 3: Comprehensive Analysis**

---

### Step 2B: awesome-micropython Fallback Path

Execute this path when upypi returns no results.

#### Run search script

```bash
python "C:/Users/Administrator/.claude/skills/upy-pkg-guide/scripts/search_awesome.py" "{device_name}"
```

Script returns JSON in format:
```json
{
  "query": "spacecan",
  "results": [
    {
      "name": "micropython-spacecan",
      "url": "https://gitlab.com/alphaaomega/micropython-spacecan",
      "desc": "...",
      "category": "Communications",
      "subcategory": "CAN"
    }
  ]
}
```

**Processing logic:**
- `results` empty → inform user the device was not found on upypi or awesome-micropython, end
- Multiple results → list all entries (name + description), ask user to choose
- Single result → use directly

#### Fetch files based on repository platform

Determine platform from `url` field, fetch README.md, main.py, and driver .py:

**GitHub repository:**
```bash
# List repository files
curl -s "https://api.github.com/repos/{owner}/{repo}/contents/"
# Download file
curl -s "https://raw.githubusercontent.com/{owner}/{repo}/master/{path}"
# Get README
curl -s "https://raw.githubusercontent.com/{owner}/{repo}/master/README.md"
# Get main.py
curl -s "https://raw.githubusercontent.com/{owner}/{repo}/master/main.py"
```

**GitLab repository:**
```bash
# List repository files (recursive=true for subdirectories)
curl -s "https://gitlab.com/api/v4/projects/{namespace}%2F{project}/repository/tree?recursive=true"
# Download file
curl -s "https://gitlab.com/{namespace}/{project}/-/raw/master/{path}"
# Get README
curl -s "https://gitlab.com/{namespace}/{project}/-/raw/master/README.md"
# Get main.py
curl -s "https://gitlab.com/{namespace}/{project}/-/raw/master/main.py"
```

Priority downloads: `README.md`, `main.py`, and all `.py` driver files (excluding `main.py` and test files).

→ Proceed to **Step 3: Comprehensive Analysis**

---

### Step 3: Comprehensive Analysis, Output Usage Guidance

Combine all fetched files and output the following structure:

---

## {device_name} Driver Usage Guide

**Source**
- Platform: `upypi` / `awesome-micropython ({category} > {subcategory})`
- Repository: {url}
- Description: {desc}

**Installation**

upypi path:
```bash
mpremote mip install {package_url}/package.json
```

awesome-micropython path (no standard package, manual copy):
```bash
# Copy the following files to the device
mpremote cp {driver_file}.py :{driver_file}.py
# If there is a subpackage directory
mpremote cp -r {pkg_dir}/ :{pkg_dir}/
# If there are dependency subdirectories (e.g., lib/)
mpremote cp -r lib/ :lib/
```

**Initialization**
```python
# Minimal runnable example from main.py
```

**Core API**

| Method/Property | Parameters | Return Value | Description |
|---|---|---|---|
| ... | ... | ... | ... |

**Notes**
- Key limitations, hardware wiring, and compatibility notes extracted from README.md

---

## Output Principles

- `main.py` is the **first priority** reference; directly show its initialization code as the minimal example
- `README.md` supplements notes and hardware wiring
- Driver `.py` is used to extract the complete API table
- Packages from the awesome-micropython path typically lack a standardized installation method; specify which files/directories to copy manually
- If a file does not exist (404 or API returns nothing), skip that section without error

## Script Description

`scripts/search_awesome.py` — awesome-micropython index search script
- Automatically fetches and caches the README of `mcauser/awesome-micropython` (24-hour cache)
- Supports case-insensitive search of library names and descriptions
- Cache file: `scripts/_awesome_cache.json`
- Supported platforms: GitHub, GitLab, Codeberg
