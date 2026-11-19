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
Stage pinned at `11db3fbb52`. The build leaves about 56 KiB of firmware flash free; pixel and
audio buffers additionally use heap.

Tested on the device, with the same board and display code on an earlier `main`: boot, USB
REPL and CIRCUITPY storage, resets and both ways into the bootloader, the display bus and the
display's colours and orientation, rendering in both Stage and Picogame, I2C and ADC, all buttons,
vibration and PWM tones. The original bootloader stayed unchanged. Not yet run on this exact
build, and audio playback hasn't been validated.

The firmware's primary licence is MIT; see the repository's [LICENSE](../../../../LICENSE)
and individual file notices for upstream and third-party terms. The request concerns an
open-source firmware port running on existing commercial hardware.
