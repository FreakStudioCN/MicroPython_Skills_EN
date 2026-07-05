---
name: upy-analyze-plugin
description: Plugin-based workflow version of analyze. Reads user natural language and plugin context to perform requirement parsing, device confirmation, driver search, alternative recommendation or cold-driver marking, and delivers results downstream via phase_complete + manifest_content. Trigger: plugin start_phase(analyze).
---

# Plugin-based Workflow Requirement Parsing and Driver Search Skill

## Role Definition

This is the plugin-based workflow version of `upy-analyze`.

The goal is not to continue the old form of "local multi-turn Q&A + direct disk write", but to change it to:

```text
User natural language + plugin context
-> Intent decomposition
-> Device confirmation
-> Driver search by workflow
-> Alternative recommendation or cold-driver marking
-> Output manifest_content
-> phase_complete(next_phase=select-hw, next_skill=/upy-select-hw-plugin)
```

This skill does not replace the original `G:\MicroPython_Skills\upy-analyze`; it is used for independent evolution of the plugin-based workflow.

## Hard Constraints

- Input is fixed to plugin context fields; no longer asks "beginner/custom" first
- The main flow retains only 1 primary confirmation point: the device confirmation card
- `custom` mode allows at most 1 supplementary card
- `beginner` mode, if supplementary scene/power/performance/output is needed, must also converge into 1 supplementary card
- `system_recommended` with no driver allows alternative recommendation, at most 2 candidates
- `user_specified` with no driver does not automatically offer alternatives; directly mark the cold-driver path
- analyze is only responsible for cold-driver tagging, not for generating drivers within this phase
- The completion criterion for analyze is `phase_complete`, not local manifest disk write
- The standard handover artifact for downstream phases is `manifest_content`
- `next_phase` is currently fixed to `select-hw`
- `next_skill` is currently fixed to `/upy-select-hw-plugin`
- `next_phase` represents the workflow stage name and cannot be changed to a plugin name; the entry point is represented by `next_skill`
- In the Claude Code direct-test mode without a real plugin host, additional debug artifacts are allowed, but these files are only evidence of direct testing and do not replace `phase_complete.manifest_content`
- At any confirmation point, do not automatically proceed to the next step until explicit user confirmation is received
- The model is not allowed to default-click "confirm" on behalf of the user
- The model is not allowed to both "display a confirmation card" and "assume the user has confirmed and continue execution" in the same reply

## Workflow Goal

The goal of this skill is not to conduct multi-turn free-form dialogue, but to produce a stable entry-stage result:

```text
Input context
-> Intent decomposition
-> Device confirmation
-> Optional requirement supplement
-> Driver search
-> Alternative recommendation or cold-driver marking
-> Manifest validation
-> phase_complete(next_phase=select-hw, next_skill=/upy-select-hw-plugin)
```

## Input Contract

This skill only accepts the following input fields:

- `user_description`
- `pre_selected_board`
- `preferences.mode`
- `preferences.locale`
- `existing_hardware`

### Input Explanation

- `user_description`
  - The user's natural language requirement description; this is the primary input for this phase
- `pre_selected_board`
  - Can be empty
  - When empty, only record "no board selected"; do not make a final selection within analyze
- `preferences.mode`
  - Affects whether a supplementary card is needed
- `preferences.locale`
  - Affects the text of subsequent cards and results
- `existing_hardware`
  - Only serves as supplementary information for the device list; do not perform complex deductions within analyze

### Input Missing Handling

- If `user_description` is missing or empty:
  - Stop immediately
  - Output a structured error; do not proceed to subsequent steps
- If `preferences` is missing:
  - Use default values:
    - `mode = "beginner"`
    - `locale = "zh"`
- If `pre_selected_board` is missing:
  - Treat as `null`
- If `existing_hardware` is missing:
  - Treat as an empty array

## Execution Steps

### Step 1: Read Plugin Input Context

- Read `user_description`
- Read `pre_selected_board`
- Read `preferences.mode`
- Read `preferences.locale`
- Read `existing_hardware`

