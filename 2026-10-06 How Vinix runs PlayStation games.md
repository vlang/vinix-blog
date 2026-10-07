Vinix's ARM64 desktop can now run original PlayStation (PS1) games in a native window. We
tested Crash Bandicoot's USA disc image through its title screen and island map
into N. Sanity Beach, then moved, jumped and spun through the starting crates.

*Updated 7 October 2026: PlayStation 2 and Nintendo 64 now have native desktop
apps too. Both run bundled Paddle homebrew; their implementation, tests and
screenshots are included below.*

![Crash Bandicoot running in Vinix's PlayStation window, with Crash jumping in N. Sanity Beach](images/ps1-crash-bandicoot-vinix-desktop.png)

The game runs through [PCSX-ReARMed](https://github.com/libretro/pcsx_rearmed),
an existing PlayStation emulator. We built its unchanged libretro core for
ARM64 and wrote a V frontend that connects it to Vinix's compositor, controller
input, audio device and files. The original game code and assets remain intact.

## Two instruction sets, one desktop

The PlayStation's CPU executes MIPS instructions. Our current Vinix desktop
build runs on ARM64, so the game needs an emulator between those two machines.
PCSX-ReARMed supplies the virtual console: its CPU, graphics, sound, CD drive
and controller interfaces.

The emulator itself is a native ARM64 program. Vinix's Linux-compatible system
calls let it open game files, allocate memory and map a shared framebuffer.
The game sees the emulated PlayStation hardware; the emulator sees Vinix.

The path from disc to desktop is:

```text
Original PS1 executable or disc image
  → PCSX-ReARMed executes MIPS code and emulates the console
  → V frontend receives game pixels and supplies controller state
  → shared framebuffer reaches the Vinix desktop compositor
  → game appears in the PlayStation window
```

PCSX-ReARMed offers several execution and rendering options. This first build
uses its **MIPS interpreter** and **NEON software GPU**. ARM's NEON instructions
help the emulator draw the console's graphics on the CPU. The host GPU does
not render the PlayStation scene.

We disable dynamic recompilation and asynchronous core workers in this build.
That keeps emulation on the frontend's calling thread and avoids executable
code generation while establishing the port. It also leaves performance work
for later: this result does not establish full-speed emulation across games.

## A small frontend around an existing emulator

Libretro defines the boundary between an emulator core and the program hosting
it. The frontend registers callbacks for video, audio, input and configuration,
loads the game with `retro_load_game()`, then calls `retro_run()` to advance
emulation.

Vinix uses that interface directly. The frontend and core are statically
linked with musl into one ARM64 executable, `vinix-ps1`. Our build pins the core
to revision `c8816799b50388e61cfe237fe2cdbb7d8175f20a` and verifies the downloaded
source archive with SHA-256. The integration uses the core's existing API and
build options without patching its source.

The V frontend speaks the same pipe protocol as standalone Vinix desktop
applications. It describes the game view, status text, **Open game**, **Pause**,
**Reset** and on-screen controller buttons. The compositor draws those controls
and sends clicks and keys back to the app. Each desktop poll advances one
emulated frame while the game is running. Pause stops those calls; Reset uses
the core's reset function.

This gives the emulator a Vinix window without an X11 server, Wayland server
or RetroArch installation. All the new PlayStation integration lives in
userspace.

## Getting the pixels onto the screen

The video callback receives the emulator's actual framebuffer, including its
dimensions, row pitch and pixel format. The frontend converts those pixels
into a 640×480 surface shared with the compositor through `mmap`.

The surface uses Vinix's existing VSF1 format and two pixel buffers. The
frontend writes the next frame into the available buffer, then publishes its
index with an atomic operation. The reader marker tells it which buffer the
compositor is using, so it avoids overwriting that buffer during presentation.

The window presents the surface at a 4:3 aspect ratio, adding dark borders
when needed. Resizing changes its presentation size. The polygons, textures
and animation inside the view all come from PCSX-ReARMed running the game.

## Buttons, sound and memory cards

Controller input takes the reverse route. A desktop event updates the
frontend's button state, and the core reads that state through libretro's
input callback. The current frontend exposes one standard digital pad.

Arrow keys or WASD control the D-pad; Z, X, C and V map to Cross, Circle,
Square and Triangle. Enter presses Start. Mouse presses can hold the on-screen
buttons. Keyboard events currently produce short button pulses through the
desktop's text-key protocol, so continuous keyboard input still has room for
improvement.

Audio samples go to Vinix's OSS-compatible `/dev/dsp` device as 44.1 kHz stereo
PCM when the device is available. Writes are nonblocking so audio cannot hold
up the desktop exchange. The regression runs use `--mute`; sound output has
not been verified by these tests.

The emulated primary memory card is a 128 KiB file under the active user's
`~/.local/share/vinix/ps1/saves`. Each absolute game path gets its own card.
The frontend restores it when loading the game and saves it every 300 emulated
frames, before reset or a game change, and during normal window close. It
writes a temporary file and renames it into place.

Both verified games used the core's high-level emulation BIOS. This implements
BIOS services inside the emulator and lets those runs boot without a Sony BIOS
file. HLE has compatibility limits; games that need firmware can use a supplied
BIOS in `~/.local/share/vinix/ps1/bios`. The
[upstream BIOS documentation](https://docs.libretro.com/library/pcsx_rearmed/#bios)
lists the supported filenames and explains the tradeoff.

## Testing with homebrew and Crash Bandicoot

The default build includes
[Tetrade 1.0](https://github.com/Logan-Campbell/Tetrade/releases/tag/v1.0),
Logan Campbell's MIT-licensed PlayStation Tetris game. We download the author's
original executable and BIN/CUE release assets, verify their hashes and keep
the license alongside them. This gives us a reproducible game for testing the
same emulation path used by a commercial disc.

![Tetrade running in the native Vinix PlayStation app](images/ps1-tetrade-vinix.png)

The ARM64 QEMU regression boots Tetrade, checks that its pixels animate, and
compares neutral and controller-input frames at the same emulated age. That
last detail matters: a difference must come from the input, rather than simply
from sampling the animation later. The recorded run found 1,920 changed pixels
in an animation sample and 1,408 changed pixels after Right at an equal age.

It also verifies pause and resume, deterministic reset, memory-card persistence
across fresh emulator processes, and successful shutdown with the shared
surface removed. Both Tetrade's PS-X executable and its CUE/BIN disc boot
passed. A separate desktop test uses real QEMU mouse events to start Marathon,
pause its falling pieces, resume and close the window.

Crash Bandicoot was a separate trial using a supplied USA CUE/BIN image. It
reached N. Sanity Beach and responded to movement, jumping and spinning. Pause
kept every game pixel unchanged for 60 poll frames; after resuming and running
120 frames, 17,654 pixels changed. Another run opened the disc through the
desktop's **Open game** path prompt and exercised mouse and keyboard controls.

These runs verify the path from PS1 instructions to interactive gameplay in
Vinix. They establish compatibility for Tetrade and the tested part of Crash.
The frontend currently has no analog-controller, multiplayer, save-state or
disc-swapping controls.
Game speed follows the desktop polling cadence, and real-time performance
across other games remains unmeasured.

## PS2 and Nintendo 64 join the desktop

The same approach now hosts two more consoles. Each emulator is compiled into
a native ARM64 executable with a V frontend, presenting its actual game pixels
through the shared surface protocol. Each gets its own window, controller
state and save files, so the two games can run alongside each other.

![PS2 and Nintendo 64 Paddle games running simultaneously in two Vinix desktop windows](images/ps2-n64-vinix.png)

### PlayStation 2

The PS2 app uses [Iris](https://github.com/allkern/iris), with its Emotion Engine
and IOP interpreters and software GS renderer. The included **Paddle** game is
MIT-licensed homebrew built from source. It executes real R5900 and IOP code,
draws through the PS2's GIF/GS hardware, and reads the emulated DualShock through
SIO2. Its best score is stored on an emulated memory card and restored when the
game is opened again.

This homebrew initializes the machine itself and boots without Sony firmware.
Other ELF programs and ISO disc images follow the emulator's BIOS boot path;
provide a dump from your console in `~/.local/share/vinix/ps2/bios/ps2.bin` or
select it with `--bios=PATH` when launching through the desktop. Each game path
gets its own card under `~/.local/share/vinix/ps2/saves`.

The controls now use the PlayStation's colored Cross, Circle, Square and
Triangle symbols in their controller arrangement, alongside a D-pad, shoulder
buttons, Select and Start. Hovering shows keyboard shortcuts, and the buttons
highlight while held. Enter starts Paddle; Left/Right or A/D move the paddle.
Pause stops emulation, and the circular-arrow Reset button boots the game again.

![The updated PS2 window with colored PlayStation controller symbols and Paddle running](images/ps2-controller-ui-vinix.png)

### Nintendo 64

The N64 app uses [paraLLEl-N64](https://github.com/libretro/parallel-n64), with
the R4300 CPU interpreter, CXD4 RSP interpreter and Angrylion software renderer.
Its own MIT-licensed **Paddle** cartridge executes N64 MIPS instructions,
clears the background through RDP commands, draws the game into the framebuffer,
and reads controller input through SI/PIF. The best score persists in cartridge
SRAM.

The frontend accepts uncompressed `.z64`, `.v64` and `.n64` cartridges. Normal
cartridge games need no external BIOS. Save files live under
`~/.local/share/vinix/n64/saves`; the core's save memory also includes EEPROM,
FlashRAM and controller paks. Saves are flushed periodically, before reset or
a game change, and on clean close, as they are in the PS2 app.

Arrows control the D-pad, WASD supplies full-deflection stick input, and Enter
presses Start. The window also exposes A/B, Z, L/R and the four C buttons, with
pointer holds for the stick directions. Both apps offer **Open game**, **Pause**
and **Reset**. Stereo output is connected to `/dev/dsp`: PS2 uses 48 kHz, while
N64 uses the rate reported by the core. Audible output has not been verified by
these regressions.

### What has been tested

Both ports passed their ARM64 Vinix guest and desktop regressions in QEMU on
7 October 2026. The guest checks animated pixels, controller effects at equal
emulated ages, pause/resume, reset, saves restored by a fresh process, and
recovery from rejected game files. The desktop tests send actual QEMU pointer
events to start Paddle, move its paddle, pause/resume and close the window.
N64 also passed tests for all three cartridge byte orders and instruction
limits that prevent stalled CPU or RSP code from hanging the app.

These are early interpreter and software-rendering ports. Expect limited
speed and incomplete compatibility. Only the bundled homebrew has been
validated for PS2 and N64; commercial games have not. Physical gamepads and
accelerated rendering remain future work, as do PS2 analog sticks and
proportional N64 stick input.

## Building and playing

From a Vinix checkout containing the PlayStation frontend, after building the
ARM64 userland sysroot:

```sh
./scripts/build-ps1-aarch64.sh
./scripts/build-ps2-aarch64.sh
./scripts/build-n64-aarch64.sh
./scripts/build-desktop-aarch64.sh
./scripts/run-desktop-aarch64.sh --no-build
```

Open **PlayStation** from the desktop. Tetrade starts by default; press Start,
then Cross to select Marathon. To load another game, choose **Open game**, type
or paste its file path, and press Enter. Keep CUE sheets beside their referenced
BIN tracks. The core also accepts formats including CHD, PBP, ISO, M3U and
PS-X EXE, though those formats have not all been exercised by our regressions.

Open **PlayStation 2** or **Nintendo 64** to start their bundled Paddle games.
Press Start or Enter, then move left and right to keep the ball in play and
clear the bricks. Their **Open game** prompts accept a PS2 ISO/ELF or an N64
cartridge path. Omit any emulator build step above if you do not want that app
included in the desktop image.

The builds include the emulator licenses and dependency notices. Tetrade and
the two Paddle games are bundled; commercial game images and Sony BIOS files
are supplied separately.

The implementation is in `games/ps1`, with build inputs in `build-support/ps1`
and regressions in `tests/ps1` in the
[Vinix repository](https://github.com/vlang/vinix). PCSX-ReARMed does the
console emulation; Vinix supplies the native process, files, input and desktop
window that let it become a usable application.

The PS2 and N64 ports follow the same layout in `games/ps2` and `games/n64`,
with their own build inputs and regression suites. The
[PS2 guide](https://github.com/vlang/vinix/blob/master/docs/ps2.md) and
[N64 guide](https://github.com/vlang/vinix/blob/master/docs/n64.md) cover setup,
controls, save handling and reproducible tests.
