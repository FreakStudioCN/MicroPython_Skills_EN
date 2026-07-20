# MicroPythonOS Packaging & Store Reference

This document is generated based on the `docs.micropythonos.com` sitemap/search index re-read on 2026-07-14, `https://upystore.io/`, `https://upystore.io/app_index.json`, `https://upystore.io/api/v1/apps`, and corrected with local `scripts/bundle_apps.sh`, `tests/test_apps_manifest.py`, `tests/test_streaming_unzip.py`, `internal_filesystem/lib/mpos/content/streaming_unzip.py`.

## When to Read

Read this file when creating `.mpk` files, validating app manifests, generating app index metadata, preparing for AppStore/upystore/BadgeHub publishing, or troubleshooting installation failures.

## Source Coverage

- `apps/bundling-apps/`
- `apps/appstore/`
- `apps/badgehub/`
- Local `scripts/bundle_apps.sh`
- Local MPK tests and `StreamingUnzip`

## MPK Contract

`.mpk` is a ZIP archive with strict stream-order constraints:

- The first ZIP local header must be a directory entry named `<fullname>/`.
- `<fullname>/` must match the `fullname` in the app manifest.
- Only one top-level directory is allowed.
- All files must reside under that top-level directory.
- Supported compression methods are stored and deflated.
- The data descriptor flag is not supported; the local file header must contain accurate sizes.

Valid stream order:

```text
com.example.app/
com.example.app/MANIFEST.JSON
com.example.app/assets/main.py
com.example.app/icon_64x64.png
```

The legacy nested layout `META-INF/MANIFEST.JSON` and `res/mipmap-mdpi/icon_64x64.png` is still compatible with the current installation path, but new packages should prefer the root directory `MANIFEST.JSON` and `icon_64x64.png`.

Malformed archives will be rejected during streaming extraction, potentially failing before or during file writing.

## Local Manifest Validation

Validate before packaging:

- The directory name equals the manifest `fullname`.
- The manifest JSON is parseable.
- `name`, `publisher`, and `version` exist and are non-empty.
- `version` is a canonical integer dot-separated string.
- Each activity/service entrypoint ends with `.py`.
- Each entrypoint exists at a relative path from the app root directory.
- Each declared classname appears in the entrypoint source code.

These rules correspond to `tests/test_apps_manifest.py`.

## Reproducible Packaging

The local batch packaging script is `scripts/bundle_apps.sh`, which generates packages from `internal_filesystem/apps`.

Single app packaging should also follow these principles:

- Change to the parent directory of the app repository before zipping, ensuring the path includes `<fullname>/`.
- Place the directory entry before file entries.
- Maintain stable entry ordering.
- Use fixed timestamps for reproducible builds.
- Exclude `.git/`, `__pycache__/`, `*.pyc`, `__MACOSX/`, `._*`.
- Ensure the icon exists at the root `icon_64x64.png`; the legacy `res/mipmap-mdpi/icon_64x64.png` is only a compatibility path.

Recommended single-app scripts:

- `mpos-package-app/scripts/validate_mpos_app.py`
- `scripts/package_mpos_app.py`
- `scripts/validate_mpk.py`
- `scripts/emit_app_index_entry.py`

## App Index Metadata

A MicroPythonOS-compatible app index should include:

- `name`
- `publisher`
- `short_description`
- `long_description`
- `icon_url`
- `download_url`
- `fullname`
- `version`
- `category`
- `activities`
- `services` (if present)

Activity metadata should preferentially use the full manifest object with `classname`, `entrypoint`, and `intent_filters`. Do not output string-type activity lists for newly generated packages.
upystore uploads also require a non-empty `publisher` in the manifest; this must be intercepted early during local packaging, and `.mpk` files with missing fields must not be handed to users for upload.

## AppStore Backend

The AppStore can pull applications from multiple backends.

- MicroPythonOS curated app index: manually reviewed app metadata and `.mpk` download URLs.
- BadgeHub: community appstore, including project summary, project detail endpoint, releases, icons, and download packages.
- upystore: an external store for developers; prepare packages and metadata locally, then let users upload manually.

Storefront fields such as `slug`, `revision`, `tags`, `hardware_tags`, `screenshots`, install count, download count, stars, and release time are suitable for publishing summaries but cannot replace the local `MANIFEST.JSON`.

## upystore Specific Recommendations

From a skill perspective, the upload process for `https://upystore.io/` should remain simple: generate and validate the package, then provide the user with a link to the website or Developer Console. Do not request or save account passwords.

2026-07-14 review results:

- `https://upystore.io/app_index.json` is currently a list of 10 apps, with fields including `activities`, `category`, `download_url`, `fullname`, `icon_url`, `long_description`, `name`, `publisher`, `short_description`, `version`.
- `https://upystore.io/api/v1/apps` currently returns `apps`, `filters`, `pagination`, with pagination showing `total=10`, `total_pages=1`, and additionally includes storefront fields such as `slug`, `revision`, `tags`, `hardware_tags`, `min_os_version`, `min_api_level`, `screenshots`, `installs_count`, `downloads_count`, `stars_count`, `released_at`.
- Reading the public app detail pages one by one for the 10 `slug` values from `api/v1/apps` resulted in `UPYSTORE_DETAIL_OK=10/10`. These detail pages can be used for manual verification after upload but cannot replace local manifest and MPK validation.

Post-upload checks:

- If the user provides a URL, fetch the uploaded app index or API detail.
- Verify all required fields are present.
- Download the generated `.mpk`.
- Verify the first ZIP entry is `<fullname>/`.
- Confirm there are no macOS resource fork files.
- Optional: perform installation verification on device or desktop.

## BadgeHub Specific Recommendations

BadgeHub publishing is a separate path from upystore. It uses BadgeHub metadata and releases. When targeting BadgeHub, check the project slug, release version, `.mpk` artifact, icon, and project description.

## Local Rules from AGENTS

- Run `make lint` after modifying code or scripts.
- Prefer using existing `Makefile` targets when equivalent entry points exist.
- Write temporary files to the repository `tmp/`.
- Do not weaken `StreamingUnzip` validation to accept malformed packages.
- Do not conflate installing apps with flashing firmware.
