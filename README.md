# The CD-i toolkit guide

1. [What a CD-i program is](#1-what-a-cd-i-program-is)
2. [Setting up](#2-setting-up)
3. [Your first program](#3-your-first-program)
4. [The library](#4-the-library)
5. [Assets: the tools](#5-assets-the-tools)
6. [Making it fast](#6-making-it-fast)
7. [Testing in MAME](#7-testing-in-mame)
8. [Real hardware](#8-real-hardware)
9. [Porting a game](#9-porting-a-game)
10. [Limits and known issues](#10-limits-and-known-issues)

---

## 1. What a CD-i program is

The CD-i (Philips, 1991) is a 68000-family computer, a Philips SCC68070 at about 15 MHz with 1 MB of RAM, that runs the OS-9 operating system from ROM. A CD-i disc is a CD-ROM (in the "CD-i Bridge" format) whose volume names a program file; the player loads that program and runs it.

A program is an **OS-9 module**: 68000 code that is position-independent (it runs wherever OS-9 loads it), with its variables in a data area that OS-9 allocates and points register `a6` at. `toolchaincdi`'s GCC produces exactly that with `-mpcrel -ma6rel`, and its `elf2mod` turns the linked program into a module.

The hardware the library drives:

- **The video chip** (MCD212): two picture planes over a backdrop. The library uses plane A, 384 × 280 pixels in 256 colours (CLUT8), double-buffered. Colour 0 is see-through.
- **The audio processor**: plays CD-i ADPCM sound, either from memory (sound effects) or as it comes off the disc (music, speech). One thing at a time.
- **The CD drive**: delivers files in real time at 75 sectors a second (single speed). A "real-time file" interleaves video and audio sectors so they arrive just when they're needed.
- **The Digital Video Cartridge** (DVC, optional, fitted to many players): MPEG-1 video and MPEG audio decoders. Its picture appears behind plane A, wherever plane A shows colour 0.
- **The controller**: a pointing device (thumbpad, joystick or mouse) with two buttons. The library reads it as a joypad.

## 2. Setting up

You need:

- **The compiler**: toolchaincdi's `m68k-elfos9-gcc`, binutils, and `elf2mod`.
- **Python 3** with numpy and Pillow, and **ffmpeg**, for the asset tools.
- **MAME** to run programs (`cdimono1`, with the CD-i BIOS files). Recent MAME versions emulate the Digital Video Cartridge.

### Windows

Use WSL (Windows Subsystem for Linux):

1. In PowerShell, run `wsl --install` (this installs Ubuntu), then restart.
2. Open Ubuntu, go to the toolkit's folder (your C: drive is `/mnt/c`, so `cd "/mnt/c/Users/you/Documents/cdikit"`), and follow the Linux steps below.
3. MAME can stay the Windows version: the disc images you build are ordinary files in your Windows folder.

### Linux (and WSL)

```sh
sh setup/build-toolchain.sh                       # 15-40 minutes; installs to ~/cdi-toolchain
echo 'export PATH="$HOME/cdi-toolchain/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
sudo apt install python3-numpy python3-pil ffmpeg make
make                                              # the library and examples
```

The script fetches toolchaincdi and builds only what the toolkit uses: binutils, GCC (C) with libgcc, and elf2mod. On Ubuntu it installs what it needs with apt. If you already have GMP, MPFR and MPC somewhere, `GMP_PREFIX=/that/dir sh setup/build-toolchain.sh` uses them.

### Docker

If you'd rather not build anything, toolchaincdi publishes a ready-made compiler image. `setup/docker/Dockerfile` adds Python and ffmpeg to it:

```sh
docker build -t cdikit setup/docker
docker run --rm -v "$PWD":/work -w /work cdikit make
```

(The image is based on toolchaincdi's published one. The Dockerfile itself hasn't been tested yet.)

## 3. Your first program

`examples/hello/main.c`, in full:

```c
#include "cdi.h"

int main(void)
{
    if (gfx_init()) return 1;
    static const u32 colours[] = {
        0x000000,   /* 0: see-through */
        0x101838,   /* 1: background */
        0xFFFFFF,   /* 2: white */
        0xFFD000,   /* 3: yellow */
        0xFF4040, 0x40FF40, 0x4080FF,   /* 4..6: the box */
    };
    gfx_palette(colours, 0, 7);
    gfx_background(0, 1);           /* gfx_restore() puts colour 1 back */

    int x = 176, y = 160, c = 4;
    for (;;) {
        u16 pad = pad_read();
        if (pad & PAD_LEFT) x -= 4;
        if (pad & PAD_RIGHT) x += 4;
        if (pad & PAD_UP) y -= 4;
        if (pad & PAD_DOWN) y += 4;
        /* (keeping the box on the screen left out here) */
        if (pad_pressed() & PAD_B1) c = c == 6 ? 4 : c + 1;

        gfx_restore();              /* last frame's drawing away */
        gfx_label(0, 40, 40, "Hello, CD-i!", 3, 4);
        gfx_label(1, 40, 90, "Move the box with the controller;", 2, 1);
        gfx_label(2, 40, 102, "button 1 changes its colour.", 2, 1);
        gfx_rect(x, y, 32, 32, c);
        gfx_swap();
    }
}
```

Build it and run it:

```sh
cd examples/hello
make                       # -> hello.bin + hello.cue
mame cdimono1 -cdrom hello.cue
```

The CD-i's own menu comes up first (about 20 seconds in MAME). Click "Play CD-i" with button 1, and the program starts.

What `make` does (see the Makefile):

1. `m68k-elfos9-gcc -m68000 -O2 -mpcrel -ma6rel -ffreestanding -fno-builtin -fomit-frame-pointer -I../../lib -c main.c` compiles the program.
2. `m68k-elfos9-ld -q -T lib/os9.lds lib/crt0.o main.o lib/libcdi.a libgcc.a` links it: `crt0.o` is the start-up code, and libgcc provides 32-bit multiply and divide for the 68000.
3. `elf2mod -n hello -s 0x2000 hello.elf CDI_HELLO.APP` makes the OS-9 module. `-s` is the stack size (8 KB here).
4. `mkcdi.py hello HELLO CDI_HELLO.APP [files...]` builds the disc image. The first file is the program the player runs; the rest are the files it loads. Files whose names end in `.RTF` go on as real-time files.

**A new project**: copy `examples/rockrun` (it has the asset rules too), rename things in its Makefile, and point `KIT` at the toolkit's folder.

**There's no C library.** Programs are freestanding: no `printf`, `malloc` or `string.h`. The library has what a game usually needs (`mem_alloc`, `file_load`, `fmt_num`, text drawing). For more, toolchaincdi also builds newlib, though its OS-9 system calls are left for you to fill in.

## 4. The library

Include `cdi.h`; link `crt0.o` first and `libcdi.a`. The types are `u8 u16 u32 s8 s16 s32`.

### The frame loop

```c
gfx_init();                  /* the display, and the controller */
for (;;) {
    u16 pad = pad_read();    /* once a frame */
    /* ...move things... */
    gfx_restore();           /* what was drawn into this buffer last time: gone */
    /* ...draw... */
    stream_feed();           /* if an MPEG stream plays */
    gfx_swap();              /* show it at the next field; then the other buffer */
}
```

### Display

| Function | |
|---|---|
| `int gfx_init(void)` | Sets up the display (and the controller). 0 on success. |
| `u8 *gfx_back(void)` | The buffer being drawn into: `GFX_W` (384) × `GFX_H` (280) bytes, one colour number per pixel. Draw into it directly if you like. |
| `void gfx_swap(void)` | Shows what you drew, at the start of the next field (50 a second on PAL), then waits until the other buffer is free to draw into. |
| `void gfx_palette(const u32 *rgb, int first, int n)` | Sets colours `first..first+n-1` from `0x00RRGGBB` values (the chip keeps 6 bits per component). Colour 0 is always see-through. Takes effect at the next field. |
| `void gfx_background(const struct gfx_image *im, u8 colour)` | What `gfx_restore` restores: a colour, with a picture over it (or 0 for none). Use colour 0 with no picture to have the MPEG video behind everything. |
| `void gfx_restore(void)` | Puts the background back wherever this buffer was drawn into last time. Call it at the start of each frame. |
| `void gfx_clear(u8 colour)` | Fills the whole back buffer. |
| `void gfx_clear_both(u8 colour)` | Fills both buffers (it swaps once). |
| `void gfx_rect(x, y, w, h, colour)`, `gfx_pixel(x, y, colour)` | Draw a filled rectangle or a single pixel. |
| `void gfx_window(x, y, w, h)` | Cuts all drawing (except labels) to this box. `gfx_window(0, 0, GFX_W, GFX_H)` resets it. |

**How the redrawing works.** The library remembers every box it draws into each buffer. `gfx_restore()` repaints only those boxes from the background, so a frame costs only what moves. You never have to clear the screen.

### Pictures and sprites

Pictures come from banks made by `mkgfx.py` (section 5), which hold up to 255 colours shared by all their pictures.

```c
struct gfx_bank *bank = gfx_load_bank("GFX.BIN");
gfx_use_palette(bank);                                 /* its colours into the palette */
const struct gfx_image *ship = gfx_bank_image(bank, IMG_SHIP);   /* IMG_ names: mkgfx --header */
gfx_draw(ship, x, y);                                  /* colour 0 = see-through */
gfx_draw_opaque(title, 0, 0);                          /* every pixel */
```

`struct gfx_image` has `w, h` (the cropped size), `full_w, full_h` (the size of the original picture) and `x, y` (where the crop sits in it). `mkgfx.py` crops away see-through borders, and `gfx_draw` puts the image exactly where the uncropped picture would go. Banks are loaded whole, so keep them under a few hundred KB.

### Text

The built-in 5 × 7 font covers ASCII 32 to 126; at scale 1 a character takes 6 × 8 pixels.

| Function | |
|---|---|
| `gfx_text(x, y, "text", colour, scale)` | Draws text, which is restored away next frame like any drawing. |
| `gfx_label(slot, x, y, "text", colour, scale)` | **Text that stays**, such as a score or a title. It's drawn only when it changes (or when something drawn over it was restored away), so on most frames it costs nothing. There are 16 slots, 47 characters each. Call it every frame with the current text. |
| `gfx_label_clear(slot)` | Removes a label. |
| `gfx_text_width("text", scale)` | Pixels wide. |
| `fmt_num(buf, n, digits)` | A number as text, with leading zeros up to `digits` (0 for none). |

### The controller

| Function | |
|---|---|
| `u16 pad_read(void)` | Once a frame: the buttons and directions held, as `PAD_UP PAD_DOWN PAD_LEFT PAD_RIGHT PAD_B1 PAD_B2`, and `PAD_B12` for both buttons together (or button 3). |
| `u16 pad_pressed(void)` | The bits that came on at the last `pad_read`: for "press button 1 to fire". |

The CD-i controller is a pointing device. The library puts the pointer back in the middle after every read, so movement since the last read gives the directions. A direction is held for a few frames after the last movement, so a steady push doesn't flicker. (`gfx_init` opens it; `pad_init` does it alone.)

### Sound effects

```c
sfx_load("SFX.BIN");      /* a mksfx.py bank: all its sounds into the audio processor */
sfx_play(SFX_LASER);      /* SFX_ names: mksfx --header */
sfx_stop();
```

One sound plays at a time: a new one cuts off the last.

### Streams: video and music from the disc

A real-time file (made by `mkstream.py`) plays straight off the disc while your program runs:

| Function | |
|---|---|
| `int stream_play("NAME.RTF", flags)` | Starts it from the beginning. `STREAM_MPEG` sends MPEG video and audio to the cartridge (ignored if there's no cartridge). `STREAM_ADPCM` sends ADPCM sound to the audio processor. A file can have both. Returns 0 on success. |
| `void stream_feed(void)` | Hands newly arrived MPEG sectors to the cartridge. Call it every frame while an MPEG stream plays. `gfx_swap` also feeds while it waits for the next field. The library keeps 32 sectors (0.43 s), so a frame can take up to about 0.4 s without losing any. |
| `int stream_ended(void)` | 1 when the file has finished (the library works this out from the file's length, and for MPEG from the sectors stopping); call `stream_play` again to loop it. |
| `void stream_stop(void)` | Stops it. |
| `void stream_window(x, y, w, h)` | Where the MPEG picture goes on the screen. The default is 352 × 280 at (16, 0), the full screen; call it before `stream_play`. |
| `u32 stream_timecode(void)` | The MPEG picture's time code (hours, minutes, seconds, pictures, from the stream), to sync things to the video. |
| `int cdi_has_mpeg(void)` | 1 if the cartridge is fitted. |

The MPEG picture shows wherever plane A is colour 0. So for a game over video, set `gfx_background(0, 0)` and draw sprites as usual. For a full-screen video, clear both buffers to 0 (`gfx_clear_both(0)`) and play it.

**One sound chip.** ADPCM music and sound effects share the audio processor: starting one stops the other. For music *with* effects, use MPEG music on the cartridge (`mkstream.py --mpeg --audio music.wav`), which plays alongside the effects. Without the cartridge, a common choice is music on menus and effects in the game. Rock Run does exactly that.

### System

| Function | |
|---|---|
| `void *mem_alloc(u32 size)`, `mem_free(p, size)` | Memory from OS-9 (0 if none is left). Free it with the same size. |
| `void *file_load("NAME", &size)`, `file_free(p, size)` | A whole file from the disc. |
| `u32 cdi_ticks(void)`, `cdi_tick_rate(void)` | The clock, usually 100 ticks a second. |
| `void cdi_sleep(u32 ticks)` | Sleeps. |
| `void cdi_exit(int status)` | Ends the program; the player returns to its menu. |
| `void cdi_catch_crashes(void)` | For debugging. If the program crashes (bus error, address error, illegal instruction, division by zero and the like), it shows "THE PROGRAM CRASHED" with the type and address on screen, instead of silently stopping. Subtract the module's load address from the PC to find the place in your `.elf` (`m68k-elfos9-objdump -d`). It uses colours 250 to 252. |

**Memory:** the player has about 1 MB of RAM in total. The display takes about 220 KB, and OS-9 and the player's own software take some. Plan on roughly 500 KB for your code and data. Keep large things (banks, level data) in `mem_alloc`/`file_load` memory. `static` variables live in the data area, which reaches at most 64 KB.

## 5. Assets: the tools

All tools are Python 3 (`python3 tools/NAME.py`, or `--help`). Inputs are ordinary files.

### Pictures: `mkgfx.py`

```sh
mkgfx.py GFX.BIN --header gfx.h title.png level1.png sprites.png@32x32
```

- Each picture becomes an image. With `@WxH`, a sprite sheet is cut into frames (`IMG_SPRITES_0`, `_1` ...).
- All images share one palette of up to 255 colours (made from all of them together). Transparent pixels (alpha below 128) become colour 0.
- `--colours N` uses fewer colours, leaving the rest for your own (Rock Run keeps colours above 250 for its text colours).
- A full-screen picture is 384 × 280. Pictures for `gfx_background` must be opaque.
- `--header` writes the `IMG_` names and `GFX_COLOURS` (colours used: 1 to N).

### Sound effects: `mksfx.py`

```sh
mksfx.py SFX.BIN --header sfx.h laser.wav boom.wav pickup.wav
```

Any format ffmpeg reads works. Each sound is converted to ADPCM level C mono (18.9 kHz), normalised to the same loudness (`--peak`), and takes about 11 KB of memory per second.

### Video and music: `mkstream.py`

| You want | Command | Plays with |
|---|---|---|
| Video with music (cartridge) | `mkstream.py V.RTF --mpeg --video clip.mp4 [--audio music.wav]` | `STREAM_MPEG` |
| Music alongside sound effects (cartridge) | `mkstream.py M.RTF --mpeg --audio music.wav` | `STREAM_MPEG` |
| Music on any player | `mkstream.py M.RTF --adpcm --audio music.wav [--stereo] [--level B]` | `STREAM_ADPCM` |
| Video with ADPCM sound | `mkstream.py V.RTF --mpeg --adpcm --video clip.mp4` | `STREAM_MPEG \| STREAM_ADPCM` |

- MPEG video is 352 × 288 at 25 fps, 1100 kbit/s by default (`--size`, `--fps`, `--vbitrate`). MPEG audio is 44.1 kHz stereo at 224 kbit/s (`--abitrate`). This is the Video CD format, near the single-speed disc's limit. If the tool warns that sectors are late, lower `--vbitrate`.
- ADPCM sound: level C mono (18.9 kHz) takes 1 sector in 16, C stereo 1 in 8, level B (37.8 kHz) mono 1 in 8, B stereo 1 in 4. Level C stereo sounds good for music.
- `--duration S` cuts the result to S seconds, or loops the sound to fill it. `--volume` sets the gain.
- Disc space: a stream takes 75 sectors (about 172 KB) per second whatever is in it. A 74-minute CD-R holds about 74 minutes of streams in total.

### The disc: `mkcdi.py`

```sh
mkcdi.py mygame MYGAME CDI_GAME.APP GFX.BIN SFX.BIN MUSIC.RTF ...
```

This writes `mygame.bin` + `mygame.cue` (Mode 2, 2352-byte sectors), in the CD-i Bridge format with the first file as the program the player runs. All files go in the root directory. Names are upper case; the program can open them by name (`"GFX.BIN"`).

### Patches: `mkpatch.py`

```sh
mkpatch.py make  old.bin new.bin new.patch.gz
mkpatch.py apply old.bin new.patch.gz new.bin
```

For sharing a port without sharing the game's data (people build the disc from their own copy), or for sending a new build of a big disc in a few MB.

## 6. Making it fast

The 68070 is slow. Measured in MAME (`testing/bench`), which emulates its speed; a real player should be close:

| Operation | Time |
|---|---|
| Waiting for the next field (`gfx_swap`) | up to 20 ms |
| 32 × 32 sprite (`gfx_draw`) | about 2 ms |
| 100 × 40 rectangle | about 3 ms |
| 40 characters of scale-1 text | about 9 ms |
| Full-screen fill | about 47 ms |
| Full-screen picture | about 80 ms |

So:

- **Never redraw the whole screen per frame.** Use `gfx_background` with `gfx_restore`; Rock Run's busiest scenes then run at 14–16 frames a second, and simple ones at 50.
- **Use labels for text that stays**, such as scores and titles: they cost nothing until the text changes.
- **Keep sprites out of the labels' way.** A sprite crossing a label makes the label redraw. `gfx_window` keeps sprites out of a HUD strip, as Rock Run does.
- **Crop sprites.** `mkgfx.py` does this, and the see-through parts cost nothing when drawn.
- **Avoid big multiplications and divisions in inner loops.** The 68000 has only 16-bit multiply; 32-bit `*` and `/` call libgcc. There's no floating point unit: `float` works through libgcc but is very slow. Use fixed-point arithmetic.

## 7. Testing in MAME

```sh
mame cdimono1 -cdrom game.cue                 # with the Digital Video Cartridge (MAME's default)
mame cdimono1 -dvc "" -cdrom game.cue         # without it, like a basic player
```

Running unattended (for automated tests): `testing/autorun.lua` presses "Play CD-i", plays a scripted controller input, takes screenshots and quits:

```sh
SHOTS="30,35" INPUT="31:2:right;32:0.2:b1" TEND=40 \
  mame cdimono1 -cdrom game.cue -video none -nothrottle \
       -snapshot_directory shots -wavwrite sound.wav -autoboot_script testing/autorun.lua
```

`-nothrottle` runs as fast as your computer can, and `-wavwrite` records the sound. The emulated times are the same either way.

## 8. Real hardware

- Burn the `.cue` as a Mode 2 disc (for example ImgBurn on Windows, or `cdrdao write game.cue` on Linux).
- Whether a player boots a CD-R depends on the model; many do.
- **Untested so far.** Everything here works in MAME. On a real player, the display, the MPEG cartridge (MAME emulates the VMPEG model; other models may differ) and the CD streaming may behave differently in details. If something doesn't work, the parts closest to the hardware are in `lib/crt0.s` (the MPEG feed), `lib/video.c` (the display program) and `docs/HARDWARE.md`.
- The library assumes a PAL player (384 × 280 and 50 fields a second, as MAME's cdimono1). On an NTSC player the picture is 384 × 240 and the bottom 40 lines won't show.

## 9. Porting a game

Porting means rewriting the game for the CD-i. The PC program can't run there: the processor (68000 versus x86), the operating system (OS-9 versus DOS) and the hardware are all different. What carries over is the game's data and its rules. This is how MegaRace was done:

1. **Get the assets out.** Convert the game's pictures, video, music and sounds into ordinary files (PNG, MP4, WAV) with decoders for its formats. For Cryo games, `extras/cryo` already has them. For others, look at ScummVM's source, documentation on file formats, or work them out.
2. **Understand the game's rules** from its executable, with a disassembler (Ghidra, IDA, or Capstone with a script). Find the main loop, the player's movement, the enemies, collisions and scoring. Write down what you find, with addresses, as you go.
3. **Write the game in C on this library**, using the rules you found: the same numbers, the same order of events. Keep the original's structures where you can (a `struct` with its variables), since it makes comparing with the original easier.
4. **Fit the CD-i's limits.**
   - The screen is 384 × 280, larger than VGA's 320 × 200.
   - The 68070 is slower than a 386, so redraw only what changes.
   - There is 1 MB of memory, so stream big things from the disc.
   - The controller has two buttons, so map the PC keys to directions, buttons, and both buttons together.
5. **Video:** a game that played video on the PC can play it on the Digital Video Cartridge as MPEG, often looking better than the original. The library's game-over-video model (plane A over the MPEG picture) is how MegaRace's races work.
6. **Music:** with the cartridge, MPEG music plays alongside the sound effects. AdLib or MIDI music can be rendered to WAV on a PC (AdPlug for AdLib, a General MIDI synth for MIDI).
7. **Test against the original constantly:** run both, compare screenshots and behaviour.
8. **Share it legally:** publish your code and tools, and a patch or build script that makes the disc from the player's own copy of the game, not the disc image.

## 10. Limits and known issues

- **One audio processor:** ADPCM streams and sound effects take turns. MPEG music doesn't have this problem.
- **Plane B is unused:** the library uses one plane. Plane B (another 256-colour or RGB555 plane, scrolling via the display program) would allow parallax or a separate HUD layer. See `docs/HARDWARE.md`.
- **PAL only** (section 8).
- **No C library**, by design (section 3). The data area holds up to 64 KB of `static` variables.
- **Not yet tested on real players** (section 8).
- **An intermittent freeze in MAME with the MPEG cartridge.** In Rock Run, about 1 game in 6 freezes after a minute or more when the MPEG video background plays (never seen without the cartridge, or in simpler programs with the stream running). When it happens, the program stops mid-drawing, with no system call in progress and no exception, and OS-9 runs only its idle loop. Signals, sound effects and the controller have been ruled out, and so have crashes in the program. It isn't yet known whether this is MAME's cartridge emulation or something the library does that a real player would also mind. If you see it on a real player, please report it.
