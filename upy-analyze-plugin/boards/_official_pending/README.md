# Official MicroPython Board Source Index

Collation batch: 2026-07-17
Condensed batch: 2026-07-17 raw-assets-condensed

This directory only retains the official board data master index, collation notes, and small audit report archives. The formal select-hw board definitions remain in the parent `boards/*.json`.

## Active Files

- `MicroPython官方板卡资料源索引.csv`: Data source index for 222 official MicroPython boards. Retains official pages, GitHub source, vendor pages, board image URLs, pinout URLs/source, formal JSON, firmware, source file parsing status, HTML/OCR summaries, and manual review items.
- `_archive_20260717/official_pending_cleanup_manifest_20260717.json`: Initial archive manifest from 2026-07-17.
- `_archive_20260717/raw_asset_condense_manifest_20260717.json`: Condensed raw asset manifest.
- `_archive_20260717/reports_20260717.zip`: HTML failure list, screenshot OCR summaries, formal release report, and final enhanced report.

## Raw Assets

Original pinout images, vendor page screenshots, old pending JSON files, and historical CSV backups are no longer retained in this directory. The main CSV retains source URLs, source paths, OCR summaries, quality ratings, and items pending review; formal board content is based on `boards/*.json`.

When visual materials need to be re-reviewed, re-fetch from the CSV's `板卡图片URL或文件名`, `pinout图片URL或文件名`, `厂商产品页HTML`, and `GitHub或source页面`.

## Notes

- `features` still uses the MicroPython official `board.json.features` as high-level capability clues.
- `onboard_peripherals` and pin assignments are based on the formal `boards/*.json`; peripherals lacking pin mappings will not form hard conflicts.
- generic target is retained as a generic target semantic, primarily based on the MicroPython source directory.
- When adding/validating boards in the future, first update the main CSV in this directory, then re-run the official source/HTML/pinout audit and formal JSON generation scripts.
