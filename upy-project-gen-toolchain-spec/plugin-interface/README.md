# Plugin Interface Documentation

> Target readers:
> - **Plugin-side engineers** (TypeScript / VS Code Extension) — implement UI + local I/O passthrough
> - **Server-side engineers** (Python / LLM integration) — implement skill scheduling + protocol message generation
> - **Skill maintainers** (embedded background) — modify SKILL.md to adapt to the plugin protocol

---

## Directory Structure

```
plugin-interface/
├── README.md                    ← This file, directory index
├── 01-architecture.md           ← System architecture: responsibilities and boundaries of plugin/server/skill
├── 02-protocol.md               ← Communication protocol: complete JSON Schema for 7 message types
├── 03-parallel-dev.md           ← Parallel development strategy: how both sides develop independently + mock testing
│
├── skills/                      ← Interface definitions for each skill (filled in one by one)
│   ├── README.md                ← Skill interface index + fill status
│   ├── _template.md             ← Skill interface document template (fill in for new skills)
│   ├── upy-analyze.md           ← [Pending] Phase 1 requirement analysis
│   ├── upy-select-hw.md         ← [Pending] Phase 2 hardware selection
│   ├── upy-scaffold.md          ← [Pending] Phase 3 project scaffolding
│   ├── upy-generate.md          ← [Pending] Phase 4 code generation
│   ├── upy-simulate.md          ← [Pending] Phase 4.5 PC simulation
│   ├── upy-deploy.md            ← [Pending] Phase 5 flash and run
│   ├── upy-autofix.md           ← [Pending] Phase 6 auto-fix
│   ├── upy-wiring.md            ← [Pending] Phase 7a wiring diagram
│   ├── upy-diagram.md           ← [Pending] Phase 7b architecture diagram
│   ├── upy-gen-driver.md        ← [Pending] Exception path: cold hardware driver
│   └── upy-firmware-wrapper.md   ← [Pending] Firmware API Wrapper package writing specification
│
└── mock-messages/               ← Mock JSON samples for each message type
    ├── README.md                ← Mock usage instructions
    ├── approval-request.json
    ├── status-update.json
    ├── device-command.json
    ├── file-operation.json
    ├── script-run.json
    ├── phase-complete.json
    └── stream.json
```

## Reading Order

| Role | Read First | Read Next | Finally |
|------|------------|-----------|---------|
| Plugin-side engineer | `01-architecture.md` → `02-protocol.md` | `03-parallel-dev.md` (plugin-side section) | `mock-messages/README.md` |
| Server-side engineer | `01-architecture.md` → `02-protocol.md` | `skills/_template.md` | Specific skill documentation |
| Skill maintainer | `01-architecture.md` | `skills/_template.md` | `02-protocol.md` (understand message types) |

## Current Status

- `01-architecture.md` — Filled
- `02-protocol.md` — Filled
- `03-parallel-dev.md` — Filled
- `skills/` — Partially filled (10/12 finalized, 1 pending, 1 non-Phase)
- `mock-messages/` — Pending

## Related Files (not in this directory)

| File | Location | Purpose |
|------|----------|---------|
| boards.json + documentation | `../../upy-analyze/boards/` | Board database, plugin gallery + LLM selection |
| project-manifest.schema.json | `../project-manifest.schema.json` | Data contract for each Phase |
| wiring.schema.json | `../wiring.schema.json` | Wiring diagram data contract |
| diagram.schema.json | `../diagram.schema.json` | Architecture diagram data contract |
