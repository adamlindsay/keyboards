---
name: qmk-framework16
description: Use when building, compiling, or flashing QMK firmware for Framework Laptop 16 keyboard, macropad, or numpad. Use when user mentions Framework 16 keyboard customization, keymaps, LED animations, or firmware updates.
---

# QMK Framework 16 Build and Flash

Build and flash custom QMK firmware for Framework Laptop 16 input modules on NixOS.

## Where things live

| Path                                              | Role                                                       |
| ------------------------------------------------- | ---------------------------------------------------------- |
| `~/projects/adamlindsay/keyboards/qmk_userspace/` | Keymap and flash helper — the source of truth              |
| `~/projects/adamlindsay/keyboards/qmk_firmware/`  | Untouched `FrameworkComputer/qmk_firmware` at tag `v0.3.1` |

- Keymap: `qmk_userspace/keyboards/framework/ansi/keymaps/custom/keymap.c`
- Flash helper: `qmk_userspace/flash-framework.sh`
- `qmk_userspace/` is a local git repo with no remote (`main`).
- `qmk_firmware/` keeps `upstream` = FrameworkComputer/qmk_firmware and `origin` =
  adamlindsay/qmk_firmware, but the fork is **not** the keymap location any more. Builds
  read the keymap out of `qmk_userspace/` and never need a fork change. Any
  `keyboards/framework/ansi/keymaps/adamlindsay/` still tracked in `qmk_firmware/` is a
  stale leftover from the fork (`adamlindsay` was the keymap's old name; the userspace copy
  is now `custom`) and is ignored by the build.

## Targets

| Target | Module | Backlight | Keys |
|--------|--------|-----------|------|
| `framework/ansi` | Keyboard | RGB/White | 78 |
| `framework/iso` | Keyboard | White | 79 |
| `framework/jis` | Keyboard | White | 83 |
| `framework/numpad` | Numpad | White | 21 |
| `framework/macropad` | Numpad | RGB | 24 |
| `framework/copilot` | Keyboard | - | - |

Adam's keyboard is ANSI.

## Build

Two things about v0.3.1 make the obvious command wrong, so the build goes through
`builddefs/build_keyboard.mk` directly:

1. **v0.3.1 predates userspace support.** Its `Makefile` only globs
   `keyboards/*/keymaps/*` and `layouts/*`, so `make framework/ansi:custom` is read as
   "keyboard `framework/ansi`, keymap `advanced`, target `custom`" and then fails with
   `No rule to make target`. Passing `KEYMAP_PATH` and `KEYMAP_C` on the make line bypasses
   the lookup: `KEYMAP_C` is what `quantum/keymap_introspection.c` includes, and
   `KEYMAP_PATH` is added to `VPATH`.
2. **nixpkgs `qmk` is 1.2.0 on Python 3.14**, which dropped `ast.Num` — used by v0.3.1's
   `lib/python/qmk/math.py`. The CLI has to run on Python 3.12.

Once per shell, shim the QMK CLI onto Python 3.12 (the shim adds the nix-provided
`qmk_cli` package to `sys.path`, so `python3` inside the nix-shell must be the 3.12 one):

```sh
QMK_SITE=$(ls -d "$(nix eval --raw nixpkgs#qmk)"/lib/python3.*/site-packages | head -1)
cat > /tmp/qmk312 <<EOF
#!/usr/bin/env bash
exec python3 -c "import sys; sys.path.insert(0, '$QMK_SITE'); from qmk_cli.script_qmk import main; sys.exit(main())" "\$@"
EOF
chmod +x /tmp/qmk312
```

Then, from `~/projects/adamlindsay/keyboards/qmk_firmware`:

```sh
KM=$PWD/../qmk_userspace/keyboards/framework/ansi/keymaps/custom

nix-shell -p "(python312.withPackages (ps: with ps; [ appdirs argcomplete colorama dotty-dict hid hjson jsonschema milc platformdirs pygments pyserial pyusb pillow ]))" \
  gcc-arm-embedded gnumake --run "
    QMK_HOME=$PWD QMK_BIN=/tmp/qmk312 make -j\$(nproc) -r -R -f builddefs/build_keyboard.mk \
      KEYBOARD=framework/ansi KEYMAP=custom KEYMAP_PATH=$KM KEYMAP_C=$KM/keymap.c SILENT=false"
```

