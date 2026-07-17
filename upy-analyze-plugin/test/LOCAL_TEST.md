# `upy-analyze-plugin` Local Mock Test Guide

## Purpose

Verify that the analyze protocol chain runs smoothly without a real plugin host.

Current verification scope:

- `approval_request`
- `status_update`
- `script_run`
- `phase_complete`

## Current Files

- `SKILL.md`
- `scripts/init_manifest.py`
- `mock_plugin.py`
- `analyze_runner.py`
- `sample/*.json`

## Current Driver Search Criteria

The local mock exercise already follows the new rules:

- First distinguish between `builtin_runtime` and "specific device drivers"
- Specific device drivers simulate `upy-pkg-guide` results via `pkg_guide_adapter`
- The real server flow should call `upy-pkg-guide`; the local adapter is only used for deterministic mock exercises
- `micropython_lib` is mainly used for official ecosystem general-purpose libraries/middleware such as `aioble`
- For broad device categories like "soil sensors", first split into implementation families, e.g., `ADC` / `Modbus` / `I2C`

## Current Behavior of mock_plugin.py

### approval_request

- `device_confirm`
  - Automatically returns `confirm`
  - Automatically selects the default option
- `requirement_supplement`
  - Automatically returns `confirm`
  - Automatically keeps the default option
- `alternative_device`
  - Automatically returns `accept_alt1`

### status_update

- Prints to `stderr` for easy timeline observation

### script_run

- Currently only supports `python`
- Actually executes the script
- Wraps `stdout/stderr` into `script_result`

### phase_complete

- Prints the result and `summary`

## Recommended Test Scenarios

### Overall Smoke Test

```text
python test/smoke_tests.py
```

Covers:

- Test module imports
- `sample/*.json` format
- `phase_complete.manifest_content` can be validated by `scripts/init_manifest.py`
- Runner/mock bridge reaches `phase_complete`

### Scenario A: Happy Path

Goal:

- Complete flow through:
  - Intent decomposition
  - Device confirmation
  - Requirement supplement
  - Driver search
  - Manifest validation
  - `phase_complete(success)`

### Scenario B: System-Recommended Device Has No Driver

Goal:

- Trigger `alternative_device`
- Mock automatically selects `accept_alt1`
- The updated devices list should reflect the alternative result

### Scenario C: User-Specified Device Has No Driver

Goal:

- Do not trigger `alternative_device`
- Directly mark `cold-driver` in the manifest

### Scenario D: Soil-Type Devices Split by Implementation Family

Goal:

- Input soil-related requirements
- Distinguish between `ADC` / `RS485/Modbus` based on the description
- Specific device drivers are simulated by the adapter from `upy-pkg-guide` query results

### Scenario E: Manifest Validation Fails

Goal:

- `scripts/init_manifest.py` returns `status=fail`
- Analyze should not output an erroneous success result

## Current Limitations

`mock_plugin.py` is not a full plugin substitute; it is only a minimal protocol tester.

Future additions may include:

- Re-analysis after user-supplemented information
- More complete script execution examples
- More complex failure recovery paths
