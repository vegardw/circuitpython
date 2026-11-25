<!--
SPDX-FileCopyrightText: Copyright (c) 2026 Vegard Wærp
SPDX-License-Identifier: MIT
-->

# CircuitPython for ELECFREAKS Retro Arcade

An independent CircuitPython port for the
[ELECFREAKS Retro Arcade](https://wiki.elecfreaks.com/en/retroarcade/) handheld, using its
STM32F412RET6 and 320×240 ILI9341 display.

Most of the port is already in CircuitPython's `main`: the STM32F412Rx MCU variant
([#11896](https://github.com/adafruit/circuitpython/pull/11896)), support for the stock
[uf2-stm32f bootloader](https://github.com/mmoskal/uf2-stm32f)
([#11903](https://github.com/adafruit/circuitpython/pull/11903)) and the FSMC parallel display
bus ([#11908](https://github.com/adafruit/circuitpython/pull/11908)). This branch is `main` plus
this board definition, which is open upstream as draft
[#11915](https://github.com/adafruit/circuitpython/pull/11915) while it waits for a USB allocation.

## Current scope

- USB serial REPL, CIRCUITPY storage, display initialization and the board's
  [named pins](pins.c) for buttons, speaker, sensors and vibration motor.
- Stage and `ugame` are enabled and frozen from the merged
  [board support](https://github.com/python-ugame/circuitpython-stage/pull/21) in
  `python-ugame/circuitpython-stage`, pinned by the submodule. Stage uses 2× scaling by
  default (160×120); pass `scale=1` to `stage.Stage` for native 320×240.
- Picogame is also enabled, using the generic parallel-display backend. Both engines
  are available; Stage games and the `ugame` adapter remain supported. Picogame uses
  more heap for pixel buffers: an 8-row strip buffer is a useful starting point on this board.
- 18 of picogame's Python helper modules are frozen into the firmware, so games can use them
  without copying anything: the base, text and UI, sound, gameplay, art and accelerometer
  helpers from [picogame-libs](https://github.com/MakerClassCZ/picogame-libs) 0.3.4, its
  `frozen/core` folder made for this (list in
  [its README](https://github.com/MakerClassCZ/picogame-libs/blob/0.3.4/frozen/core/README.md)),
  pinned by the `frozen/picogame-libs` submodule. CIRCUITPY is too small
  for the whole library; copy other modules to `CIRCUITPY/lib` as `.mpy` when a game needs them.
  The frozen modules come before `/lib` in `sys.path`, so a newer copy of one of them only takes
  effect in the CIRCUITPY root, not in `/lib`. To make room, `bitbangio`, `gifio`, `jpegio`,
  `pulseio`, `usb_midi`, `atexit`, `getpass`, `locale` and `warnings` are left out. The upstream
  board PR does the same in its second commit.
- picogame's helpers know this board: its button helper picks the board's profile by itself
  (the D-pad, A, B, and MENU as START), so no `settings.toml` line is needed. To steer by
  tilting as well, add this line to `settings.toml` on CIRCUITPY:

  ```toml
  PICOGAME_TILTPAD = "retro_arcade"
  ```

  The accelerometer zeroes at start, so hold the console the way you play when a game starts.
- Audio uses `main`'s one-bit `audiopwmio`, which includes the fix for the hang on silent
  sources ([#11889](https://github.com/adafruit/circuitpython/pull/11889)). Picogame's sound
  effects play, but sound rough; other audio playback hasn't been validated.
- `alarm` is disabled because its wake flag conflicts with the bootloader's RTC register use.
- USB `1209:0001` is a development placeholder, not the final USB identity for a release or the
  upstream board submission. `1209:F412` has been requested from pid.codes
  ([#1379](https://github.com/pidcodes/pidcodes.github.com/pull/1379)) for this open-source
  firmware port on existing commercial hardware; it isn't allocated yet.

## Build

Set up the normal STM build prerequisites described in [BUILDING.md](../../../../BUILDING.md),
including the Arm GNU toolchain and required Python packages. On Linux:

```sh
git clone --branch elecfreaks-retro-arcade --single-branch https://github.com/vegardw/circuitpython.git
cd circuitpython
make -C ports/stm fetch-board-submodules BOARD=elecfreaks_retro_arcade
make -C ports/stm BOARD=elecfreaks_retro_arcade -j$(nproc)
```

The build makes `mpy-cross` itself if it is missing. Output:
`ports/stm/build-elecfreaks_retro_arcade/firmware.uf2`.

## Install

The firmware uses the console's existing `ARCADE-F4` UF2 bootloader, which stays as it is, and
places its vector table at `0x08010000`.

1. Connect the console by USB and press RST, so that the `ARCADE-F4` drive appears.
2. Copy `firmware.uf2` to that drive. The console restarts into CircuitPython, and the
   `CIRCUITPY` drive appears.

CIRCUITPY occupies flash sectors 2–3, the area MakeCode uses for saved games; its first format
erases those saves. Preserve anything needed from the device first. To go back to MakeCode,
copy a MakeCode `.uf2` to `ARCADE-F4` the same way; that replaces CircuitPython and CIRCUITPY.

- **Switching on with USB connected** opens the `ARCADE-F4` drive instead of CircuitPython: the
  bootloader waits there whenever USB is connected after a power-on. Press RST once to start
  CircuitPython. On battery, CircuitPython starts after about 3 seconds.
- **CIRCUITPY is small:** 32 KiB, with about 26 KiB free on a new drive. macOS adds a 4 KiB `._`
  file next to each file it copies there; `dot_clean -m /Volumes/CIRCUITPY` removes them.

## Verification and licence

The branch is Adafruit `main` (last merged at `7d4ca58b48`) plus this board definition, with
Stage pinned at `11db3fbb52`. With the frozen picogame helpers, the build leaves about 35 KiB of
firmware flash free; pixel and audio buffers additionally use heap.

Tested on the device on 2026-11-19, with `main` at `d6be0e7b39` and the same board and display
code: boot, USB REPL and CIRCUITPY storage, resets and both ways into the bootloader, the display bus and the
display's colours and orientation, rendering in both Stage and Picogame, I2C and ADC, all buttons,
vibration and PWM tones. Flash readbacks in earlier runs showed the original bootloader
unchanged. With the frozen helpers:
boot, a picogame game (Picotris) using only the frozen modules, with sound effects, and tilt
steering in all four directions. The board's button profile and `retro_arcade` tilt preset, new
in picogame-libs 0.3.4, were tested on the device before that release: all seven buttons with no
`settings.toml` line, tilt in all four directions, and a picogame game played with both. Audio
playback beyond those sound effects hasn't been validated. The merge of `main` at `7d4ca58b48`,
which brings two STM RTC fixes, has been built but not yet run on the device.

The firmware's primary licence is MIT; see the repository's [LICENSE](../../../../LICENSE)
and individual file notices for upstream and third-party terms.
