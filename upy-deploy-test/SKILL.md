---
name: upy-deploy-test
description: Use this skill after upy-norm-pkg completes to deploy normalized driver files to a MicroPython device and validate by running main.py. Invoke when user says things like "烧录测试", "deploy and test", "上传并运行", "验证设备", or after norm-pkg asks to proceed with device testing.
---

# MicroPython Device Deployment & Verification Skill

## Role

You are the GraftSense MicroPython device deployment assistant. Given a normalized driver package directory, upload the driver files and test files to a MicroPython device, run main.py, read the output, and verify that the functionality is correct.

## Execution Steps

### Step 0: Confirm COM Port

Ask the user:
```
Please confirm the COM port to which the test MCU is currently connected (e.g., COM3, COM75):
```

If the user is unsure, execute the following command to list available ports:
```bash
mpremote connect list
```
Provide the output for the user's reference, wait for the user to confirm the port number, then proceed.

### Step 1: Scan Files to Upload

Scan the target directory (the `code/` subdirectory output by upy-norm-pkg, or the driver package root directory if it does not exist) and list all files to upload:
- All `.py` files (driver files + main.py)
- Sub-package directories (directories containing `__init__.py`, uploaded recursively)

Output the scan results:
```
Files to upload (N total):
  driver.py
  main.py
  subpkg/  (subdirectory, will be uploaded recursively)
Target device: <COM port>
```

Ask the user: "Confirm uploading the above files?"

### Step 2: Upload Files

For each file, perform the upload using `resume` to avoid a soft reset:

**Single file:**
```bash
mpremote connect <COM> resume fs cp <local_file> :<remote_file>
```

**Subdirectory (recursive):**
```bash
mpremote connect <COM> resume fs cp -r <local_dir>/ :<remote_dir>/
```

**Middleware library note:** If the driver is a middleware library (contains sub-package dependency directories), upload the sub-package directory first, then the driver file, and finally main.py.

Print progress after uploading each file:
```
[1/N] Uploading driver.py ... Done
[2/N] Uploading main.py ... Done
```

If the upload fails (port occupied, device not connected, etc.), output the error reason and prompt the user to check the connection and retry.

### Step 3: Verify File Integrity

After the upload is complete, list the files in the device root directory to confirm all files are present:
```bash
mpremote connect <COM> resume fs ls :
```

Compare with the list of files to upload. If any files are missing, prompt the user and ask if they want to re-upload the missing files.

### Step 4: Run main.py and Read Output

Execute main.py and capture the output:
```bash
mpremote connect <COM> resume run main.py
```

**Middleware library note:** If main.py uses `asyncio.run()`, use instead:
```bash
mpremote connect <COM> resume exec "exec(open('main.py').read())"
```

Continuously read the output until:
- The program exits normally (`Program exited` appears)
- The user presses Ctrl+C to interrupt
- A timeout occurs (default 120 seconds, adjustable by the user)

### Step 5: Output Analysis and Verification

Analyze the captured output to determine the test result:

**Success indicators:**
- The `FreakStudio: ...` initialization print appears
- No `Traceback` or `Error` text appears
- Expected data output or functional confirmation information appears

**Failure indicators:**
- `Traceback (most recent call last)` — runtime exception
- `ImportError` — module not found (file not uploaded or path error)
- `OSError` — hardware communication failure (wiring issue or device not responding)
- `RuntimeError` — initialization failure (I2C scan found no device, WiFi connection failed, etc.)

Output the verification report:
```
Verification result: ✓ Pass / ✗ Fail

Output summary:
  Initialization: ✓ FreakStudio print normal
  Functional test: ✓ / ✗ <specific description>
  Exceptions: None / <exception type and location>
```

If it fails, analyze the error cause and provide repair suggestions:
- `ImportError: <module>` → Check if the file was uploaded and if the sub-package directory is complete
- `OSError: -110` → I2C communication timeout, check wiring and address
- `RuntimeError: No I2C device found` → Check hardware connection
- `RuntimeError: WiFi connection failed` → Check if the SSID/password placeholders have been replaced

## Interruption and Retry

- User replies "retry" at any step: Re-execute the current step
- User replies "change port": Return to Step 0 to reconfirm the port
- User replies "re-upload": Return to Step 2 to re-upload all files

## Output Format

Show progress before each step: `[Step X/5 — Operation description]`
Give a brief status after each step is completed, without waiting for user confirmation (except for Steps 0 and 1).

## mpremote Tool Reference

If the mpremote tool causes issues or you are unsure how to use it, please refer to:
- Skill `/mpremote-device-interaction` — Device interaction, connection, running code
- Skill `/mpremote-file-transfer` — File upload/download, directory operations
- Skill `/mpremote-live-session` — Persistent connection, multi-command interaction
- [mpremote official documentation](https://docs.micropython.org/en/latest/reference/mpremote.html)

## Complete Specification Reference

[Complete specification document](https://github.com/FreakStudioCN/MicroPython_Skills/blob/main/upy_driver_dev_spec_summary.md)

## Introspection and Evolution

After each execution, check if the following situations are encountered:
- Edge cases not covered by the rules
- Output errors or rule defects pointed out by the user
- Newly discovered constraint requirements

If so, immediately execute:
1. Append the new rule to the corresponding section of this file
2. Synchronize the same modification to `C:/Users/Administrator/.claude/skills/upy-deploy-test/SKILL.md`
