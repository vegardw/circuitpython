# picogame-core: picogame helpers frozen for the Retro Arcade

A subset of [picogame-libs](https://github.com/MakerClassCZ/picogame-libs) release **0.3.2**
(`75dc151e88372a2715592b68b2657818d05f4d17`), frozen into the ELECFREAKS Retro Arcade firmware
on this branch. The files are unchanged copies; the licence is in [LICENSE](LICENSE) (MIT).

The board's CIRCUITPY drive has only 32 KiB, too little for picogame-libs (about 108 KB as
`.mpy`). Frozen modules take no space on CIRCUITPY and run from flash, which also saves RAM.
These 18 modules (about 49 KB) cover what most games on this console need:

| Group | Modules |
|---|---|
| Base | `picogame_game`, `picogame_input`, `picogame_clock`, `picogame_rand`, `picogame_math` |
| Text and UI | `picogame_font`, `picogame_ui`, `picogame_bitfont` |
| Sound | `picogame_audioout`, `picogame_synth`, `picogame_sfx` |
| Gameplay | `picogame_pool`, `picogame_anim`, `picogame_tiles`, `picogame_seq` |
| Art in code | `picogame_shapes`, `picogame_palette` |
| The console's accelerometer | `picogame_tiltpad` |

Other picogame-libs modules can be copied to `CIRCUITPY/lib` as `.mpy` when a game needs them.
Left out: USB host and add-on pads (no such hardware here), `picogame_save` (this board has no
`microcontroller.nvm`), the launcher, and the asset-streaming modules (no room for asset files).

This folder is meant to be replaced by a freezable subset folder in picogame-libs itself, if
one is added.

To regenerate from another release, copy the same files from that release's tag, with its
`LICENSE`, and update the version above.
