# `upy-analyze-plugin/boards`

This directory stores the board metadata used by `upy-analyze-plugin` itself.

## Current Strategy

- The plugin-based `analyze` no longer treats `boards/` as a placeholder directory
- It currently syncs the board JSON files from the original `G:\MicroPython_Skills\upy-analyze\boards`
- If the plugin side later needs to extend fields, evolution will continue only in this directory

## Purpose of These Board Data

Primarily used to fulfill the input contract related to `pre_selected_board`, including:

- `id`
- `display_name`
- `mcu`
- `chip_family`
- `firmware_url`

And in the future, to provide unified basic board data for:

- The plugin-side board selector
- Downstream integration with `select-hw`
- Local interactive simulation entry points

## Current Principles

- The original skill `upy-analyze` remains unchanged
- The plugin-based skill `upy-analyze-plugin` maintains its own copy of board assets
- If the plugin version's board schema changes in the future, the version in this directory takes precedence
