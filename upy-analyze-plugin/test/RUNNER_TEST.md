# `upy-analyze-plugin` Runner Walkthrough Guide

## Goal

Use the minimal `analyze_runner.py` to:

- sample input
- `mock_plugin.py`
- `scripts/init_manifest.py`

Chain them into a locally walkable analyze protocol chain.

## Current Positioning

`analyze_runner.py` is NOT a full server-side analyze implementation.

It currently handles:

1. Read `sample/start_phase.analyze.json`
2. Output minimal `status_update`
3. Output `approval_request(device_confirm)`
4. Receive `approval_response`
5. Output `approval_request(requirement_supplement)` as needed
6. Output driver search progress
7. Conditionally trigger `approval_request(alternative_device)`
8. Output `script_run(init_manifest.py)`
9. Receive `script_result`
10. Complete local manifest validation
11. Output `phase_complete`

The focus is on getting the protocol chain working first.

## Current Driver Search Display Rules

The runner now displays according to the new rules:

- `builtin_runtime` indicates underlying runtime/peripheral capability is available
- For specific `I2C / SPI / UART` devices, an additional hint "should still prioritize checking upypi" is shown
- `micropython_lib` indicates official ecosystem common libraries/middleware
- `upypi / awesome-micropython / github` indicates specific device driver sources

## Current Walkthrough Method

Correct way:

```text
python test/run_local_mock_session.py
```

Full smoke check:

```text
python test/smoke_tests.py
```

Do NOT use directly:

```text
python test/analyze_runner.py | python test/mock_plugin.py
```

Because this is a unidirectional pipe, not a bidirectional protocol bridge.

## Current Walkthrough Scope

The runner currently covers:

- Happy path
- Device confirmation card
- Requirement supplement
- Driver search progress
- Alternative device basic branch
- Cold-driver basic branch
- Manifest validation
- Successful completion

Not yet fully covered:

- Real `upy-pkg-guide` skill invocation and network queries
- Complex re-analysis paths after user provides supplementary information
- More complex failure recovery paths
- Deep board consumption and selection rule participation

## How to Understand It

The most valuable aspect right now is NOT "it's already a complete analyze service", but rather:

- The protocol message order is already established
- The mock plugin integration point is already in place
- The manifest validation step is already chained in
- The alternative and cold-driver entry branches can already be walked through

In other words, `upy-analyze-plugin` has moved from "static documentation" to a "walkable structure".
