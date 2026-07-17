---
name: mpos-package-app
description: Package and validate a single MicroPythonOS App as an MPK release artifact. Use when Codex needs to create a .mpk, validate an MPOS App manifest/icon/package structure, emit one app_index_entry.json fragment, run optional temporary install validation, or prepare AppStore/upystore publishing artifacts without uploading.
---

# MicroPythonOS App Packaging

## Role

Package an existing MicroPythonOS App directory into an installable, publishable `.mpk` artifact. Only handle release artifact preparation for a single App. Do not generate or fix App code, download dependencies, run a desktop simulator, or log in to or upload to upystore.

Prefer to use this skill after `mpos-gen-app` static gating is complete and `mpos-test-app` runtime smoke has finished. If test results are missing or failed, packaging may still proceed, but a warning must be recorded in `package_result.json`.

## Unified Project Log

After completing packaging and producing `package_result.json`, you must log to the project state directory:

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

Missing or failed generation/test results may still allow packaging to continue with a warning, but the project state must retain these warnings so that `mpos-plan-app` can decide whether to proceed to deploy/publish.

## Required Context

First load `mpos-dev`, and read:

- Packaging, manifest, MPK, AppStore/upystore constraints: `mpos-dev/reference/docs-packaging.md`
- Local hard constraints: `<repo-root>/AGENTS.md`
- Current manifest test facts: `<repo-root>/tests/test_apps_manifest.py`
- Current MPK/streaming install facts: `<repo-root>/tests/test_streaming_unzip.py`
- Current installer facts: `<repo-root>/internal_filesystem/lib/mpos/content/streaming_unzip.py`

When older analysis documents or docs conflict with the current repository, the current repository and tests take precedence.

## Boundaries

- Do not modify business code, manifest, or icon in `internal_filesystem/apps/<fullname>/`.
- Do not fix lint, flake8, pylint, manifest, syntax, or import errors; return these to `mpos-gen-app repair`.
- Do not download third-party dependencies; dependency preparation returns to `mpos-prepare-deps`.
- Do not run a desktop simulator, Web Port, or device interaction; running tests returns to `mpos-test-app`.
- Do not install to the real `/apps`, do not overwrite the real `internal_filesystem/apps`.
- Do not log in, upload, or save upystore credentials; publishing/uploading is the user's responsibility or deferred to a future `mpos-publish-app`.
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

If both the root `MANIFEST.JSON` and the legacy `META-INF/MANIFEST.JSON` exist, the root manifest takes precedence, and a warning about the legacy path coexistence must be recorded. The same applies to the icon: the root `icon_64x64.png` takes precedence.

## Workflow

1. Determine `fullname` and the App directory, defaulting to `<repo-root>/internal_filesystem/apps/<fullname>`.
2. Read optional `generation_result.json` and `app_test_result.json`. Missing or failed results only produce warnings and do not block packaging.
   - Missing, non-existent path, unparseable JSON, schema/phase mismatch, or `result != "success"` are all treated as warnings.
   - Upstream generation/test warnings must be output to both the terminal and `package_result.json`.
   - As long as the App directory, manifest, icon, entrypoint, and MPK structure validation can still pass, continue to generate the `.mpk` and `app_index_entry.json`.
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
  --compression stored
```

Default output directory:

```text
<repo-root>/tmp/mpos-package-app/<fullname>/
```

5. When the user requests temporary install validation, add `--install-check`. This check only extracts to `tmp/mpos-package-app/<fullname>/install-check/` and re-validates the unpacked App, without writing to the real App directory:

```bash
PYTHONDONTWRITEBYTECODE=1 /home/leeqingshui/mp_env/bin/python \
  /home/leeqingshui/MicroPython_Skills/mpos-package-app/scripts/package_mpos_app.py \
  --repo <repo-root> \
  --app-fullname <fullname> \
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

- `.mpk` is a ZIP archive.
- The first local file header must be the `<fullname>/` directory entry.
- There can only be one top-level directory.
- All entries must be under `<fullname>/`.
- The default compression method is `stored`. `--compression deflated` can be explicitly specified, but must still pass local-header validation.
- Data descriptor flags are not allowed; the local header must contain the exact size.
- Exclude `.git/`, `__pycache__/`, `*.pyc`, `__MACOSX/`, `._*`, `.DS_Store`.
- Use a fixed timestamp and stable sorting to enable reproducible builds.

## App Index

This skill only generates the `app_index_entry.json` for the current App by default. It does not merge or modify the full `app_index.json`. Full store index merging involves release repositories, URL strategies, sorting, conflict resolution, and go-live processes, which are not the default responsibility of this skill.

`emit_app_index_entry.py` generates single-entry metadata based on the manifest, and generates the following based on the base URL:

```text
<base_url>/apps/<fullname>/icons/<fullname>_<version>_64x64.png
<base_url>/apps/<fullname>/mpks/<fullname>_<version>.mpk
```

The default base URL is `https://apps.micropythonos.com`. If the target is upystore upload, only local metadata and MPK are prepared; no upload is performed.

## Output JSON

`package_result.json` must match the shape of `templates/package_result.json` and pass `scripts/validate_package_result.py`:

- `schema_version` is `mpos-package-app-v1`.
- `phase` is `package`.
- `result` is `success`, `partial`, or `failed`.
- `checks[]` must include at least `app_validation`, `generation_result`, `app_test_result`, `mpk_validation`, and `app_index_entry`.
- When `--install-check` is requested, append `temporary_install_validation`.
- Missing/failed generation/test results only record warnings and must not cause packaging to fail.
- `handoff.next_skill` is typically `mpos-publish-app` or `null`.

## Failure Handling

- Missing App manifest, entrypoint, classname, or icon: stop packaging, hand back to `mpos-gen-app repair`.
- MPK local-header, top dir, data descriptor, or illegal file failures: fix this skill's script or re-package; do not let `mpos-gen-app` modify business code.
- Temporary install validation failure: if it is an MPK structure issue, stay in this skill; if the unpacked manifest/App files are missing, hand back to `mpos-gen-app repair`.
- Missing or failed generation/test: continue to output the MPK, but set `result` to `partial` and record warnings.