Output goal:

- Establish the working context for this analyze session
- Do not ask the user questions at this step
- Immediately prepare the first progress message:
  - `status_update(step_id="intent_extraction", level="info", message="Analyzing requirements...")`

### Step 2: Intent Decomposition

- Extract functional description from natural language
- Extract devices explicitly specified by the user
- Supplement system-recommended devices
- Distinguish between `user_specified` / `system_recommended`

Structured artifacts that must be completed:

- `project_name`
- `requirements.description`
- Initial `devices[]`
- For each device:
  - `name`
  - `type`
  - `interface`
  - `source`

Work requirements:

- Do not silently lock the user into a specific model when the device model is unclear
- Models explicitly specified by the user must be retained as `user_specified`
- Behavioral/level/trigger semantics supplemented by the user for a specified device must be retained on that device, not just written into `requirements.description`. For example, "touch button uses TTP223, outputs low level when pressed" should output `devices[].notes`, and be structured as `devices[].behavior.active_level="low"` where possible.
- System-supplemented devices must be marked as `system_recommended`

After completing this step, must output:

- `status_update(step_id="intent_done", level="success", message="Extracted N devices ...")`

### Step 3: Device Confirmation

- Issue a device confirmation card
- The user can confirm, delete, or add devices

This is the only mandatory confirmation point in the main flow.

After confirmation, must obtain:

- Final device list
- List of added devices
- List of removed devices
- Whether the user modified system-recommended devices

If the user provides supplementary information midway:

- Treat this supplement as a trigger for re-analysis
- Retain the current context
- Return to Step 2 for re-decomposition
- Then regenerate the device confirmation card

Do not perform only local string patching.

Protocol goal for this step:

- Issue `approval_request(device_confirm)`
- Plugin returns `approval_response`
- analyze updates the devices list based on the result

Hard stop rule:

- Upon reaching `device_confirm`, must stop and wait for user reply
- Do not proceed to Step 4 or Step 5 until the user has explicitly expressed "confirm / modify / supplement"
- If the current runtime environment does not have a real plugin card UI, must still stop in dialogue form and wait for user input
- Do not automatically treat "user has confirmed device list" just because of `beginner` mode

### Step 4: Optional Supplementary Card

- Only enabled when needed
- beginner/custom modes each allow at most 1 supplementary card
- Do not revert to multi-turn Q&A

Purpose:

- Collect important information that is clearly useful for subsequent phases but missing from the user's original input
- For example:
  - scene
  - power
  - output
  - coarse-grained levels of sample_rate / precision

Requirements:

- Only 1 structured card is allowed
- Do not break it down into multiple command-line prompts
- Allow falling back to default values if the user does not fill them in

Protocol goal for this step:

- Issue 1 supplementary `approval_request`
- Plugin returns `approval_response`
- analyze updates the corresponding fields in requirements

Hard stop rule:

- If `requirement_supplement` is issued, must stop and wait for user selection
- Do not display a supplementary card and simultaneously default to recommended values and continue execution in the same reply
- Only proceed to driver search after the user has explicitly confirmed

### Step 5: Driver Search

- Perform driver search for each device
- Write the `driver` status based on the result
- Specific device driver search must be delegated to the `upy-pkg-guide` skill; analyze must not fabricate upypi package names or installation commands itself
- The local runner/mock environment can use `pkg_guide_adapter` to return fixed test results, but this adapter must simulate the output semantics of `upy-pkg-guide`

Mandatory evidence requirements:

- When `driver.source = "upypi" | "awesome-micropython" | "github"`, must write `driver.search_provider`
- In formal plugin mode, the `driver.search_provider` for specific device drivers must be `upy-pkg-guide`
- The local `test/` mock runner is only allowed to use `pkg_guide_adapter`, and must simultaneously write `driver.search_mode = "mock"` and `driver.mock = true`
- When `driver.source = "none" | "cold-driver"` and the device is not builtin runtime only, must also record `driver.search_provider` and query description, proving that the conclusion was not reached without searching
- When `driver.source = "builtin_runtime"`, must write `driver.search_required = false`, and use `driver.search_provider = "builtin_runtime_classifier"` to indicate this is a classification of built-in capability, not a package search result
- When `driver.source = "micropython_lib"`, must write `driver.search_provider = "micropython_lib_classifier"` or `driver.search_provider = "upy-pkg-guide"`

