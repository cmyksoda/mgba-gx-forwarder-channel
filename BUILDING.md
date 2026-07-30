# Building / modifying this channel

Notes for rebuilding the WAD from the assets in this repo, and the non-obvious things that
cost time to work out. Also serves as the description of what was changed in the GPL'd
components this channel is derived from.

## What's in a channel WAD

A WAD is a container: certificate chain, ticket, TMD, then numbered **contents**.
This channel has three:

| content | size | what it is |
|---|---:|---|
| 0 | ~1.15 MB | `00000000.app` — the banner archive |
| 1 | 138,752 | NAND loader stub. `BootIndex` points here. Unmodified, and byte-identical to the one in Tantric's FCEUGX and Snes9xGX channels (`e99125ee169e…`). |
| 2 | 934,656 | The forwarder — scans SD/USB for `boot.dol` and launches it. Patched, see below. |

Content 0 is a U8 archive holding `banner.bin`, `icon.bin` and `sound.bin`.
`banner.bin` and `icon.bin` are themselves U8 archives (with an `IMD5` header,
**LZ77-compressed**) laid out as:

```
arc/
  anim/   banner_Start.brlan, banner_Loop.brlan     (icon: icon.brlan)
  blyt/   banner.brlyt                              (icon: icon.brlyt)
  timg/   *.tpl
```

Because they're compressed, the raw size in the app and the size IMET records differ —
IMET stores the **uncompressed** size. A mismatch there is normal, not corruption.

## Tools

- **Benzin 2.1.12BETA** — converts `.brlyt`/`.brlan` to editable XML and back.
- **libWiiSharp 0.2.1** (ships inside CustomizeMii) — drive it directly to build WADs.
  It is **x86-only**; load it from 32-bit PowerShell
  (`%WINDIR%\SysWOW64\WindowsPowerShell\v1.0\powershell.exe`) or it fails with
  "incorrect format".
- **CustomizeMii 3.1.1** — optional. Fine for simple texture swaps; painful once
  texture names change, since every TPL must be re-added by hand.

## Editing layouts and animations

```
benzin r banner.brlyt banner.xmlyt      # rip to XML
benzin m banner.xmlyt banner.brlyt      # pack back
```

**Benzin dispatches on file extension.** A `.brlan` must become **`.xmlan`**, not
`.xmlyt` — feed it the wrong extension and it dies with `Couldn't get tree!` even though
the XML is perfectly valid. Both are provided in `layout/`.

Benzin recomputes all section offsets and sizes on pack, so renaming things or adding
keyframes is safe. Round-trips are byte-exact.

### Things that are not where you'd expect

- **On-screen banner text** (`mGBA GX`) lives in the brlyt's `txt1` pane as **UTF-16
  Big Endian**, and Benzin writes it as a raw hex blob. Searching for the string finds
  nothing. The buffer is a fixed **48 bytes** (`<length>0030-0030</length>`); keep
  replacements padded to exactly 96 hex characters and nothing else needs touching.
- **The header and bar colours are not in the textures.** Those textures are greyscale;
  the colour comes from `forecolor`/`backcolor` on the material — white pixels map to
  `forecolor`, black to `backcolor`. Only `BarsMaterial` and `TitleBarMaterial` are
  tinted. Pane vertex colours are all `ff`.
- **Pane names are a fixed 16-byte field** that overflows into the 8-byte `userdata`
  field. `{Name}01PictureA` = name length + 10, so character names must be ≤ 6 chars to
  stay inside. Material names get 20 bytes.
- **`RLTP` frame swapping indexes the brlan's own `timg` list**, not the brlyt's `txl1`.
  `<data1>` is the frame, `<data2>` is a hex index into that list.

### The icon walk cycle

`framesize` is 1200. Mario and Luigi each use an 8-frame cycle at 8 ticks per frame
(64-tick stride), advancing 0.3 px/frame — the same pace as the karts in the original
Snes9xGX icon.

```
Mario01Pane  X Translation  frame 0 = -140  ->  frame 1200 = 220
Luigi01Pane  X Translation  frame 0 = -160  ->  frame 1200 = 200   (20px behind = ~4.5px gap)
Mario01Mat   RLTP  indices 0x02..0x09,  150 keys
Luigi01Mat   RLTP  indices 0x0A..0x11,  150 keys, phase-shifted 4 frames
```

