# Building Guide

This guide will help you build the project on your local machine. The process will require you to provide a decompressed ROM of the US version of the game.

These steps cover: decompressing the ROM, running the recompiler and finally building the project.

## 1. Clone the DonkeyKong64Recompiled Repository
This project makes use of submodules so you will need to clone the repository with the `--recurse-submodules` flag.

```bash
git clone --recurse-submodules
# if you forgot to clone with --recurse-submodules
# cd /path/to/cloned/repo && git submodule update --init --recursive
```

## 2. Install Dependencies

### Linux
For Linux the instructions for Ubuntu are provided, but you can find the equivalent packages for your preferred distro.

```bash
# For Ubuntu, simply run:
sudo apt-get install cmake ninja-build libsdl2-dev libgtk-3-dev lld llvm clang
```

### Windows
You will need to install [Visual Studio 2022](https://visualstudio.microsoft.com/downloads/).
In the setup process you'll need to select the following options and tools for installation:
- Desktop development with C++
- C++ Clang Compiler for Windows
- C++ CMake tools for Windows

You'll also need [vcpkg](https://vcpkg.io/en/). Once you have it built and installed, in the root of the DK64 recomp repo you'll need to run

```bash
vcpkg install curl
```

to install the required curl dependency for the mod loader.

The last tool necessary will be `make` which can be installed via [Chocolatey](https://chocolatey.org/):
```bash
choco install make
```

## 3. Decompressing the target ROM
You will need to decompress the NTSC-U 1.0 N64 Donkey Kong 64 ROM (sha1: cf806ff2603640a748fca5026ded28802f1f4a50) before running the recompiler.

The most straightforward way to do this is to set up the [Donkey Kong 64 decompilation](https://gitlab.com/dk64_decomp/dk64), which will decompress the ROM when building. Alternatively, you can run the [Donkey Kong 64 ROM Decompressor Python Script](https://gitlab.com/dk64_decomp/dk64/-/blob/main/tools/generate_decompressed_rom.py) directly, which is what the decompilation uses to decompress the ROM.

Regardless of which method you use, copy the decompressed ROM to the root of the DonkeyKong64Recompiled repository with this filename:
- `donkeykong64.decompressed.us.z64`

## 4. Generating the C code

Now that you have the required files, you must build [N64Recomp](https://github.com/N64Recomp/N64Recomp) and run it to generate the C code to be compiled. The building instructions can be found [here](https://github.com/N64Recomp/N64Recomp?tab=readme-ov-file#building). That will build the executables: `N64Recomp` and `RSPRecomp` which you should copy to the root of the DonkeyKong64Recompiled repository.

After that, go back to the repository root, and run the following commands:
```bash
./N64Recomp us.toml
./RSPRecomp n_aspMain.toml
```

## 5. Building the Project

Finally, you can build the project! :rocket:

On Windows, you can open the repository folder with Visual Studio, and you'll be able to `[build / run / debug]` the project from there.

If you prefer the command line or you're on a Unix platform you can build the project using CMake:

```bash
cmake -S . -B build-cmake -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_C_COMPILER=clang -G Ninja -DCMAKE_BUILD_TYPE=Release # or Debug if you want to debug
cmake --build build-cmake --target DK64Recompiled -j$(nproc) --config Release # or Debug
```

## 6. Success

Voilà! You should now have a `DK64Recompiled` executable in the build directory! If you used Visual Studio this will be `out/build/x64-[Configuration]` and if you used the provided CMake commands then this will be `build-cmake`. You will need to run the executable out of the root folder of this project or copy the assets folder to the build folder to run it.

> [!IMPORTANT]
> In the game itself, you should be using a standard ROM, not the decompressed one.

## The web page

This fork's `main` branch also builds with [Emscripten](https://emscripten.org) into a page, `DK64Recompiled.html`,
that draws through WebGPU. `lib/rt64`, `lib/N64ModernRuntime` and `lib/RecompFrontend` are Donkey Kong 64:
Recompiled's own, at the same commits (its mod store and `--game` included), and
[N64Recomp-browser](https://github.com/n64hub/N64Recomp-browser) (`lib/N64Recomp-browser`) patches their checkouts
with the web support and makes the game a page. The patches change nothing in the desktop build. Mods and the mod store
are off on the page: the store fetches with curl, which a browser hasn't got, so `web/curl` stands in for it there
and nothing reaches it. RT64's render to RAM stays on, as the game reads its own
frames: a Banana Fairy photo is a 160x128 patch of the frame on screen that the CPU copies out of RDRAM
(`func_global_asm_806FFF88`, patched in `patches/patches_heap.c`), and without render to RAM nothing draws the frame
there, so the photo would show whatever the memory last held. On the page RT64 copies each frame into RDRAM a list
late (`recompui::renderer::set_render_to_ram_late`): the copy lands once the GPU has it, while the next list is drawn
at the latest, so a list doesn't wait for the GPU process to hand the frame back. In headless Chrome 155 on an RTX 2070, through
the menus and walking about DK Isles, a list takes 3.7 ms (median, two runs) against 7.8 ms with the copy made before
the list ends. The photo then holds the frame a list or two earlier, which shows only while the camera turns. Before
the game starts, every VI draws a dummy frame into 0x80700000 or 0x80725800, where the game's main code and audio
microcode load; a late copy of one landed on them and crashed 2 of about 26 boots in the audio microcode (`Unhandled
jump target in microcode n_aspMain`). RecompFrontend-web 75fe596 keeps the dummy frame out of RDRAM on a page, and
since then none of 29 cold boots in headless Chrome 155 and 25 in Firefox 158 crashed. Most of them report one CPU
write to a framebuffer with a copy pending (`RT64: the CPU wrote to the framebuffer at 0x00034000 ...`): the boot draws
there, and the overlay loaded at 0x80024000 then zeroes its BSS over it. RT64 drops that copy, so the BSS keeps its
zeroes. What it can't see is a zero written over a zero, which would get the frame, but that framebuffer's black is
0x00010001 throughout the boot and none of its words is 0. `patches/shader_survey.c` gives
the camera and film on request (its `film`), which is how that was checked by script: warp to a map, Z then C-down,
then B. The game's other uses of a frame, the snapshots in
transitions (`func_global_asm_8070A848`, patched in `patches/patches_framebuffer.c`), copy it on the GPU, which RT64
does either way.

The game's sound once went silent for good on the page, a second or three after the boot began, in 5 of 7 boots in
headless Chrome. Chrome holds the graphics thread for the boot's prebuilt shaders, about 5 s. Meanwhile the scheduler
sees the RSP busy with the graphics task it started, so the audio task waits, and a retrace's message to the audio
thread lands while the thread waits for that task. The thread takes that message at the top of its next round, before
its new task is queued, and so asks the scheduler for nothing. It then waits for a task nobody starts, and with the
message never taken no other is sent. Now the retrace handler (`patches/timing_fixes.c`) starts a pending audio task
itself, as the thread's own request would, and all 9 boots with it played on. Firefox runs the game through the wait,
and 3 boots without the fix kept their sound.

After Start the page presents nothing for ~1.5 s twice in the boot. Both are the game's: its main thread
(`func_global_asm_805FBFF4`) waits on one message for 1.30 s and then 1.42 s, with every other game thread, the
display list thread and the page's thread idle and no RSP task running, so there is no frame to present (a CPU
profile of every thread, headless Chrome 155). What the web support adds to them, the threads the boot starts
waiting for a worker, is in N64Recomp-browser's docs/findings.md (Present gaps after Start).

With steps 1 to 3 done, patch the submodules:

```bash
cmake -P lib/N64Recomp-browser/patches/apply.cmake lib
```

`lib/RecompFrontend` takes its patch with a three-way merge, which fetches the upstream commit the patch was made on
from N64Recomp-browser's fork of it. Then step 4, with the N64Recomp built from the patched
`lib/N64ModernRuntime/N64Recomp` (its `funcs.h` declares every function the generated code calls, which a WebAssembly
build needs); the generated code is the same for both builds. Then, with the Emscripten SDK installed and activated and [naga](https://github.com/gfx-rs/wgpu/tree/trunk/naga)
installed (`cargo install naga-cli`), and `spirv-opt` on `PATH` (SPIRV-Tools, `apt install spirv-tools`; without it
the prebuilt shaders below come out several times the size):

```bash
embuilder build sdl2-mt freetype-legacysjlj      # once
emcmake cmake -S . -B build-web -G Ninja -DCMAKE_BUILD_TYPE=Release -DRT64_NAGA_EXECUTABLE=<naga> \
    -DPATCHES_C_COMPILER=<clang with a MIPS target> -DPATCHES_LD=<ld.lld>
cmake --build build-web --target DK64Recompiled
```

A first build compiles about 670 files, which takes half an hour or more on Windows. Two things make later builds
shorter:

- [ccache](https://ccache.dev) on `PATH` when you configure. Every compile then goes through it, so a new build dir,
  or another checkout of this repo with its build dir in the same place, compiles nothing that was compiled before:
  a second, fresh build dir took 8 minutes instead of 35, most of it the link. Keep the build dir
  inside the checkout (`build-web`): from one outside it, about half the files miss. The cache is per user, so it also
  serves other recomps built on the same web support.
- `-DN64RECOMP_BROWSER_FAST_LINK=ON` when you only want to try something. The link then takes seconds instead of
  minutes, because it skips Binaryen's optimization of the whole program. Leave it off for a page that people play.

vcpkg isn't needed for the page: it brings curl, for the mod store, which `web/curl` stands in for. `build-web` then holds `DK64Recompiled.html`,
`.js`, `.wasm`, `.data` and `coi_serviceworker.js`. The page needs a browser with WebGPU (Chrome or Edge 113 and up)
and must be cross-origin isolated: serve the folder with `Cross-Origin-Opener-Policy: same-origin` and
`Cross-Origin-Embedder-Policy: require-corp`, or on a host that can't set headers, as is, and `coi_serviceworker.js`
sets them after one reload. The player picks their ROM (the plain one, not the decompressed one) in the launcher; it,
the settings and the saves are kept in the browser's IndexedDB for that origin.

The page builds the shaders the game draws with from `src/rt64_shaders.txt`, specialized and run through `spirv-opt`
when the page is built, so no scene draws with the slow ubershader. The list is in 15 sections, 519 descriptions (475 of
them 361 pipelines): section 0, the logos (10 pipelines), then the DK Rap (49), which every visit reaches (Start in the logos
goes to it) and whose first frame wants 31 of its shaders at once, then the menus (55: a moment of DK TV's Jungle Japes
demo, the title, the main menu and its screens), then the new game's opening, Training Grounds and DK's house, DK
Isles, and a world each with its lobby (and, in eight, places further in: bosses, rooms, arcade games). In Chrome the whole list is built in the background,
sections 0 and 1 first on half the host's cores and then the rest one at a time, a section the game has reached moved
to the front, and the game waits only when a frame wants 4 or more of a section's shaders that aren't built yet, under
a small "Preparing graphics" indicator in a corner. Firefox builds a pipeline in line with the rest of its GPU work, so
there section 0 is built before the game passes its first frame and each other section when the game first draws with
its shaders, the game waiting, and it keeps none between visits, so every visit waits as long: the logos 0.5-0.7 s,
the DK Rap 6.3-6.5 s as it begins and the menus 4.3 s (Firefox 158), Frantic Factory 3.6 s (Firefox 157). The logos
are a section of their own so that the page draws its first frame at once: as one section with the Rap and the menus,
115 pipelines, Firefox waited 13-30 s before it. A shader the list
misses is built behind the game, and the page remembers it and builds it with its section on the next visit. To
record the list again, build the page and run `bash tools/record_shaders.sh build-web <the plain ROM>` (its header
says how): it takes about half an hour, and then the page is built again with the new list. `tools/play_plan.json`
plays the page as a player would, for N64Recomp-browser's `tools/play_page.mjs`, in 2-3 minutes: the logos and the
DK Rap, then the main menu and Jungle Japes, reached by `tools/shader_survey.js`, and about 45 s of DK on foot there.
`tools/play_plan_extended.json`, for when a longer play is needed, goes on to Angry Aztec, Gloomy Galleon and Creepy
Castle in the same game, about 35 s each, in at most 5 minutes (their headers say more).

Where the GPU reports finished work late, as Firefox does on a laptop's Intel UHD, even copies made before a list ends
fall seconds behind the VIs that queue them. Without RecompFrontend-web 75fe596 the runtime's dummy frames, drawn into
0x80700000 every VI before the game starts, were copied to RDRAM that way and kept landing after Start, on the game's
code and audio microcode: there the page never booted, logging `Failed to execute task type: 2` or
`RuntimeError: index out of bounds` in the wasm 22-27 s in, with nothing drawn and the sound dry. A preload that holds
`onSubmittedWorkDone` back 100 ms reproduces it in Firefox on any GPU, at the same wasm offset in three boots of three,
and with 75fe596 the page boots through it and plays into Frantic Factory.

How the page plays, in each browser and build, is recorded in N64Recomp-browser's
[`plays.jsonl`](https://github.com/n64hub/N64Recomp-browser/blob/main/plays.jsonl) and its
[games table](https://github.com/n64hub/N64Recomp-browser#games).