The `test/` directory is only for local JSON/protocol exercises; the real plugin flow must not treat `pkg_guide_adapter` as a real driver search source.

#### 5A. General Principles of Driver Search

Before performing driver search, Analyze must first distinguish two layers:

1. `builtin_runtime`
2. Specific device driver source

Do not conflate "MCU/firmware already provides underlying peripheral API" with "a specific device driver package has been found" into the same result.

First answer two questions:

1. Does this device's underlying layer depend on MicroPython's built-in peripheral API?
2. Does a ready-made MicroPython driver package exist for this specific device?

If the second question involves a specific device driver package, `upy-pkg-guide` must be called for that device:

```text
for device in confirmed_devices:
  if device is builtin runtime only:
    mark builtin_runtime
  else:
    call upy-pkg-guide(device.name / aliases / chip model)
    normalize result into devices[].driver
```

Responsibility boundary of `upy-pkg-guide`:

- First check `upypi`
- If no result, fallback to `awesome-micropython`
- For available packages, extract `package_name`, `version`, `install_cmd`, `api_ref`, `repo_url`
- `api_ref` should preferably be written as a structured object, e.g., `{"init": "...", "read": "...", "calibration": "..."}`; do not just write an unparseable string. If the source material can only confirm a summary sentence, it can be written into `notes` first; do not pretend it is a complete API.
- If no available MicroPython driver is found, return "no driver", and let analyze decide on alternative recommendation or cold-driver marking

#### 5B. Builtin Runtime Determination

`builtin_runtime` only means:

- The MicroPython firmware already provides the underlying runtime/peripheral API
- The current device can at least perform low-level access based on these built-in APIs

Typical examples:

- GPIO input/output
  - `machine.Pin`
- ADC sampling
  - `machine.ADC`
- I2C / SPI / UART bus access
  - `machine.I2C`
  - `machine.SPI`
  - `machine.UART`
- I2S microphone / amplifier / speaker
  - `machine.I2S`
- WiFi networking
  - `network`
- Bluetooth
  - `bluetooth`
- WS2812 / NeoPixel
  - `neopixel`

These cases should not be reported as "no driver", but marked as:

- `driver.source = "builtin_runtime"`

And it is recommended to supplement:

- `driver.module`
- `driver.notes`

But note:

- `builtin_runtime` does not equal "a ready-made driver package for this specific device has been found"
- For specific devices on `I2C / SPI / UART`, Analyze should still continue to prioritize checking `upypi`

#### 5C. micropython-lib Determination

If the capability does not belong to the firmware's built-in, but belongs to the MicroPython official ecosystem general library/middleware, it should be separately marked as:

- `driver.source = "micropython_lib"`

This source is not "built-in firmware", nor should it be confused with ordinary third-party GitHub libraries.

In the analyze phase, the positioning of `micropython_lib` is:

- Official ecosystem general library
- Middleware
- Protocol/capability extension

Typical examples:

- `aioble`

It is recommended to supplement:

- `driver.package_name`
- `driver.install_cmd`
- `driver.repo_url`

Hard constraints:

- `micropython_lib` is not the default first search source for specific device drivers like temperature/humidity, soil, displays, actuators, etc.
- If a result is essentially a "specific device driver", `upypi` should be considered first

#### 5D. Specific Device Driver Priority

If the target is a "specific device driver", not an "official ecosystem general library/middleware", check external driver sources in the following order:

1. `upypi`
2. `awesome-micropython`
3. `github`
4. Other clearly verifiable MicroPython-compatible sources

Execution requirements:

