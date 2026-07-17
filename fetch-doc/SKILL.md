---
name: fetch-doc
description: Use this skill when the user provides a URL and wants to extract key information from it. Supports GitHub files, upypi pages, and general web pages. Invoke when user says things like "帮我看一下这个链接", "从这个URL提取信息", "这个页面说了什么", "读取这个文档", or pastes any URL and asks about its content.
---

# Document Fetching and Key Information Extraction Skill

## Role Definition

Given any URL, automatically fetch the content and extract key information. Supports GitHub files, upypi package pages, general web pages, etc.

## Script Path

```
{skill_dir}/scripts/fetch_github.py
```

Usage:
```bash
# Fetch text content
python {skill_dir}/scripts/fetch_github.py "{url}"

# Download images
python {skill_dir}/scripts/fetch_github.py --image "{url}" "{save_dir}"
```

## Execution Steps

### Step 1: Identify URL Type

| URL Pattern | Handling Method |
|---|---|
| `github.com/.../blob/...` | Automatically convert to raw URL, fetch via script |
| `raw.githubusercontent.com/...` | Fetch directly via script |
| `upypi.net/pkgs/...` | Fetch JSON via curl |
| Images (.png/.jpg/.gif) | Download locally using `--image` parameter |
| Other web pages | Fetch HTML via curl or script, extract body text |

### Step 2: Fetch Content

Prefer using Bash tools to invoke the script:
```bash
python "C:/Users/Administrator/.claude/skills/fetch-doc/scripts/fetch_github.py" "{url}" 2>/dev/null
```

GitHub URLs are automatically converted; SSL certificate issues are automatically skipped.

### Step 3: Extract Key Information

Extract based on content type:

| Content Type | Extraction Focus |
|---|---|
| README.md | Introduction, feature list, quick start code, notes |
| Driver .py | Class name, constructor parameters, public API table |
| package.json | Package name, version, dependencies, file list |
| General documents | Title structure, core paragraphs, code blocks |

### Step 4: Output

Output is guided by the user's question, with no fixed format. If the user just wants to "take a look", output a summary; if the user asks a specific question, answer directly.