Luigi's pane sits at `y = -16` and Mario's at `y = -18` so their feet share a baseline
at −34 (Luigi's sprite is 4px taller).

## Textures

| | format | size |
|---|---|---|
| banner `Background` | RGBA8 | 512×332 |
| banner `Logo` | RGBA8 | 456×128 |
| banner `Controller` | RGBA8 | 256×136 |
| banner `Stripe` | RGB5A3 | 830×200 |
| banner `TitleBar` | RGB5A3 | 104×64 |
| banner `Bars` | RGB5A3 | 8×20 |
| banner characters | RGB5A3 | 128×128 (Fawful), 120×120 (others) |
| icon everything | RGB5A3 | 256×96, 120×48, 16×32 (Mario), 16×36 (Luigi), 16×16 |

**Keep both dimensions a multiple of 4.** GX tiles these formats in 4×4 blocks.
`TPL.FromImage` will happily accept 15×36 and write `width=15` into the header.
(`Stripe` at 830×200 is inherited and renders fine, so this is a rule to follow rather
than a proven hard failure — but don't add new violations.)

Use RGBA8 for smooth gradients and RGB5A3 for pixel art; RGB5A3 is 5 bits per channel
and will band a gradient badly.

## Rebuilding the WAD

**Never build `banner.bin` or `icon.bin` with `U8.FromDirectory`.** It writes garbage
parent indices into directory nodes (`arc` gets 641 instead of 0, everything under it
642 instead of 1). The result passes every obvious check — valid IMD5, correct LZ77
magic, 32-byte aligned data, all texture references resolving — and then **bricks the
System Menu**: black screen after the Health & Safety screen.

Instead, `U8.Load` the existing archive and mutate it: `RenameNode`, `ReplaceFile`,
`AddFile`, `RemoveFile`. Those preserve the tree correctly. Reuse existing file slots
where you can and `AddFile("/arc/timg/Name.tpl", bytes)` for the rest. Set
`Lz77Compress = true` before `ToByteArray()` to match the original scheme.

Note that renaming a node does **not** re-encode its pixels — if you edit a PNG, you must
explicitly `ReplaceFile` that node or the WAD keeps the old image.

Other API notes: `ChangeChannelTitles()` takes exactly **one** string, which it fans out
to all 8 language slots — passing 8 throws. And in PowerShell, `return $bytes` from a
function unrolls the array and binds to the wrong overload; use `return ,$bytes` and cast
with `[byte[]]` at the call site.

## Verify before installing

Parse the U8 node table (12 bytes per node: type, nameOffset(3), dataOffset(4), size(4))
and assert every **directory** node's parent index — root and `arc` → 0, everything under
`arc` → 1. This is the check that catches the brick above; nothing else does.

Then confirm: IMD5 valid, every `.tpl` referenced by the brlyt/brlan exists in `timg`,
every material a brlan animates exists in the brlyt, and all texture dimensions are
multiples of 4.

Test in Dolphin before hardware. If a bad banner does take out Dolphin's menu, the
emulated NAND is at `%APPDATA%\Dolphin Emulator\Wii\` — move
`title\00010001\47424758` and `ticket\00010001\47424758.tik` aside and it boots again.

## Changes made to the forwarder

Content 2 is Tantric's forwarder taken from the **FCE Ultra GX** channel WAD
(<https://github.com/dborth/fceugx>, GPL) with three in-place byte edits. The file size is
unchanged at 934,656 bytes.

| offset | original | change |
|---|---|---|
| `0x081700` | 4:3 splash PNG, 165,683 b | replaced with `splash/Splash4-3_final.png`, null-padded |
| `0x0A9E40` | 16:9 splash PNG, 157,270 b | replaced with `splash/Splash16-9_final.png`, null-padded |
| `0x0D369D` | `fceugx` | `mGBAGX` (same 6 bytes, in the string `%s:/apps/%s/boot.dol`) |

Replacement PNGs must be **≤** the original byte length; the remainder is zero-filled.
libpng stops at the `IEND` chunk so the padding is ignored. Both splashes are 640×480
colortype 2 (RGB, no alpha).

The 16:9 splash is the 4:3 art squashed to 75% width and centred, with the 80px side bars
filled by per-row edge replication. That pre-compression cancels out the Wii stretching a
640×480 framebuffer to widescreen.

Nothing else in the forwarder was touched — it still carries libfat, libntfs-3g and
libpng, and still scans both SD and USB.

## Channel properties

```
Title ID     0001000147424758  ("GBGX")
Boot IOS     58
Region       Free
NAND blocks  18
Looks for    apps/mGBAGX/boot.dol   on SD or USB
```

---

#### AI Disclosure

The entirety of *this* `.md` file was generated using Claude Code.