- analyze does not directly use string concatenation to generate `package_name` / `install_cmd`
- analyze does not directly write rule-inferred results as `driver.source = "upypi"`
- `driver.source = "upypi" | "awesome-micropython" | "github"` must come from the structured result of `upy-pkg-guide` or an equivalent adapter
- Fixed results returned by the local mock adapter must be marked as mock/test data and must not be disguised as real network queries

Hard constraints:

- Do not treat Python `PyPI` as the primary search entry point for MicroPython driver packages
- Prioritize searching the MicroPython official/compatible ecosystem
- If only ordinary Python packages are found, do not directly treat them as available MicroPython drivers
- In the analyze phase, do not write "firmware built-in capabilities" as `local`
- In the analyze phase, `local` is only allowed when there is a very clear existence of local private driver assets
- For capabilities like `machine.*`, `network`, `bluetooth`, `neopixel`, they should be uniformly written as `builtin_runtime`
- If a device essentially depends on built-in capabilities like `machine.ADC`, `machine.Pin`, `machine.I2S`, `network`, `bluetooth`, but is written as `driver.source = "none"`, it should be considered an analyze output error, not an acceptable weak result
- `driver.source = "none"` should only be used in the following two cases:
  - It is indeed not a built-in runtime capability
  - And `upypi / awesome-micropython / github / micropython_lib` have no available ready-made driver

For devices that are "not a single model, but a broad category of implementation schemes", the implementation family must first be decomposed before driver search.

For example, "soil temperature and humidity sensor" could at least be decomposed into:

- `ADC` capacitive soil moisture sensor
- `UART/RS485/Modbus` integrated soil temperature and humidity sensor
- `I2C/SPI` digital soil sensor
- Combination scheme of "soil moisture + independent temperature probe"

Rules:

- When the user has clearly specified the protocol/interface/model, search according to the user-specified family
- When the user has not specified, only generate "system-recommended implementation families"; do not pretend to have locked in a specific model

#### 5E. No Driver and Cold Driver

Only enter the "no ready-made driver/cold driver" judgment when none of the following conditions are met:

- Not `builtin_runtime`
- `upypi` has no available result
- `awesome-micropython / github / other trusted MicroPython driver sources` have no available result
- If it belongs to the official ecosystem general capability library, `micropython_lib` also has no available result

Each device must obtain one of the following results:

1. `driver.source = "builtin_runtime"`
2. `driver.source = "micropython_lib"`
3. `driver.source = "upypi"`
4. `driver.source = "awesome-micropython"`
5. `driver.source = "github"`
6. `driver.source = "none"`
7. `driver.source = "cold-driver"`

The set of `driver.source` values recommended for the current analyze phase is:

- `builtin_runtime`
- `micropython_lib`
- `upypi`
- `awesome-micropython`
- `github`
- `none`
- `cold-driver`

Explanation:

- `local` is not a default regular option in the analyze phase
- If `local` is actually used, it must be clearly explained where the corresponding local private driver assets come from

Must continuously output progress:

- Start of search
- Completion of search for each device
- Hit driver source
- No driver result

Do not silently run the entire search and only give the final conclusion.

Protocol goal for this step:

- When search starts:
  - `status_update(step_id="driver_search", level="info", message="Searching for drivers... (1/N)")`
- After each device is completed, output based on the result:
  - `driver_found`
  - `driver_fallback`
  - `driver_none`
  - `driver_cold`

Supplementary explanation:

- `builtin_runtime` falls under the "supportable" category, but the message should clearly state:
  - e.g., `OK INMP441 -> builtin_runtime (machine.I2S)`
  - or `OK Soil moisture sensor -> builtin_runtime (machine.ADC)`

### Step 6: Diversion

- `system_recommended` with no driver -> alternative recommendation possible
- `user_specified` with no driver -> directly mark cold-driver
- User rejects alternative and insists on original device -> cold-driver marking

#### 6A. Alternative Recommendation Conditions

Alternative recommendation is only allowed when all of the following conditions are met:

