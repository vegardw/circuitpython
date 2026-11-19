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
this board definition, which waits for a USB allocation before it can be submitted upstream.

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
  effect in the CIRCUITPY root, not in `/lib`. To make room, `bitbangio`, `gifio`, `jpegio`, `pulseio`, `usb_midi`, `atexit`, `getpass`,
  `locale` and `warnings` are left out. This is specific to this branch; the upstream board PR
  doesn't freeze the helpers.
- picogame's helpers know this board: its button helper picks the board's profile by itself
  (the D-pad, A, B, and MENU as START), so no `settings.toml` line is needed. To steer by
  tilting as well, add this line to `settings.toml` on CIRCUITPY:

  ```toml
  PICOGAME_TILTPAD = "retro_arcade"
  ```

  The accelerometer zeroes at start, so hold the console the way you play when a game starts.
- Audio uses `main`'s `audiopwmio`, which includes the fix for the hang on silent sources
  ([#11889](https://github.com/adafruit/circuitpython/pull/11889)). Audio playback hasn't been
  validated on this branch.
- `alarm` is disabled because its wake flag conflicts with the bootloader's RTC register use.
- USB `1209:0001` is a development placeholder. A dedicated allocation is pending; this is
  not the final USB identity for a release or the upstream board submission.

## Build

Set up the normal STM build prerequisites described in [BUILDING.md](../../../../BUILDING.md),
including the Arm GNU toolchain and required Python packages. On Linux:

```sh
git clone --branch elecfreaks-retro-arcade --single-branch https://github.com/vegardw/circuitpython.git
cd circuitpython
make -C ports/stm fetch-board-submodules BOARD=elecfreaks_retro_arcade
make -C mpy-cross -j$(nproc)
make -C ports/stm BOARD=elecfreaks_retro_arcade -j$(nproc)
```

Output: `ports/stm/build-elecfreaks_retro_arcade/firmware.uf2`.

The firmware uses the existing `ARCADE-F4` UF2 bootloader and places its vector table at
`0x08010000`. CIRCUITPY occupies flash sectors 2–3, the area MakeCode uses for saved games;
its first format erases those saves. Preserve anything needed from the device before installing
CircuitPython. The bootloader itself is retained.

## Verification and licence

The branch is Adafruit `main` (last merged at `d6be0e7b39`) plus this board definition, with
Stage pinned at `11db3fbb52`. With the frozen picogame helpers, the build leaves about 35 KiB of
firmware flash free; pixel and audio buffers additionally use heap.

Tested on the device, with the same board and display code on an earlier `main`: boot, USB
REPL and CIRCUITPY storage, resets and both ways into the bootloader, the display bus and the
display's colours and orientation, rendering in both Stage and Picogame, I2C and ADC, all buttons,
vibration and PWM tones. The original bootloader stayed unchanged. With the frozen helpers:
boot, a picogame game (Picotris) using only the frozen modules, with sound effects, and tilt
steering in all four directions. The board's button profile and `retro_arcade` tilt preset, new
in picogame-libs 0.3.4, were tested on the device before that release: all seven buttons with no
`settings.toml` line, tilt in all four directions, and a picogame game played with both. Audio
playback beyond those sound effects hasn't been validated.

The firmware's primary licence is MIT; see the repository's [LICENSE](../../../../LICENSE)
and individual file notices for upstream and third-party terms. The request concerns an
open-source firmware port running on existing commercial hardware.
