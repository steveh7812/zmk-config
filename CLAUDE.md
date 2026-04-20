# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Personal ZMK firmware config for a Corne split keyboard in a **dongle-central** topology:
- **Central (dongle):** Seeeduino XIAO BLE, optionally with a Prospector-style dongle screen driven by the `janpfischer/zmk-dongle-screen` module.
- **Peripherals:** two nice_nano_v2 halves, each paired with a nice_view display.
- Both halves are peripherals — there is no "left-is-central" build. The dongle must be running for the keyboard to function over BLE.

## Build

Firmware is built exclusively via GitHub Actions (`.github/workflows/build.yml` invokes `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`). There is no local build command — push to GitHub and download the `firmware.zip` artifact.

The build matrix lives in `build.yaml`. Current outputs:
- `corne_peripheral_left` / `corne_peripheral_right` — nice_nano_v2 + nice_view, split with `ZMK_SPLIT_ROLE_CENTRAL=n` forced via `cmake-args` (overrides the Kconfig default, since the shield Kconfig would otherwise make left central).
- `corne_central_left` — nice_nano_v2 build kept as a fallback central if the dongle dies.
- `corne_central_dongle` — XIAO BLE dongle, with a `dongle_screen` variant that layers in the zmk-dongle-screen shield.
- `settings_reset` — flashed to clear BLE pairings on either nice_nano_v2 or XIAO BLE.

ZMK revision is pinned to `v0.3` in `config/west.yml`; the dongle-screen module tracks `main`.

## Keymap architecture

Four layers: `default` (0), `lower` (1), `raise` (2), `adjust` (3). Lower/raise are momentary via `&mo`; adjust is reached by a combo of the two thumb keys (positions 37+40), or via left/right thumb + outer-corner combos.

**Important — there are two keymap files and they must stay in sync:**
- `config/corne.keymap` — the user-config keymap at the top level (what ZMK expects for user builds).
- `config/boards/shields/corne/corne.keymap` — a shield-local copy that additionally defines layer display names (`Base`, `Lower`, `Raise`, `Adjust`) used by nice_view / dongle screen, plus `#define`s for layer indices.

If you edit one, mirror the change into the other. The shield copy is the more complete version; prefer editing it first and porting the behavior changes back. (The bootloader/reset binding on the adjust layer also differs between them — review carefully.)

## Shield layout (`config/boards/shields/corne/`)

- `corne.dtsi` — common matrix transform + kscan stub. All four overlays `#include` this.
- `corne_central_left.overlay` / `corne_peripheral_{left,right}.overlay` — real kscan GPIO matrix bindings for the nice_nano_v2 halves.
- `corne_central_dongle.overlay` — uses `zmk,kscan-mock` (no physical switches on the dongle).
- `Kconfig.defconfig` sets `ZMK_SPLIT_ROLE_CENTRAL=y` by default whenever either central shield is selected; the peripheral builds override this via `cmake-args` in `build.yaml` rather than via their own `.conf` files. Don't add `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n` to the peripheral `.conf` files — the `cmake-args` path is the intentional mechanism.

## Common tweaks

- Keymap changes → edit both keymap files, push, download artifact for the peripheral halves only (central doesn't have keys). Rebuilding the dongle is only needed for split/central-side behavior changes.
- Dongle screen tuning → `CONFIG_PROSPECTOR_*` knobs in `config/boards/shields/corne/corne_central_dongle.conf` (currently commented).
- BLE pairing wiped → flash the `settings_reset` artifact for the relevant board, then re-pair.