- The device source is `system_recommended`
- The current device has no ready-made driver
- An alternative device of the same category, same interface, with an existing driver can be found

Alternative candidates must also be verified through `upy-pkg-guide`:

- First generate candidate chip/module keywords based on the category
- Call `upy-pkg-guide` for each candidate
- Only allow displaying candidates that `upy-pkg-guide` confirms have an available MicroPython driver

Alternative recommendation constraints:

- Give at most 2 candidates
- Do not provide an overly long candidate list
- After the user confirms the alternative, the device list must be updated

#### 6B. Cold Driver Path Conditions

The following cases directly enter the cold path marking:

- `user_specified` with no driver
- `system_recommended` with no driver, user rejects alternative and insists on original device

Analyze only does the following here:

- Mark in the manifest
- Explain in warnings that subsequent cold driver generation and verification will follow

Analyze does not:

- PDF collection
- Arduino collection
- Cold driver generation
- Cold driver verification

Protocol goal for this step:

- When `system_recommended` with no driver:
  - Issue `approval_request(alternative_device)`
  - Plugin returns `approval_response`
  - analyze updates the devices list based on user selection
- When `user_specified` with no driver:
  - Do not issue an alternative recommendation card
  - Directly mark the cold path in the manifest

### Step 7: Output

- Generate manifest draft
- Call validation script to validate manifest
- Output `phase_complete`
- `next_phase` fixed to `select-hw`
- `next_skill` fixed to `/upy-select-hw-plugin`
- If running in Claude Code direct-test mode and the user has provided a project/test directory, additionally write debug artifacts

#### 7A. Manifest Draft Requirements

Must at least include:

- `schema_version`
- `phase = "analyze"`
- `project_name`
- `requirements`
- `devices`

And ensure:

- `requirements` fields are complete
- `devices[].source` is clear
- `devices[].driver.source` is clear

#### 7B. Validation Requirements

The validation script's responsibilities are:

- Validate enum values
- Fill in default values
- Return structured validation results

The validation script is not the completion criterion for analyze.

The completion criterion for analyze is:

- Successfully generate `manifest_content` that can be consumed downstream
- Output `phase_complete`

#### 7C. Handover to Downstream

Analyze must ultimately hand over the following to downstream:

- `manifest_content`
- `warnings`
- `errors`
- `next_phase = "select-hw"`
- `next_skill = "/upy-select-hw-plugin"`

Do not treat "successful local disk write" as the sole source of truth for this phase.

Protocol goal for this step:

- Call `init_manifest.py` for validation
- Read structured validation results
- Generate `phase_complete`

#### 7D. Claude Code Direct-Test Mode Debug Artifacts

When there is no real plugin host and no message bus to persist `phase_complete`, it is difficult for Claude Code direct tests to check results from the file system. Therefore, additional debug artifacts are allowed.

Trigger conditions:

- The current runtime environment is a normal Claude Code conversation/local skill invocation
- The user has provided a project directory, test directory, or the current working directory is clearly a test project directory
- For example, the user is testing this skill under `G:\test\test`

Recommended files to write:

```text
{project_dir}/manifest_draft.json
{project_dir}/manifest_validated.json
{project_dir}/phase_complete.analyze.json
{project_dir}/driver_search_log.md
```

Content to write:

- `manifest_draft.json`
  - Manifest draft before validation
  - Must include `project_name`, `requirements`, `devices`
- `manifest_validated.json`
  - Manifest after validation and normalization by `init_manifest.py`
  - Must be consistent with the final `phase_complete.manifest_content`
  - Must include `schema_version`, `phase`, `created_at`, `updated_at`, `final_status`
- `phase_complete.analyze.json`
  - Complete `phase_complete` message payload
  - Must include `manifest_content`
  - Must maintain the protocol shape: `artifacts` is an array, not allowed to be written as a path mapping object like `{ "manifest_draft": "..." }`
  - If debug file paths need to be recorded, use a `file_list` artifact, for example:
    ```json
    {
      "type": "file_list",
      "title": "Claude Code Direct Test Artifacts",
      "files": [
        { "path": "manifest_draft.json", "status": "created" },
        { "path": "manifest_validated.json", "status": "created" },
        { "path": "phase_complete.analyze.json", "status": "created" },
        { "path": "driver_search_log.md", "status": "created" }
      ]
    }
    ```
