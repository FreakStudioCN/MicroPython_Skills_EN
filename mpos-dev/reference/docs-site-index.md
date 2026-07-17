# MicroPythonOS Docs Site Index

This file is generated based on the `https://docs.micropythonos.com/sitemap.xml` and `search/search_index.json` re-read on 2026-07-14.

Purpose: Audit docs coverage. This round read all 61 pages sequentially according to the sitemap, with result `DOCS_FETCH_OK=61/61 failed=0`; the docs search index contains 977 page/section entries. Specific work rules are split into the topic reference files below.

## Reference Routes

- `docs-app-model.md`: App model, Activity, Service, Intent, built-in apps, native apps.
- `docs-packaging.md`: MPK, app index metadata, AppStore, BadgeHub, upystore packaging checks.
- `docs-frameworks.md`: Framework architecture and manager/service APIs.
- `docs-deploy-targets.md`: Runtime targets, desktop, device installation, firmware flashing, QEMU, WebSerial, browser preview.
- `docs-os-development.md`: Compilation, testing, porting, release/merge checklist, file formats.
- `docs-web-port.md`: WebAssembly/browser runtime and web build target.

## Sitemap Coverage Mapping

| Docs page | Reference |
|---|---|
| `/` | `docs-app-model.md`, `docs-deploy-targets.md` |
| `/overview/` | `docs-app-model.md`, `docs-deploy-targets.md` |
| `/apps/` | `docs-app-model.md` |
| `/apps/app-lifecycle/` | `docs-app-model.md` |
| `/apps/appstore/` | `docs-app-model.md`, `docs-packaging.md` |
| `/apps/badgehub/` | `docs-packaging.md` |
| `/apps/built-in-apps/` | `docs-app-model.md` |
| `/apps/bundling-apps/` | `docs-packaging.md` |
| `/apps/creating-apps/` | `docs-app-model.md` |
| `/apps/native-apps/` | `docs-app-model.md`, `docs-os-development.md` |
| `/architecture/boot-sequence/` | `docs-os-development.md` |
| `/architecture/filesystem/` | `docs-app-model.md`, `docs-os-development.md` |
| `/architecture/frameworks/` | `docs-frameworks.md` |
| `/architecture/intents/` | `docs-app-model.md` |
| `/architecture/overview/` | `docs-app-model.md`, `docs-os-development.md` |
| `/frameworks/` | `docs-frameworks.md` |
| `/frameworks/app-manager/` | `docs-frameworks.md` |
| `/frameworks/appearance-manager/` | `docs-frameworks.md` |
| `/frameworks/audiomanager/` | `docs-frameworks.md` |
| `/frameworks/battery-manager/` | `docs-frameworks.md` |
| `/frameworks/build-info/` | `docs-frameworks.md` |
| `/frameworks/connectivity-manager/` | `docs-frameworks.md` |
| `/frameworks/device-info/` | `docs-frameworks.md` |
| `/frameworks/display-metrics/` | `docs-frameworks.md` |
| `/frameworks/download-manager/` | `docs-frameworks.md` |
| `/frameworks/file-explorer-activity/` | `docs-frameworks.md` |
| `/frameworks/focus/` | `docs-frameworks.md` |
| `/frameworks/font-manager/` | `docs-frameworks.md` |
| `/frameworks/input-activity/` | `docs-frameworks.md` |
| `/frameworks/input-manager/` | `docs-frameworks.md` |
| `/frameworks/lights-manager/` | `docs-frameworks.md` |
| `/frameworks/notification-manager/` | `docs-frameworks.md` |
| `/frameworks/number-format/` | `docs-frameworks.md` |
| `/frameworks/preferences/` | `docs-frameworks.md` |
| `/frameworks/sensor-manager/` | `docs-frameworks.md` |
| `/frameworks/service/` | `docs-frameworks.md`, `docs-app-model.md` |
| `/frameworks/setting-activity/` | `docs-frameworks.md` |
| `/frameworks/settings-activity/` | `docs-frameworks.md` |
| `/frameworks/task-manager/` | `docs-frameworks.md` |
| `/frameworks/time-zone/` | `docs-frameworks.md` |
| `/frameworks/webserver/` | `docs-frameworks.md` |
| `/frameworks/widget-animator/` | `docs-frameworks.md` |
| `/frameworks/wifi-service/` | `docs-frameworks.md` |
| `/getting-started/` | `docs-deploy-targets.md` |
| `/getting-started/running/` | `docs-deploy-targets.md` |
| `/getting-started/supported-hardware/` | `docs-deploy-targets.md` |
| `/os-development/` | `docs-os-development.md` |
| `/os-development/automated-testing/` | `docs-os-development.md` |
| `/os-development/compiling/` | `docs-os-development.md` |
| `/os-development/emulating-esp32-on-desktop/` | `docs-deploy-targets.md`, `docs-os-development.md` |
| `/os-development/installing-on-esp32/` | `docs-deploy-targets.md`, `docs-os-development.md` |
| `/os-development/linux/` | `docs-deploy-targets.md`, `docs-os-development.md` |
| `/os-development/macos/` | `docs-deploy-targets.md`, `docs-os-development.md` |
| `/os-development/porting-guide/` | `docs-os-development.md` |
| `/os-development/running-on-desktop/` | `docs-deploy-targets.md` |
| `/os-development/windows/` | `docs-deploy-targets.md`, `docs-os-development.md` |
| `/other/merge-checklist/` | `docs-os-development.md` |
| `/other/release-checklist/` | `docs-os-development.md` |
| `/other/supported-file-formats/` | `docs-os-development.md` |
| `/web-port/developer/` | `docs-web-port.md`, `docs-os-development.md` |
| `/web-port/using/` | `docs-web-port.md`, `docs-deploy-targets.md` |

## Notes

- This is not a verbatim mirror of the docs site, but rather a coverage table and routing table for the split reference files.
- Each topic reference will paraphrase the docs content and add local `AGENTS.md` constraints when the public documentation is inconsistent with the local repository.
- If the task requires precise API signatures, in addition to the topic docs reference, also read `mpos-api-reference.md` and `lvgl_api_summary.json`.
