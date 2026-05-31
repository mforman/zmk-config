# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal [ZMK firmware](https://zmk.dev) configuration for a 34-key **Urchin** split keyboard, forked from [urob/zmk-config](https://github.com/urob/zmk-config). Pinned to **ZMK v0.3.0** and **urob modules v0.3**. The Nix flake + direnv provides a fully isolated build environment; no system-level toolchain install is needed.

## Build environment

The nix shell activates automatically via `direnv` when you `cd` into the workspace. In Claude Code sessions, prefix commands with `direnv exec .` since direnv doesn't activate automatically:

```bash
direnv exec . just <recipe>
```

After cloning fresh or after a `west.yml` change:

```bash
direnv exec . just init   # west init + update + zephyr-export
```

## Common commands

```bash
just build all            # build all targets in build.yaml
just build all -p         # pristine (clean) build
just build urchin         # build only the urchin targets
just update               # pull latest module revisions (after west.yml changes)
just draw                 # regenerate draw/base.svg from base.keymap
just list                 # list all build targets
just clean                # delete .build/ and firmware/
just clean-all            # also wipe .west/ and zmk/
```

After `just update`, always run `direnv exec . west zephyr-export` before building — CMake won't find Zephyr otherwise.

Firmware lands in `firmware/*.uf2`. Flash by double-tapping the reset button to enter UF2 bootloader mode, then copy the file.

## Repository layout

```
config/          — ZMK user config (the only thing you normally edit)
  west.yml       — dependency manifest: pins ZMK, Zephyr, and all modules
  base.keymap    — the shared 4-layer Colemak keymap (included by board-specific keymaps)
  urchin.keymap  — board entry: sets CONFIG_WIRELESS, includes base.keymap
  urchin.conf    — Kconfig flags for the Urchin build
  combos.dtsi    — combo definitions, sourced from base.keymap
build.yaml       — GitHub Actions / just build matrix (board + shield combos)
Justfile         — build recipes wrapping west
flake.nix        — Nix dev shell (Zephyr SDK 0.16.9, python-yq, keymap-drawer)
zmk/             — ZMK source (managed by west, do not edit directly)
zephyr/          — Zephyr RTOS (managed by west, do not edit directly)
modules/zmk/     — urob ZMK modules: adaptive-key, auto-layer, helpers, tri-state
draw/            — keymap-drawer config and output SVG/PNG
firmware/        — build output UF2 files
```

## Keymap architecture

All keymap logic lives in `config/base.keymap`. It defines four layers:

- **colemak** (0) — base alpha layer with homerow mods
- **lower** (1) — F-keys (left) + nav cluster (right); activated by left thumb `&lt LOWER TAB`
- **raise** (2) — numpad (left) + symbols/brackets (right); activated by right thumb `&lt RAISE RET`
- **adjust** (3) — Bluetooth + media; activated by holding both lower and raise simultaneously (`ZMK_CONDITIONAL_LAYER`)

Key behaviors defined in `base.keymap`:
- **HRMs** via `MAKE_HRM` macro (uses zmk-helpers `ZMK_HOLD_TAP`): `hml` (left-hand), `hmr` (right-hand). Balanced flavor, 280ms tapping term, 150ms require-prior-idle, positional hold-trigger on release.
- **Magic shift** (`MAGIC_SHIFT`, right thumb): tap after alpha → key repeat; tap after other → sticky shift; double-tap/shift+tap → caps-word; hold → shift. Uses zmk-adaptive-key.
- **Nav cluster hold-taps** (`NAV_LEFT/RIGHT/UP/DOWN/BSPC/DEL`): tap for normal key, long-tap for home/end/doc-start/doc-end/word-delete.
- **CTL_SPC** (left thumb): tap → space, hold → Ctrl.

Combos are in `config/combos.dtsi` using `ZMK_COMBO` macros from zmk-helpers. Key positions use label aliases from `zmk-helpers/key-labels/34.h` (e.g. `LT0`–`LT4`, `LM0`–`LM4`, etc.).

## Dependency pinning

`config/west.yml` controls everything:
- `defaults.revision: v0.3` — applies to all urob-remote modules (adaptive-key, auto-layer, helpers, tri-state)
- `zmk` is explicitly pinned to `revision: v0.3.0` on the zmkfirmware remote
- `zephyr` is overridden to `v3.5.0+zmk-fixes` on the urob remote (ZMK v0.3.0 requires this exact fork)
- `urchin-zmk-module` and `nice-view-battery` track `main`

**Do not upgrade the `flake.nix`/`flake.lock`** when bumping ZMK unless Zephyr's major version also changes — the Nix SDK must match the Zephyr version. The current flake targets Zephyr SDK 0.16.9 / Zephyr 3.5.

## Adding or changing behaviors

ZMK helper macros used throughout:
- `ZMK_HOLD_TAP(name, ...)` — define a hold-tap behavior
- `ZMK_MOD_MORPH(name, ...)` — define a mod-morph
- `ZMK_COMBO(name, binding, positions, layers, term_ms, idle_ms)` — define a combo
- `ZMK_LAYER(name, bindings)` / `ZMK_BASE_LAYER(name, LT, RT, ...)` — define a layer
- `ZMK_ADAPTIVE_KEY(name, ...)` — from zmk-adaptive-key module

All macros expand to devicetree nodes. Include `zmk-helpers/helper.h` and the appropriate key-label header (`key-labels/34.h` for 34-key boards).