- `driver_search_log.md`
  - Search keywords for each device, results of calling `upy-pkg-guide`, final `driver.source`

Constraints:

- These files are only for Claude Code direct testing and manual troubleshooting
- In formal plugin mode, `phase_complete.manifest_content` remains the sole phase handover artifact
- Do not judge analyze as successful or failed due to failure to write debug files; the phase result is still determined by manifest validation and `phase_complete`
- Before writing, explain that these are direct-test debug artifacts, not final project code
- After writing `phase_complete.analyze.json`, it must be validated using the validation script's phase_complete mode:
  ```bash
  python {skill_dir}/scripts/init_manifest.py --validate-phase-complete --input {project_dir}/phase_complete.analyze.json
  ```
  If validation fails, `phase_complete.analyze.json` must be corrected; do not declare the analyze phase successful.

## Standard Message Sequence

The current analyze plugin version should follow this message order:

```text
Step 1 Input context establishment
  -> status_update(intent_extraction)

Step 2 Intent decomposition complete
  -> status_update(intent_done)

Step 3 Device confirmation
  -> approval_request(device_confirm)
  -> approval_response

Step 4 Optional supplementary card (as needed)
  -> approval_request(requirement_supplement)
  -> approval_response

Step 5 Driver search
  -> status_update(driver_search)
  -> status_update(driver_found / driver_fallback / driver_none / driver_cold)

Step 6 Alternative recommendation (conditionally triggered)
  -> approval_request(alternative_device)
  -> approval_response

Step 7 Manifest validation
  -> script_run(init_manifest.py)
  -> script_result

Step 8 Phase complete
  -> phase_complete(result=success, next_phase=select-hw, next_skill=/upy-select-hw-plugin)
```

## Message Definition Requirements

### approval_request #1: device_confirm

Purpose:

- Confirm the device list
- Display project name, function summary, board status
- Allow adding/deleting/confirming devices

Requirements:

- This is the only mandatory confirmation point in the analyze main flow
- Must support:
  - `allow_add = true`
  - `allow_remove = true`
  - `multi_select = true`

