## 2026-07-12 Update: GenDriver / Skill Handover Material Supplement

The original content of this index is the 2026-07-11 material map. The 2026-07-12 update has supplemented the Skill-side GenDriver field decisions, a second review of `upy-gen-driver-plugin`, and new board onboarding materials. The next new session should prioritize reading the documents listed below.

### New Priority Reading Documents

1. `G:\blockless-plugin-course(1)\Skill Engineer GenDriver Field Decision and Erson Confirmation Issues-2026-07-12.md`
   - Use the final sections "Correction: cold driver generation should belong to upy-gen-driver-plugin" and "Second Review" as the authoritative reference.
   - The earlier `upy-generate-plugin` analysis and old Erson wording are for historical reference only and are not the final external stance.
   - It has been confirmed that the main body for cold driver generation should be `G:\MicroPython_Skills\upy-gen-driver-plugin`, not `upy-generate-plugin`.

2. `G:\blockless-plugin-course(1)\Skill Engineer GenDriver Field Standardization To-Do-2026-07-12.md`
   - Skill engineer minimum executable checklist: `select-hw` writes `devices[].driver.status="cold_driver_required"`.
   - Keep `driver.source="cold-driver"` as a source classification; do not use it as a workflow gate.

3. `G:\blockless-plugin-course(1)\GenDriver Cold Driver Status Field and Cloud Scheduling Next Steps-2026-07-12.md`
   - Splits the subsequent processing order for Skill / Backend / Extension / Packaging / GenDriver contract.
   - Used to determine which tasks the Skill side can currently handle and which require confirmation from Erson / cloud/backend.

4. `G:\blockless-plugin-course(1)\Skill Lead Board Onboarding Current Work List-2026-07-12.md`
   - Current execution checklist for the board onboarding direction.

5. `G:\blockless-plugin-course(1)\Board Onboarding Process and Current Code Evidence Analysis-2026-07-12.md`
   - Board onboarding process and local code evidence.

6. `G:\blockless-plugin-course(1)\MicroPython Official Board Verification Material List-2026-07-12.md`
   - List of official MicroPython board materials to be verified.

### Current Final GenDriver Stance

```text
canonical gate field:
  devices[].driver.status

source classification:
  driver.source = "cold-driver"

workflow status:
  driver.status = "cold_driver_required"

Skill-side minimum:
  select-hw must normalize source="cold-driver" into status="cold_driver_required"
  while preserving source.

cold-driver owner:
  upy-gen-driver-plugin owns missing/cold driver generation.

generate behavior:
  upy-generate-plugin should only consume ready/local drivers, or block deploy-ready success
  and emit partial/next_action when cold_driver_required is still unresolved.

cloud/backend behavior:
  before generate, route cold_driver_required to upy-gen-driver-plugin
  and hold generate until a verified ready driver exists.
```

### Updated GenDriver Reading Order

If the next new session only cares about the GenDriver / cold-driver blocker, it is recommended to read in this order:

```text
1. G:\blockless-plugin-course(1)\Local Reference Material Master Index and New Session Handover Notes-2026-07-11.md
2. G:\blockless-plugin-course(1)\GenDriver Cold Driver Status Field and Cloud Scheduling Blocking Explanation-2026-07-11.md
3. G:\blockless-plugin-course(1)\Skill Engineer GenDriver Field Decision and Erson Confirmation Issues-2026-07-12.md
4. G:\blockless-plugin-course(1)\GenDriver Cold Driver Status Field and Cloud Scheduling Next Steps-2026-07-12.md
5. G:\blockless-plugin-course(1)\Skill Engineer GenDriver Field Standardization To-Do-2026-07-12.md
6. G:\MicroPython_Skills\upy-select-hw-plugin\scripts\select_hw_manifest.py
7. G:\MicroPython_Skills\upy-gen-driver-plugin\SKILL.md
8. G:\MicroPython_Skills\upy-gen-driver-plugin\sample\start_phase.upy_gen_driver_plugin.pipeline.json
9. G:\MicroPython_Skills\upy-gen-driver-plugin\sample\phase_complete.upy_gen_driver_plugin.success.json
10. G:\MicroPython_Skills\upy-generate-plugin\scripts\download_drivers.py
```

Note: `F:\mpy-hardware-extension` is not a repository that the current Skill engineer needs to modify. The current Skill side priority is `G:\MicroPython_Skills`. Plugin / backend / cloud scheduling and submodule bumps need to be handled by the respective owners.

### Updated Board Material Reading Order

If the next new session only cares about board onboarding / official board mapping, it is recommended to read in this order:

```text
1. G:\blockless-plugin-course(1)\Board Dual JSON Configuration and Official Mapping Transformation Explanation-2026-07-11.md
2. G:\blockless-plugin-course(1)\Board Onboarding Process and Current Code Evidence Analysis-2026-07-12.md
3. G:\blockless-plugin-course(1)\Skill Lead Board Onboarding Current Work List-2026-07-12.md
4. G:\blockless-plugin-course(1)\MicroPython Official Board Verification Material List-2026-07-12.md
5. G:\MicroPython_Skills\upy-analyze-plugin\boards
6. G:\MicroPython_Skills\upy-analyze-plugin\boards\matching-rules.json
```

### Supplementary Tips for the New Session

```text
The original reading order of the index is the 2026-07-11 version. When handling GenDriver, you must continue reading the 2026-07-12 Skill Engineer GenDriver Field Decision and Erson Confirmation Issues document, and use the "Correction" and "Second Review" sections as the latest conclusions.

Do not attribute cold-driver generation to upy-generate-plugin; the correct owner is upy-gen-driver-plugin. upy-generate-plugin is only responsible for blocking deploy-ready success or outputting partial/next_action when cold_driver_required is unresolved.

The current Skill engineer does not modify F:\mpy-hardware-extension, only G:\MicroPython_Skills. Both plugin invocation and local skill invocation test scenarios need to be compatible, and continuous attention must be paid to protocol aspects such as session/checkpoint/resume/cancel/retry/timeout/idempotency/protocol/capability/error/artifact/permission prompt.
