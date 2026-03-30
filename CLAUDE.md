# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal ZMK firmware configuration based on [urob/zmk-config](https://github.com/urob/zmk-config). Builds a 34-key base layout shared across multiple split/unibody keyboards (Cradio/Ferris Sweep, Fract, MikeFive, MikeCinq). Targets ZMK v0.3 with custom ZMK modules for extended functionality. Currently configured for macOS with [AeroSpace](https://github.com/nikitabobko/AeroSpace) tiling window manager.

## Build Commands

The project uses `just` (command runner) with a Nix-based dev environment (auto-activated via `direnv`).

```bash
just build all              # Build firmware for all targets in build.yaml
just build <expr>           # Build matching targets (e.g., "cradio", "fract")
just build all -p           # Pristine build (passed to west)
just list                   # List all build targets
just clean                  # Clear build cache and firmware output
just init                   # Initialize west workspace (west init + update + zephyr-export)
just update                 # Update ZMK dependencies via west
just draw                   # Generate split and unibody SVG visualizations with keymap-drawer
just test <path>            # Run a test case (builds with native_posix_64, compares keycode snapshots)
just test <path> --verbose  # Run test with output
just test <path> --auto-accept  # Accept new snapshot as baseline
```

Build artifacts go to `firmware/` (UF2/BIN files). Build cache is in `.build/`.

## Architecture

### Config Structure

- **`config/base.keymap`** — Primary keymap shared by all keyboards. Defines all layers (DEF, NAV, FN, NUM, SYS, MOUSE), homerow mods, mod-morphs, and behaviors. This is the main file you'll edit.
- **`config/combos.dtsi`** — 27 combos for symbols/shortcuts (included by base.keymap). Uses a custom `ZMK_COMBO_8` macro for HRM-compatible combos.
- **`config/leader.dtsi`** — Leader key sequences for Unicode (German umlauts, Greek letters) and system commands.
- **`config/mouse.dtsi`** — Mouse emulation config with acceleration settings (tuned for 4K displays).
- **`config/{cradio,fract}.keymap`** — 34-key per-keyboard keymaps. Simple wrappers that include base.keymap.
- **`config/{mikefive,mikecinq}.keymap`** — 36-key per-keyboard keymaps. Define `USE_SMART_MOUSE` and override `ZMK_BASE_LAYER` to add extra thumb keys (`&kp LGUI`, `&smart_mouse`).
- **`config/{cradio,fract,mikefive,mikecinq}.conf`** — Per-keyboard Kconfig (sleep, BLE, battery settings).
- **`config/west.yml`** — West manifest pinning ZMK, Zephyr, and all module versions.

### Custom Shields

Custom board definitions live in `config/boards/shields/{fract,mikefive,mikecinq}/` with overlay files, matrix transforms, and Kconfig.

### ZMK Modules

Custom functionality is provided via ZMK modules in `modules/zmk/`:
- **adaptive-key** — Context-aware key behavior
- **auto-layer** — Auto-deactivating layers (powers Numword and Smart-Mouse)
- **helpers** — Devicetree helper macros (`ZMK_HOLD_TAP`, `ZMK_COMBO`, `ZMK_LAYER`, etc.)
- **leader-key** — Leader key sequences
- **tri-state** — Alt-Tab window switcher behavior
- **unicode** — Unicode input macros

### Key Design Patterns

- **Timeless homerow mods**: Uses `balanced` flavor with large `tapping-term-ms` (280ms) + `require-prior-idle-ms` (150ms) + positional hold-tap with `hold-trigger-on-release`. Left-hand mods (`hml`) only trigger on right-hand keys and vice versa.
- **HRM-compatible combos**: Custom `ZMK_COMBO_8` macro wraps combos so they don't interfere with homerow mods.
- **Nav cluster**: Hold-tap keys for navigation with word/line/document jumps. `NAV_BSPC`/`NAV_DEL` delete words on long-tap (Option+Bspc/Del on macOS). `NAV_LEFT`/`NAV_RIGHT` jump to start/end of line (Cmd+Left/Right). `NAV_UP`/`NAV_DOWN` jump to start/end of document (Cmd+Up/Down). Linux variants use Ctrl instead of Cmd/Option.
- **Conditional layers**: FN+NUM → SYS. On 34-key boards: NAV+FN → MOUSE (conditional layer). On 36-key boards: smart_mouse via tri-state (combo or extra thumb key).
- **USE_SMART_MOUSE**: Preprocessor flag defined by 36-key keymaps. Enables `ZMK_TRI_STATE(smart_mouse)` and its combo, disables the `ZMK_CONDITIONAL_LAYER(mouse)`. ZMK_TRI_STATE and ZMK_CONDITIONAL_LAYER cannot coexist for the same layer.
- **Mod-morphs for punctuation**: `, → ;` (shifted), `. → :` (shifted), `? → !` (shifted), with Ctrl+Shift variants for `<` and `>`.
- **Magic shift/repeat**: Right thumb is context-aware — repeat after alpha, sticky-shift after other keys, capsword on double-tap.
- **macOS shortcuts**: Copy/cut/paste combos use Cmd+C/X/V. Desktop switching uses AeroSpace keybindings (Alt+Shift+Tab, Alt+Ctrl+Shift+Tab). Full Screen (Alt+Shift+F) and Float (Alt+Ctrl+Shift+F) via AeroSpace. Exposé via Ctrl+Up.

### Visualization

`just draw` generates `draw/split.svg` (split keyboard) and `draw/unibody.svg` (ortholinear Fract) from `draw/base.yaml`. Config files: `draw/config.yaml` (split, includes `raw_binding_map` for human-readable key labels) and `draw/config-unibody.yaml` (unibody with zero split gap). When changing key bindings, update the `raw_binding_map` in `draw/config.yaml` to keep SVG labels accurate.

## Build Targets

Defined in `build.yaml`. All use `nice_nano_v2` board:
- `cradio_left`, `cradio_right` (Ferris Sweep, 34-key split)
- `fract` (34-key unibody ortholinear)
- `mikefive`, `mikecinq` (36-key variants)

## CI/CD

GitHub Actions workflows in `.github/workflows/`:
- `build-nix.yml` — Nix-based build, triggers on changes to `config/` or `build.yaml`
- `build.yml` — Standard ZMK build (currently disabled)

## Devicetree Syntax

Config files use ZMK's Devicetree syntax (`.keymap`, `.dtsi`, `.overlay`). The helper macros from `modules/zmk/helpers` simplify common patterns — use them instead of raw Devicetree when possible (e.g., `ZMK_HOLD_TAP`, `ZMK_COMBO`, `ZMK_LAYER`, `ZMK_MOD_MORPH`).
