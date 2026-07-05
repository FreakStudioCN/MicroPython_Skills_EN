# Skill Interface Documentation Index

Each skill has one interface document, filled in according to the `_template.md` format.

| # | Skill | Phase | Document Status | Filled by | Notes |
|---|-------|-------|---------|--------|------|
| 1 | upy-analyze | Phase 1 | ✅ Finalized | — | Requirements analysis + driver search |
| 2 | upy-select-hw | Phase 2 | ✅ Finalized | — | MCU selection + pin assignment + BOM |
| 3 | upy-scaffold | Phase 3 | ✅ Finalized | — | Project scaffold generation |
| 4 | upy-generate | Phase 4 | ✅ Finalized | — | Business code generation (heaviest) |
| 5 | upy-simulate | Phase 4.5 | ✅ Finalized | — | Full PC-side simulation |
| 6 | upy-deploy | Phase 5 | ✅ Finalized | — | Flashing and execution |
| 7 | upy-autofix | Phase 6 | ✅ Finalized | — | Auto-fix orchestration |
| 8 | upy-wiring | Phase 7a | ✅ Finalized | — | Wiring diagram generation |
| 9 | upy-diagram | Phase 7b | ✅ Finalized | — | Architecture diagram generation |
| 10 | upy-gen-driver | Exception path | ✅ Finalized | — | Uncommon hardware driver generation |
| 11 | upy-publish | Wrap-up | ⚠ Pending | — | Driver specification packaging + publishing to upypi |
| 12 | upy-firmware-wrapper | Non-Phase | ⚠ Pending | — | Firmware API Wrapper package writing specification |

Status description: ⚠ Pending → 📝 In progress → ✅ Finalized → 🔄 Needs revision

## Suggested Filling Order

No need to follow the Phase order. It is recommended to follow **plugin-side dependency priority**:

1. **Fill upy-analyze first** — Contains the first approval_request (device confirmation card), allowing the plugin side to develop the approval card component based on this.
2. **Then fill upy-deploy** — Contains device_command and stream, allowing the plugin side to develop device passthrough and terminal panels.
3. **Then fill upy-generate** — Contains a large number of status_update + file_operation, verifying the progress timeline and file operations.
4. Fill the rest as needed.

This way, the plugin side can develop the most core UI components at the earliest stage.