Suggested payload example:

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "device_confirm",
    "header": "Confirm Project Plan",
    "question": "Please confirm if the following devices are correct",
    "summary": {
      "project_name": "Plant Assistant",
      "description": "Reads soil temperature and humidity, supports touch interaction and voice dialogue",
      "board": {
        "status": "selected",
        "display_name": "ESP32-S3-DevKitC-1",
        "mcu": "ESP32-S3-WROOM-1"
      }
    },
    "items": [
      {
        "id": "d1",
        "name": "Soil Moisture Sensor",
        "subtitle": "ADC Soil Moisture Sensor",
        "meta": "System Recommended",
        "selected": true
      },
      {
        "id": "d2",
        "name": "I2S Microphone",
        "subtitle": "I2S Voice Input",
        "meta": "System Recommended",
        "selected": true
      }
    ],
    "allow_add": true,
    "allow_remove": true,
    "multi_select": true,
    "actions": [
      {
        "label": "Confirm, Start Driver Search",
        "value": "confirm",
        "primary": true
      },
      {
        "label": "Modify Device List",
        "value": "modify"
      }
    ]
  }
}
```

Field constraints:

- `summary.project_name` is required
- `summary.description` is required
- `summary.board.status` can only be:
  - `selected`
  - `none`
- `items[].id/name/subtitle/meta/selected` are required
- `actions` must contain at least 1 primary action
- If `board.status = "none"`, then `display_name/mcu` are not required

In a dialogue environment without a real UI, after displaying this card, must adhere to:

- The reply ends here
- Wait for user input
- Do not output "Confirmed, starting driver search" below the card
- If the questioning tool of the runtime environment has a "limit on the number of options", first compress grouped options or change to a text confirmation format, then stop and wait for user reply; do not skip the confirmation point due to tool limitations

### approval_request #2: requirement_supplement

Purpose:

- Supplement important but missing requirements fields in beginner/custom modes

Requirements:

- At most 1 card is allowed
- Do not break it down into multiple rounds

Suggested payload example:

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "requirement_supplement",
    "header": "Supplement Requirement Information",
    "question": "Please supplement scene, power, performance, and output requirements",
    "summary": {
      "project_name": "Plant Assistant"
    },
    "items": [
      {
        "id": "scene_indoor",
        "name": "Indoor Desktop Scene",
        "subtitle": "Default Recommendation",
        "meta": "scene=indoor",
        "selected": true
      },
      {
        "id": "power_usb",
        "name": "USB Power",
        "subtitle": "Default Recommendation",
        "meta": "power=usb",
        "selected": true
      },
      {
        "id": "perf_normal",
        "name": "General Performance",
        "subtitle": "1Hz / Normal Precision / 1-second Response",
        "meta": "sample_rate=normal_1hz",
        "selected": true
      },
      {
        "id": "output_serial_oled",
        "name": "Serial + OLED Output",
        "subtitle": "Default Recommendation",
        "meta": "output=serial,display_oled,buzzer",
        "selected": true
      }
    ],
    "allow_add": false,
    "allow_remove": false,
    "multi_select": true,
    "actions": [
      {
        "label": "Confirm Supplementary Information",
        "value": "confirm",
        "primary": true
      }
    ]
  }
}
```

### approval_request #3: alternative_device

Purpose:

- When a `system_recommended` device has no driver, provide at most 2 alternative devices

Suggested payload example:

```json
{
  "type": "approval_request",
  "payload": {
    "approval_id": "alternative_device",
    "header": "Sensor: Recommended Alternative Devices",
    "question": "The current device has no ready-made driver. The following alternative devices are recommended",
    "items": [
      {
        "id": "alt1",
        "name": "HDC1080",
        "subtitle": "Higher precision, has upypi driver",
        "meta": "Recommended",
        "selected": false
      },
      {
        "id": "alt2",
        "name": "AHT20",
        "subtitle": "Lower cost, has upypi driver",
        "meta": "Alternative",
        "selected": false
      }
    ],
    "allow_add": false,
    "allow_remove": false,
    "multi_select": false,
    "actions": [
      {
        "label": "Use HDC1080 (Recommended)",
        "value": "accept_alt1",
        "primary": true
      },
      {
        "label": "Use AHT20",
        "value": "accept_alt2"
      },
      {
        "label": "Keep Original Device, Use Cold Driver",
        "value": "cold_driver"
      }
    ]
  }
}
```

### status_update

Analyze should currently define at least the following progress messages:

- `intent_extraction`
- `intent_done`
- `driver_search`
- `driver_found`
- `driver_fallback`
- `driver_none`
- `driver_cold`

### phase_complete

Analyze's `phase_complete` must at least include:

- `phase = "analyze"`
- `result`
- `summary`
- `next_phase = "select-hw"`
- `next_skill = "/upy-select-hw-plugin"`
- `manifest_content`
- `artifacts`: Must be an array. Debug file paths are expressed using `file_list` artifacts; object mapping is not allowed to replace the array.
- `warnings`
- `errors`

## Dialogue Environment Specific Constraints

If analyze is running in an environment without a real plugin card host, such as a normal chat-style skill invocation, the following rules must be followed:

1. After `approval_request(device_confirm)` appears, the reply must end, waiting for user input
2. After `approval_request(requirement_supplement)` appears, the reply must end, waiting for user input
3. After `approval_request(alternative_device)` appears, the reply must end, waiting for user input
4. Do not automatically advance to the following without user reply:
   - Driver search
   - Manifest validation
   - phase_complete
