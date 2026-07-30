# Building / modifying this channel

Notes for rebuilding the WAD from the assets in this repo, and the non-obvious things that
cost time to work out. Also serves as the description of what was changed in the GPL'd
components this channel is derived from.

## What's in a channel WAD

A WAD is a container: certificate chain, ticket, TMD, then numbered **contents**.
This channel has three:

| content | size | what it is |
|---|---:|---|
| 0 | 627,584 | `00000000.app` — the banner archive |
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

| | format | size | bytes |
|---|---|---|---:|
| banner `Background` | RGB565 | 512×332 | 340,032 |
| banner `Logo` | RGBA8 | 456×128 | 233,536 |
| banner `Controller` | RGBA8 | 256×136 | 139,328 |
| banner `Stripe` | CI8, 33-colour palette | 830×200 | 166,592 |
| banner `TitleBar` | CI8, 28-colour palette | 104×64 | 6,816 |
| banner `Bars` | RGB5A3 | 8×20 | 384 |
| banner `Fawful01–10` | CI8, 26-colour shared palette | 128×128 | 16,544 ea |
| banner `AWars01–10` | CI8, 27-colour shared palette | 120×120 | 14,560 ea |
| banner `Samus01–10` | CI4, 15-colour shared palette | 120×120 | 7,328 ea |
| banner `Lucas01–08` | CI4, 14-colour shared palette | 120×120 | 7,328 ea |
| icon everything | RGB5A3 | 256×96, 120×48, 16×32 (Mario), 16×36 (Luigi), 16×16 | |

**Keep both dimensions a multiple of 4.** GX tiles RGB5A3/RGB565/RGBA8 in 4×4 blocks;
**CI8 is 8×4 and CI4 is 8×8**, so palettized textures want a width that is a multiple of 8.
`TPL.FromImage` will happily accept 15×36 and write `width=15` into the header.
(`Stripe` at 830×200 is inherited and renders fine, so this is a rule to follow rather
than a proven hard failure — but don't add new violations.)

Format guidance: **RGB565 for opaque images** (6 bits of green, strictly better than
RGB5A3 when you don't need alpha — check that the source has no partial-alpha pixels
first), RGBA8 where you need smooth alpha, and **CI4/CI8 for anything with few colours**.

## Banner size limit — read this before adding art

**Total uncompressed banner memory across all installed channels is a hard constraint.**
Exceed it and the Wii Menu slows to a crawl and then hard-freezes while you page the
channel grid with `+`/`-`. It is not a brick — power-cycling recovers — but it is fatal to
usability, and it is *not* detectable by any structural check.

The failure needs two large banners installed together. Original v1 measurements:

| installed pair | uncompressed banners | freeze |
|---|---|---|
| FCEUGX + Snes9xGX | 1.39 + 2.57 MB | no |
| FCEUGX + mGBA GX v1 | 1.39 + 2.57 MB | no |
| Snes9xGX + mGBA GX v1 | **2.57 + 2.57 MB** | **yes** |

Roughly: one ~2.5 MB banner is survivable, two are not. This channel now ships at
**1,364,864 bytes uncompressed**, just under FCEUGX's 1,388,896 — pick a budget like that
for anything new. Note this is not really a defect in any one WAD: Snes9xGX is also
2.57 MB, so *any* second Tantric-style channel would have triggered it.

### Halving a banner without touching the art

Game sprite art has very few colours, so palettizing is close to free. Every character
sprite here has **under 32 colours**, which is why the v2 banner is 47% smaller than v1
while every sprite decodes **pixel-identical**:

- Dimensions stay the same, so **no `brlyt` pane geometry or `brlan` edits are needed** —
  texture format is invisible to the layout.
- Use **one shared palette for all frames of a character**. They are frames of the same
  sheet, so the union still fits in 16 (CI4) or 256 (CI8) entries. This makes `RLTP` frame
  swapping safe by construction: whichever palette the runtime binds, the pixels match.
- Palette entries are RGB5A3. Collapse every fully transparent pixel onto one entry.
- Verify by decoding the result back (`TPL.Load(...).ExtractTexture()`) and comparing
  every pixel against the original. Abort the build on any mismatch.

Cropping transparent padding is a much weaker lever here (344 KB versus 866 KB) *and* it
forces pane geometry edits, since the panes use `origin=Center` with full 0→1 texcoords.
Palettize first; crop only if still over budget.

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
with `[byte[]]` at the call site. PowerShell also wraps `byte[]` and `Join-Path` results in
`PSObject`, which reflection `Invoke` refuses to convert — prefer libWiiSharp's file-path
overloads (`U8.Load(string)`, `TPL.Load(string)`, `ReplaceFile(int, string, bool)`).

**`libWiiSharp.Lz77` is not reliable on this data.** Its methods are instance, not static,
and `Decompress(inFile, outFile)` produced a file of the correct length that was **entirely
zero-filled** — which then failed `U8.Load` with "Invalid Magic!". Write your own type-0x10
LZ77 codec instead (4096-byte window, match length 3–18, flag byte MSB-first, 1 = back
reference); a greedy encoder with a 3-byte-prefix hash chain round-trips Tantric's banner
byte-exact and lands within 0.4% of the original tooling's compressed size.

The layout inside `banner.bin` / `icon.bin` is: `IMD5` header (32 bytes), then the ASCII
magic **`LZ77`** (4 bytes), then the `0x10` stream. So the compressed data starts at offset
**36**, while the IMD5 length and MD5 cover everything from offset 32 (magic included).

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
NAND blocks  13
Looks for    apps/mGBAGX/boot.dol   on SD or USB
```

## Banner sound

`sound.bin` is a `BNS`: NGC-DSP ADPCM, mono, looping, **32 kHz**, 480,000 samples (15.000 s).

32 kHz is the Wii DSP's native mixer rate. CustomizeMii will happily carry a 44.1 kHz
source straight through into the BNS — Tantric's two channels both use 22.05 kHz, so
resample before converting rather than leaving it at 44.1.

The BNS `INFO` chunk stores the sample rate as a `u16` at offset `+12` from the chunk
start. Sample count divided by ADPCM data bytes is always exactly **1.75** (14 samples per
8-byte frame); if you compute a different ratio you have mis-read the codec field.

---

#### AI Disclosure

The entirety of *this* `.md` file was generated using Claude Code.