Success ends with `Creating UF2 file for deployment: .build/framework_ansi_custom.uf2`
followed by `Copying framework_ansi_custom.uf2 to qmk_firmware folder`. The `-r -R` and
`SILENT=false` are required, not cosmetic: without them make's implicit rules and silent mode
mangle the direct `build_keyboard.mk` invocation.

Confirm the keymap came from the userspace copy — this must print the `qmk_userspace` path:

```sh
grep -o 'DKEYMAP_C=[^ ]*' .build/obj_framework_ansi_custom/cflags.txt | head -1
```

Output: `framework_ansi_custom.uf2` in the `qmk_firmware/` root.

In-tree keymaps still build the normal way, e.g. `make framework/ansi:default`.

## Flash

### flash-framework.sh (preferred)

Sends the same USB HID bootloader command the web configurator uses, then copies
the firmware to the RP2040 mass storage device automatically. It lives in
`qmk_userspace/`; run it against the `.uf2` the build just produced:

```sh
cd ~/projects/adamlindsay/keyboards/qmk_firmware
../qmk_userspace/flash-framework.sh                               # defaults to framework_ansi_custom.uf2
../qmk_userspace/flash-framework.sh framework_ansi_custom.uf2   # explicit file
```

Works with both stock and custom firmware (the factory HID handler is in shared keyboard code).
Uses direct hidraw writes (no external Python packages beyond stdlib).
Auto-detects the correct hidraw device via sysfs (interface 1 = raw HID / VIA).

### FN+Backspace

Press FN+Backspace (mapped to `QK_BOOT`) to enter bootloader manually. Then copy
the `.uf2` to the mass storage device that appears.

### Hardware method (fallback)

If the keyboard is completely unresponsive:

1. Pull the keyboard module partially out.
2. Hold both ALT keys while reinserting it.
3. Copy the `.uf2` to the mass storage device that appears.

For macropad/numpad: hold 2+6 keys instead of ALT keys.

Recovery: reflash stock firmware from the upstream repo's releases page. Cannot be bricked.

## Hardware

- Microcontroller: RP2040
- RGB driver: IS31FL3743A (ANSI keyboard, 97 LEDs)
- Flashing: UF2 via mass storage (no special programmer needed)

## Pulling Upstream Updates

The keymap is out of tree, so an upstream bump is just re-pointing the checkout:

```sh
cd ~/projects/adamlindsay/keyboards/qmk_firmware
git fetch upstream --tags
git checkout v<new-tag>
```

Rebuild with the same `build_keyboard.mk` line — `KEYMAP_PATH`/`KEYMAP_C` are absolute, so
the userspace keymap carries over unchanged. Only touch the userspace keymap if upstream
renames a keyboard-level header it uses (`framework.h`).

## Raw HID Communication

The keyboard exposes 4 HID interfaces. Interface 1 (`/dev/hidraw1` typically) is the
raw HID / VIA interface used for host-to-keyboard communication.

Protocol: 32-byte reports. Byte 0 = command ID, rest = payload.

| Command | Byte 0 | Byte 1 | Effect |
|---------|--------|--------|--------|
| Bootloader | `0x0B` | `0xFE` | Enters RP2040 UF2 mass storage mode |

The `via_command_kb` hook in `factory.c` intercepts command `0x0B` before VIA
processes it, routing to `handle_factory_command`. Custom commands can be added here.

Host-side writes: use direct hidraw (Python `os.open`/`os.write`). The `hidapi`
Python library fails to open these devices despite correct permissions.

## Common Mistakes

- Editing the keymap inside `qmk_firmware/` — the build reads `qmk_userspace/`, so such an
  edit silently does nothing
- Running `make framework/ansi:custom` against v0.3.1 — it parses `custom` as a
  target and dies; use the `build_keyboard.mk` line above
- Using the nixpkgs `qmk` CLI as-is on Python 3.14 — `ast.Num` is gone, so v0.3.1's CLI
  must run under `python312` (the `/tmp/qmk312` shim)
- Forgetting `SILENT=false` or `-r -R` on the direct `build_keyboard.mk` invocation
- Building from the default branch instead of a release tag (keyboard definitions missing on default branch)
- Using bare `nix-shell -p qmk` without the extra Python packages (fails on missing `appdirs`)
- Flashing without an external keyboard connected
- Forgetting `git submodule update --init --recursive` after cloning or switching tags
- Using Python `hidapi` library for HID writes -- use raw hidraw `os.write` instead
