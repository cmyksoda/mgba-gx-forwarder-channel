# mGBA GX Channel Forwarder

## Screenshots

| 4:3 Icon | 4:3 Banner |
| ------------- | ------------- |
| <img width="640" height="478" alt="0000000100000002_2026-07-30_02-00-12" src="https://github.com/user-attachments/assets/ed1ceda5-fcef-49bb-b8c7-d08d02677a3d" /> | <img width="640" height="478" alt="0000000100000002_2026-07-30_02-00-26" src="https://github.com/user-attachments/assets/1766e59b-4658-4374-b675-0933bf3ecbf2" />


| 16:9 Icon | 16:9 Banner |
| ------------- | ------------- |
| <img width="834" height="456" alt="0000000100000002_2026-07-30_01-58-47" src="https://github.com/user-attachments/assets/b6f45811-2def-4784-b620-e398601608a6" /> | <img width="834" height="456" alt="0000000100000002_2026-07-30_01-58-57" src="https://github.com/user-attachments/assets/148f3f8a-6c91-4929-b686-d50b261116f7" />

## Requirements

- **IOS58** — update to System Menu 4.3, or use the [IOS58 Installer](https://wiibrew.org/wiki/IOS58_Installer).
- mGBA-GX installed to `apps/mGBAGX/boot.dol` on SD or USB.

| Title ID | Region | NAND Blocks |
|---|---|---|
| GBGX (0001000147424758) | Free | 13 |

## Install

> **Install BootMii and/or Priiloader first.** I took time to make sure that this `.wad` is safe, but one should always have protections in place in case of a banner brick.

1. Download the compiled `.wad` on the releases page.
2. Install using the WAD manager of your choice.
3. Ensure mGBA-GX is installed to `apps/mGBAGX/boot.dol` on SD or USB — that exact folder name.

## Uninstall

- Use the WAD manager you used to install the channel to uninstall it, or, delete it from the Wii system settings.

## Credits

### Inspiration/Channel forwarded to

- [**mGBA-GX**](https://github.com/nateynaate/mgba-gx) by nateynaate/daillou et al. Logo is designed by them as well.
- Based on **VBA GX** by **Tantric**
- Powered by [**mGBA**](https://github.com/mgba-emu/mgba) by **endrift** and contributors

### Channel Base

**Snes9x GX Channel**  

- **wilsoff**: coding
- **MrNick666**: artwork
- **Tantric**: forwarder and installer
- **svpe** and **megazig**: installer exploit

### Forwarder

**Taken from the FCE Ultra GX Channel**  

*Two PNG files and the path pointing towards the boot.dol have been changed.*
- **wilsoff**: coding
- **MrNick666**: artwork
- **Tantric**: forwarder and installer
- **svpe** and **megazig**: installer exploit

### Sprites:

**All sprites sourced from The Spriters Resource**  

*All characters are the intellectual property of Nintendo. Mario & Luigi sprites are also the intellectual property of AlphaDream. Advance Wars sprites are also the intellectual property of Intelligent Systems. Mother 3 sprites are also the intellectual property of HAL Laboratory and Brownie Brown.*
- All Mario & Luigi sprites ripped by A.J. Nitro
- Lucas sprites ripped by Smalls
- Orange Star Land Unit sprites ripped by Rogultgot
- Samus Fusion Suit sprites ripped by Rogultgot  

### Tools Used
- **CustomizeMii 3.1.1** by **Leathl**
- **libWiiSharp 0.2.1** by **Leathl**
- **Benzin 2.1.12BETA** by **SquidMan (Alex Marshall), comex, and megazig, © 2009 HACKERCHANNEL**

### Sound

The banner sound is [this chiptune loop](https://www.looperman.com/loops/detail/406626/chiptune-melody-loop-agrrx-128bpm-free-128bpm-8bit-chiptune-synth-loop) by user AGRRXEDM on Looperman.

### Other graphics

- The banner background is my own work.  
- Game Boy Advance logo is the intellectual property of Nintendo.  
- All other graphics not already mentioned are royalty-free with no attribution required.

## License

GPLv3 — see [`LICENSE`](LICENSE). This channel is derived from GPL'd work: the forwarder from
[FCE Ultra GX](https://github.com/dborth/fceugx) and the banner/icon from
[Snes9x GX](https://github.com/dborth/snes9xgx). Every modification is documented in
[BUILDING.md](BUILDING.md), which also covers rebuilding the WAD from the assets here.

## Repository contents

| path | |
|---|---|
| `banner/` | source PNGs for the banner textures |
| `icon/` | source PNGs for the icon textures |
| `layout/` | `.brlyt`/`.brlan` layouts and animations extracted from the released WAD, plus their Benzin XML sources |
| `splash/` | forwarder loading screens (4:3 and 16:9) |
| `BUILDING.md` | how to rebuild, and what was changed in the GPL'd components |

<sub><sup>There is one(1) unused silly texture leftover from the Snes9x GX forwarder, can you find it?</sup></sub>
