---
name: mpos-package-app
description: Package and validate a single MicroPythonOS App as an MPK release artifact. Use when Codex needs to create a .mpk, validate an MPOS App manifest/icon/package structure, emit one app_index_entry.json fragment, run optional temporary install validation, or prepare AppStore/upystore publishing artifacts without uploading.
---

# MicroPythonOS App Packaging

## Role

Package an existing MicroPythonOS App directory into an installable, publishable `.mpk` artifact. Only handle release artifact preparation for a single App. Do not generate or fix App code, download dependencies, run the desktop simulator, or log in to or upload to upystore.

Prefer using this skill after `mpos-gen-app` static gates and `mpos-test-app` runtime smoke have completed. If test results are missing or failed, packaging may still proceed, but a warning must be recorded in `package_result.json`.

## User-Visible Language

Follow the language continuity rule of `mpos-dev`: if the current workflow started in Chinese, continue the packaging summary, warnings, failure reasons, and next-step suggestions in Chinese; if it started in English, continue in English. Code, commands, paths, API names, and JSON field names remain in English.

## Unified Project Log

After packaging is complete and `package_result.json` is produced, it must be registered in the project state directory:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-plan-app/scripts/update_plan_state.py record \
  --repo <repo-root> \
  --fullname <fullname> \
  --skill mpos-package-app \
  --phase package \
  --result <success|partial|failed> \
  --artifact package_result=<package_result.json> \
  --next-skill <handoff.next_skill-or-null> \
  --event "Packaged App as MPK and app_index_entry"
```

Missing or failed generation/test results may still allow packaging to continue with a warning, but the project state must retain these warnings so that `mpos-plan-app` can decide whether to proceed to deploy/publish. In the release pipeline, a successful or partial packaging result is handed off by default to `mpos-deploy-app` to produce `deploy_result.json`, and then to `mpos-publish-app`.

## Required Context

First load `mpos-dev`, and read:

- MPOS API precise index: `mpos-dev/reference/mpos_api_summary.json`
- LVGL API precise index: `mpos-dev/reference/lvgl_api_summary.json`
- Packaging, manifest, MPK, AppStore/upystore constraints: `mpos-dev/reference/docs-packaging.md`
- Local hard constraints: `<repo-root>/AGENTS.md`
- Current manifest test facts: `<repo-root>/tests/test_apps_manifest.py`
- Current MPK/streaming install facts: `<repo-root>/tests/test_streaming_unzip.py`
- Current installer facts: `<repo-root>/internal_filesystem/lib/mpos/content/streaming_unzip.py`

When old analysis documents or docs conflict with the current repository, the current repository and tests take precedence.
The API summary JSON must be read completely; it cannot be omitted because this phase "does not write business code". It is used to confirm AppManager/package-related MPOS APIs and to avoid stale manifest/API assumptions.

## Boundaries

- Do not modify business code, manifest, or icon in `internal_filesystem/apps/<fullname>/`.
- Do not fix lint, flake8, pylint, manifest, syntax, or import errors; return to `mpos-gen-app repair` for those.
- Do not download third-party dependencies; dependency preparation returns to `mpos-prepare-deps`.
- Do not run the desktop simulator, Web Port, or interact with devices; running tests returns to `mpos-test-app`.
- Do not install to the real `/apps` or overwrite the real `internal_filesystem/apps`.
- Do not log in, upload, or save upystore credentials; publishing handoff belongs to `mpos-publish-app`, and uploading is still done manually by the user.
- Do not modify MicroPythonOS OS/build source code.

## App Layout Strategy

Default new layout:

```text
internal_filesystem/apps/<fullname>/
  MANIFEST.JSON
  icon_64x64.png
  assets/<entrypoint>.py
```

Legacy layout is compatible, but must produce a warning:

```text
internal_filesystem/apps/<fullname>/
  META-INF/MANIFEST.JSON
  res/mipmap-mdpi/icon_64x64.png
  assets/<entrypoint>.py