5. If the user's reply is "modify device" or "supplement requirement", re-analyze based on the user's new input; do not pretend it is still the old list

One-sentence requirement:

**Stop first, then wait for the user; without user confirmation, do not proceed.**

## manifest_content Minimum Delivery Requirements

The `manifest_content` that Analyze hands over to the downstream `select-hw` must at least include:

- `schema_version`
- `phase = "analyze"`
- `project_name`
- `requirements.description`
- `requirements.experience`
- `requirements.output`
- `requirements.existing_hardware`
- `requirements.mcu_specified`
- `devices`

Where each `devices[]` must at least include:

- `name`
- `type`
- `interface`
- `source`
- `driver.source`

Optional but should be retained device-level fields:

- `notes`: The user's natural language supplement for the device, such as module model, trigger method, level semantics, installation method
- `behavior`: Structurable behavioral facts, such as `role`, `event`, `active_level`, `idle_level`

When the user clearly describes device behavior, write both `notes` and `behavior` preferentially. For example, the TTP223 touch button "outputs low level when pressed" should be retained as a device-level fact for select-hw and generate to determine GPIO input, pull-up/pull-down, and trigger conditions.

If `driver.source` belongs to a ready-made driver source, subsequently continue to complete:

- `package_name`
- `install_cmd`
- `version`
- `api_ref`: Preferably an object; string form is only acceptable as a temporary weak result and should be exposed in validation warnings

`driver` should also retain search evidence fields for analyze validation and troubleshooting; downstream `select-hw` can ignore these fields:

- `search_provider`: `upy-pkg-guide`, `pkg_guide_adapter`, `builtin_runtime_classifier`, or `micropython_lib_classifier`
- `search_mode`: `real` or `mock`
- `mock`: Only allowed to be `true` for local test adapter results
- `search_required`: Should be `false` for builtin runtime only
- `query`: Keywords passed to `upy-pkg-guide` or adapter
- `evidence`: Structured source evidence, such as package name, repository URL, result description

## boards Asset Constraints

`upy-analyze-plugin` should carry its own independent `boards/` directory assets.

Purpose:

- Accept `pre_selected_board`
- Provide a board list for the local interactive simulator
- Serve as the basic data source for the plugin-side board selector

Principle:

- Do not overwrite the original skill's `boards`
- The plugin version independently maintains its own board data copy

## Local Simulation Test Entry Points

Currently available local test entry points include:

- `python run_local_mock_session.py`
  - Bidirectional bridge between runner and mock plugin
  - Suitable for verifying the minimal happy path
- `python interactive_local_session.py`
  - Terminal interactive simulation of user input
  - Can input requirements, select mode, select board, modify requirements, confirm devices

## Current Status

This is the first version of the plugin-based analyze workflow that can be exercised, and it currently has:

- Plugin input boundary
- Main confirmation card
- Optional supplementary card
- Alternative recommendation / cold driver diversion
- `script_run(init_manifest.py)` validation chain
- `phase_complete(next_phase=select-hw, next_skill=/upy-select-hw-plugin)` handover chain

Subsequent enhancement focus is no longer on "whether there is a workflow", but on supplementing:

- Stricter protocol constraints
- More complete board assets
- Stronger manifest validation
- A more realistic analyze engine
## Session Boundary Addendum

- Treat an explicit user-supplied session path as the `workflow_session_root` unless the user explicitly says it is only a diagnostic/log source.
- A `diagnostic_log_session` is evidence only. Do not move `manifest_content`, project files, phase_complete files, or follow-up artifacts into it.
- If a user provides both a target workflow session and another session containing logs, write outputs under `workflow_session_root` and reference the other path only in `artifacts`, `warnings`, or downstream `error_context`.
- Do not infer the active workflow session from the newest log file, reopened chat, or prior conversation memory. Use `start_phase.payload.runtime_context.session_root`, `source_phase_complete_path`, or the explicit command argument.
- Final `phase_complete.payload.runtime_context.session_root` must point to the workflow session that owns the artifact chain.
