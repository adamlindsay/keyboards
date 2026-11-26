# Keyboards Project

## QMK Build
- Build is documented in the project skill: `.pi/skills/qmk-framework16/SKILL.md`
- The Framework 16 ANSI keymap (`custom`) and `flash-framework.sh` live in `qmk_userspace/`, not in `qmk_firmware/`
- Builds run against an untouched `FrameworkComputer/qmk_firmware` at tag `v0.3.1`; the keymap is passed in via `KEYMAP_PATH`/`KEYMAP_C`
- v0.3.1 predates userspace support, so `make framework/ansi:custom` does not work — invoke `builddefs/build_keyboard.mk` directly
- The `qmk` nixpkg alone is insufficient; extra Python packages (appdirs, etc.) are required, under `python312` (v0.3.1's CLI uses `ast.Num`, removed in 3.14)
- Must use release tags (e.g. v0.3.1), not the default branch which lacks keyboard definitions
- Working directory for builds: `qmk_firmware/`

## LED Index Mapping
- LED indices are NOT sequential by physical position
- Trace: LAYOUT macro key name → matrix position in ansi.h → LED index in g_led_config (ansi.c)
- Use `rgb_matrix_indicators_advanced_user` to overlay colors on top of animations

## Flashing
- `flash-framework.sh` in `qmk_userspace/` sends HID bootloader command and copies .uf2 automatically (run it against `qmk_firmware/framework_ansi_custom.uf2`)
- Uses direct hidraw writes (Python `os.write`), not the `hidapi` library (which fails to open)
- Auto-detects hidraw device via sysfs; interface 1 = raw HID / VIA
- RP2040 bootloader mounts as `RPI-RP2` label under `/run/media/`

## Git
- Stale `.git/index.lock` files appear frequently; safe to remove
- Upstream remote is FrameworkComputer/qmk_firmware; origin is the adamlindsay fork, which is no longer where the keymap lives — that is `qmk_userspace/`
- `upstream/master` is a stale 2023 QMK export and carries nothing — releases are tags (`v0.3.1`, 2025-07-21, still newest) and Framework's work sits on unmerged `fl16-*` topic branches. `fl16-power` (power/suspend) is the only actively developed one. Check `git tag --sort=-v:refname` and branch dates, never `master`
- This directory is the `adamlindsay/keyboards` repo (`origin`), and it tracks `AGENTS.md`, `.pi/skills/`, the `qmk_userspace/` keymaps, `flash-framework.sh`, `plans/`, and the CI workflow — so edits here are versioned and go through a PR like anywhere else. Only `qmk_firmware/` and `epomaker_qmk_firmware/` are out-of-repo checkouts (both git-ignored); they are the untouched upstream trees, not places to edit

## Framework 16 (ANSI)
- **Separation:** `qmk_userspace/keyboards/framework/ansi/keymaps/custom/keymap.c` is the source of truth; `qmk_firmware/` stays an untouched `FrameworkComputer/qmk_firmware` v0.3.1 checkout
- **Build:** Inside `qmk_firmware/`, `nix-shell -p "(python312.withPackages ...)" gcc-arm-embedded gnumake --run "QMK_HOME=$PWD QMK_BIN=/tmp/qmk312 make -j$(nproc) -r -R -f builddefs/build_keyboard.mk KEYBOARD=framework/ansi KEYMAP=custom KEYMAP_PATH=<userspace keymap dir> KEYMAP_C=<that dir>/keymap.c SILENT=false"` — `/tmp/qmk312` is the Python 3.12 shim described in the skill
- **Flash:** `qmk_userspace/flash-framework.sh framework_ansi_custom.uf2`, or FN+Backspace for the bootloader and copy the `.uf2` to the RP2040 mass storage device
- **RGB keys:** FN+W `RGB_TOG` is the only real off switch; FN+E/FN+D change effect and force RGB back on if it was off (`framework.c:278-284`); FN+R/F, T/G, Y/H, U/J are hue, sat, speed, brightness ±. Brightness 0 is not off — the animation runs and `rgb_matrix_indicators_advanced_user` still paints the F-row groups
- **`BL_*` keys are not what they sound like:** `keyboards/framework/rules.mk:12` enables the GP25 PWM white backlight for the whole family, but `framework.c:275-301` intercepts `BL_STEP` and repurposes it to step the **RGB matrix** value through 0/85/170/255. Only `BL_BRTG` reaches the white PWM. Do not add a white-backlight step key expecting it to work

## Epomaker Luma40
- **Separation:** `qmk_firmware/` is Framework 16 only. Luma40 builds from `epomaker_qmk_firmware/` (`carlosedp/qmk_firmware` `master`, untouched) with `QMK_USERSPACE` pointing at `qmk_userspace/` (`keyboards/epomaker/luma40/keymaps/planck/`)
- **Build:** Inside `epomaker_qmk_firmware/`, run the same `nix-shell` toolchain as Framework with `QMK_HOME=$PWD QMK_USERSPACE=$PWD/../qmk_userspace make -j$(nproc) epomaker/luma40:planck` (do not overwrite global `qmk config` user.qmk_home)
- **Bootloader & Flash:** Enter bootloader via `R4C1` (`SYS`) + `Tab` (`QK_BOOT`), holding `Tab` `(0,0)` while plugging in, or the PCB reset button. Mounts as `RDMCTMZT_DFU` (~106K USB mass storage under `/run/media/alindsay/`). Copy `epomaker_luma40_planck.bin` to the mounted drive and `sync`; it flashes and reboots automatically
- **VIA:** Disabled (`VIA_ENABLE = no`, `DYNAMIC_KEYMAP_ENABLE = no`) in `keymaps/planck/rules.mk` so `keymap.c` always takes effect directly without EEPROM overrides
- **Mode indicator:** Holding `R4C1` (`SYS`) lights `LED_USB_INDEX` (`5`, the `T` key) white in USB mode (`Q`/`W`/`E` blue for BLE1–3, `R` green for 2.4G) — normal `lib/rdmctmzt_common` behavior

## KPRepublic BM40 (rev1)
- **Build:** Uses `epomaker_qmk_firmware/` (`kprepublic/bm40hsrgb/rev1:planck`) with `QMK_USERSPACE` (`qmk_firmware/` fails on `avr-gcc` 16.2.0 / `dyn_serial.h`). Requires `LTO_ENABLE = yes` in `rules.mk` to fit the 28 KB ATmega32U4 flash
- **Bootloader & Flash:** Enter DFU (`03eb:2ff4`) via `SYS` + `Tab` (`QK_BOOT`), holding `Tab` `(0,0)` while plugging in, or the PCB reset button; flash with `nix-shell -p dfu-programmer --run "dfu-programmer atmega32u4 erase --force && dfu-programmer atmega32u4 flash qmk_userspace/kprepublic_bm40hsrgb_rev1_planck.hex && dfu-programmer atmega32u4 reset"`