```

If both the root `MANIFEST.JSON` and the legacy `META-INF/MANIFEST.JSON` exist, the root manifest takes precedence, and a warning about the coexistence of the legacy path is recorded. The same applies to the icon: the root `icon_64x64.png` takes precedence.

## Workflow

1. Determine `fullname` and the App directory, defaulting to `<repo-root>/internal_filesystem/apps/<fullname>`.
2. Read upstream `generation_result.json` and `app_test_result.json`. Prefer obtaining them from the artifact paths in `<repo-root>/tmp/mpos-plan-app/<fullname>/plan_state.json`; if they cannot be read, require the user to explicitly provide `--generation-result` / `--app-test-result`, or record a `not_provided` warning in `package_result.json`.
   - Missing, non-existent path, unparseable JSON, schema/phase mismatch, or `result != "success"` are all treated as warnings.
   - Upstream generation/test warnings must be output to both the terminal and `package_result.json`.
   - As long as the App directory, manifest, icon, entrypoint, and MPK structure validation can still pass, continue to generate `.mpk` and `app_index_entry.json`.
3. Run App validation:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-package-app/scripts/validate_mpos_app.py \
  --repo <repo-root> \
  --app-fullname <fullname>
```

4. Generate MPK, `app_index_entry.json`, and `package_result.json`:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-package-app/scripts/package_mpos_app.py \
  --repo <repo-root> \
  --app-fullname <fullname> \
  --revision 1 \
  --compression stored
```

Default output directory:

```text
<repo-root>/tmp/mpos-package-app/<fullname>/
```

5. When the user requests temporary install validation, add `--install-check`. This check only extracts to `tmp/mpos-package-app/<fullname>/install-check/` and re-validates the extracted App, without writing to the real App directory:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-package-app/scripts/package_mpos_app.py \
  --repo <repo-root> \
  --app-fullname <fullname> \
  --revision 1 \
  --compression stored \
  --install-check
```

6. Review the result:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-package-app/scripts/validate_package_result.py \
  <repo-root>/tmp/mpos-package-app/<fullname>/package_result.json
```

## MPK Rules

Must be enforced by the script:

- The manifest must contain a non-empty `publisher`; if missing, stop and hand back to `mpos-gen-app repair`, do not wait for a `MISSING_FIELD` error during upystore upload.
- `.mpk` is a ZIP archive.
- The MPK filename must use the upystore release revision format: `<fullname>_rN.mpk`, e.g., `<fullname>_r1.mpk`. The manifest `version` retains the App semantic version; the filename does not use `<version>`.
- The first local file header must be a directory entry for `<fullname>/`.
- There can only be one top-level directory.
- All entries must be under `<fullname>/`.
- The default compression method is `stored`. `--compression deflated` can be explicitly used, but must still pass local-header validation.
- Data descriptor flags are not allowed; the local header must contain the exact size.
- Exclude `.git/`, `__pycache__/`, `*.pyc`, `__MACOSX/`, `._*`, `.DS_Store`.
- Use a fixed timestamp and stable sorting to enable reproducible builds.

## App Index

This skill only generates the `app_index_entry.json` for the current App by default; it does not merge or modify the full `app_index.json`. Full store index merging involves release repositories, URL strategies, sorting, conflict resolution, and go-live processes, which are not the default responsibility of this skill.

`emit_app_index_entry.py` will generate a single metadata entry based on the manifest, and generate the following based on the base URL:

```text
<base_url>/apps/<fullname>/icons/<fullname>_<version>_64x64.png
<base_url>/apps/<fullname>/mpks/<fullname>_r<revision>.mpk
```

The default base URL is `https://apps.micropythonos.com`. If the target is upystore upload, only local metadata and MPK are prepared; no upload is performed.

## Output JSON

`package_result.json` must match the shape of `templates/package_result.json` and pass `scripts/validate_package_result.py`:

- `schema_version` is `mpos-package-app-v1`.
- `phase` is `package`.
- `result` is `success`, `partial`, or `failed`.
- `checks[]` must include at least `app_validation`, `generation_result`, `app_test_result`, `mpk_validation`, `app_index_entry`.
- When `--install-check` is requested, append `temporary_install_validation`.
- `package.revision` must be a positive integer, `package.filename_policy` must be `upystore-release-revision`, and `package.mpk_path` must match `<fullname>_rN.mpk`.
- Missing/failed generation/test results only record warnings; they cannot cause the package to fail.
- `handoff.next_skill` is typically `mpos-deploy-app` or `null`; only hand off to `mpos-publish-app` when a matching `deploy_result.json` already exists and the user explicitly wants only the publishing handoff.

## Failure Handling

- Missing App manifest, entrypoint, classname, or icon: stop packaging, hand back to `mpos-gen-app repair`.
- MPK local-header, top-level directory, data descriptor, or illegal file failures: fix this skill's script or re-package; do not let `mpos-gen-app` modify business code.
- Temporary install validation failure: if it is an MPK structure issue, stay in this skill; if the extracted manifest/App files are missing, hand back to `mpos-gen-app repair`.
- Missing or failed generation/test: continue to output MPK, but set `result` to `partial` and record warnings.
