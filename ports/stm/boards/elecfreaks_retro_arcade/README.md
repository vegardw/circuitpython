<!--
SPDX-FileCopyrightText: Copyright (c) 2026 Vegard Wærp
SPDX-License-Identifier: MIT
-->

# CircuitPython for ELECFREAKS Retro Arcade

An independent CircuitPython port for the
[ELECFREAKS Retro Arcade](https://wiki.elecfreaks.com/en/retroarcade/) handheld, using its
STM32F412RET6 and 320×240 ILI9341 display. This branch publishes the port source while the
USB allocation and upstream submissions are being prepared.

The port consists of four commits: the STM32F412Rx MCU variant, support for the stock
[uf2-stm32f bootloader](https://github.com/mmoskal/uf2-stm32f), the FSMC parallel display bus,
and this board definition. These are being prepared as separate upstream PRs. Other fixes merged upstream are inherited from the branch's base.

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
- The separate STM audio repair and PWM/DMA rewrite are not included. During development, the existing
  `audiopwmio` implementation hung on silent sources; such playback can stop USB until a
  hardware reset. Audio playback is outside this branch's validation.
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

The branch is based on Adafruit `main` at `8373f3be5d` and includes merged Stage support
at `11db3fbb52`. The dual-engine configuration passed build, boot, USB REPL, full-frame
and sprite rendering in both engines, `ugame` adapter checks and firmware flash readback.
The original bootloader was unchanged. Earlier hardware checks covered storage/reset paths,
I2C/ADC, all buttons, vibration, PWM tones, display colours and orientation; controls and
visual appearance were not manually rechecked on the latest configuration.

Audio playback remains outside this branch's validation. The engine/audio comparison used
a separate experimental audio-repair build. The current port build leaves about 57 KiB of
firmware flash free; pixel and audio buffers additionally use heap.

The firmware's primary licence is MIT; see the repository's [LICENSE](../../../../LICENSE)
and individual file notices for upstream and third-party terms. The request concerns an
open-source firmware port running on existing commercial hardware.
