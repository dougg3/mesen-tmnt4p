# mesen-tmnt4p

mesen-tmnt4p is a fork of [Mesen Community Edition](https://github.com/nesdev-org/MesenCE) that adds a few extra capabilities to its emulated SNES, making the [TMNT IV: Turtles in Time 4-player hack](https://github.com/dougg3/turtles-in-time-snes-4-players) possible.

With this hack, the emulated SNES behaves like a normal SNES until the hacked Turtles in Time ROM switches the extensions on.

## AI Disclosure

This hack was developed with the assistance of Claude Opus 5.5. Please do not bother the upstream Mesen developers about any of these changes. This hack exists solely for the purpose of making it possible to play a modified SNES version of Turtles in Time that supports 4 simultaneous players.

## Extension specs

This hack works by adding some additional registers to the SNES in the `$21C0` to `$21CF` range. Registers `$21C0` through `$21C3` are used for accessing a new 16-bit address space (referred to below as the "extension port").

### New registers:

B bus registers (also reachable by DMA, with B-bus addresses `$C0`-`$C6`):

| Register | Access | Meaning |
|---|---|---|
| `$21C0` | write | extension port address, low byte |
| `$21C1` | write | extension port address, high byte |
| `$21C2` | write | data: writes the byte at the extension port address, then increments the address by 1 |
| `$21C3` | read | data: reads the byte at the extension port address, then increments the address by 1 |
| `$21C4` | write | control: bit 0 enables extended sprites, bit 1 raises per-line sprite limits, bit 2 enables second Mode 7 layer |
| `$21C5` | write | extra CPU time: invisible scanlines before NMI (0-255; starting from the next frame) |
| `$21C6` | write | extra vblank time: scanlines after NMI (0-255; starting from the next frame) |
| `$21CE`, `$21CF` | read | ID: `'S'` (`$53`), `'+'` (`$2B`). The game ROM can use this to see if the emulated SNES supports the extensions. |

### Extension port address space (byte addresses):

| Range | Contents |
|---|---|
| `$0000-$3FFF` | extra OBJ tile memory: 512 tiles of 4bpp (same format as VRAM) |
| `$4000-$421F` | extended OAM: entries 128-255, same layout as OAM (512 + 32 bytes) |
| `$4300-$433F` | extended attributes for all 256 OAM entries. 2 bits per entry. See below. |
| `$4400-$44FF` | extra OBJ palettes 8-15: 128 colors, little-endian BGR555 |
| `$4600-$468F` | second Mode 7 layer: `$4600` split line, parameter sets at `$4610` and `$4650`. See below. |

Writes are allowed at any time (no vblank restriction). The memories are cleared only by power-on. Reset clears registers `$21C4` through `$21C6`.

### Extended sprites (`$21C4` bit 0)

- Each line, OAM entries 0-127 are evaluated as usual (including the priority rotation), then the extended entries 128-255.
- Tiles are fetched in the same order as the SNES PPU (last sprite in range first), so between overlapping sprites the lower entry wins, and entries 128-255 are behind entries 0-127.
- An entry with attribute bit 0 = 1 takes its tile from the extra OBJ memory (tiles 0-511 in order; OBSEL's base and gap don't apply).
- An entry with attribute bit 1 = 1 uses a palette from 8-15 instead of 0-7. Color math treats palettes 12-15 like 4-7.

#### Sprite limits (`$21C4` bit 1)

On a normal SNES, up to 32 sprites in range are kept and up to 34 tiles fetched. When `$21C4` bits 0 and 1 are both 1, those limits are raised to 128 sprites and 256 tiles. (Bit 1 has no effect without bit 0 also set.) The range-over and time-over flags in `$213E` are set as usual when a limit is exceeded.

### Second Mode 7 layer (`$21C4` bit 2)

If bit 2 is set, in BG mode 7, a second layer is drawn from the same Mode 7 map and tiles as BG1, with its own transform, so two rotated/zoomed pictures can share a scanline.

- Drawn behind BG1, with BG1's main/sub screen enable, windows, color math, M7SEL (fill/large map/flips) and center (M7X/M7Y).
- Two parameter sets of `$40` bytes: set 0 (`$4610`) for lines < split (`$4600`), set 1 (`$4650`) for the others. A set: `+$00` A, `+$02` B, `+$04` C, `+$06` D, `+$08` H scroll,
  `+$0A` V scroll (16-bit little-endian, as `$211B-$211E`, `$210D/E`); `+$10` color override mask (bit *n*: color *n*, 0-15, comes from the set); `+$12` flags (bit 0 set enabled, bit 1 screen mirrored horizontally); `+$20` 16 colors (BGR555). Other colors come from CGRAM as for BG1.
- Sets written in vblank apply to the whole next frame.

### Extra time/overclocking (`$21C5`, `$21C6`)

These are the same invisible scanlines already supported by stock Mesen's overclock settings (Options > Emulation > Overclocking), but requested by the ROM, only while it needs them. The sound CPU does not run during the extra lines. These values are added to any existing overclock specified by the user.

## Lua additions

| Function | |
|---|---|
| `emu.getExtensionMemory()` | Gets the extension port's memories as one string, in port address order: extra OBJ tiles, extended OAM, attributes, extra palettes, second Mode 7 layer. |
| `emu.startVideoRecording(file)` / `emu.stopVideoRecording()` | Starts/stops recording AVI video (ZMBV, with sound), just like Tools > Video Recorder. |
| `emu.startMovieRecording(file)` | Records a Mesen movie (`.mmo`) from the current state. Call it from a CPU exec callback, like `emu.createSavestate`. |
| `emu.playMovie(file)` | Plays a movie recorded from a save state after the current frame, without a power cycle (which would unload the script). Returns false if the file doesn't exist. |
| `emu.stopMovie()` | Stops the movie being played or recorded. |
| `emu.isMoviePlaying()` | Returns true while a movie plays, and in the gap after `playMovie()` before playback starts at the end of the frame. |

## Switches

- `MESEN_TMNT4P_EXT=0` in the environment removes the extension port (open bus, nothing renders differently), for comparisons with upstream Mesen.
- `MESEN_TMNT4P_EMBED` (compile-time define): additional functionality for a front end that embeds the core and runs frames itself, useful for embedding in a browser window.
