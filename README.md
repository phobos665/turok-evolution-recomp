# turok-evolution-recomp

Turok: Evolution for the original Xbox, recompiled into a native program for
Windows and Linux (including SteamOS). You build it from your own copy of the
game: the build translates the game's code to C on your machine and compiles
it against the [DirectXbox-Recomp](https://github.com/phobos665/DirectXbox-Recomp)
toolkit (`xboxrecomp/` here). This repository contains no game code or data.

## What works

- The intro movies, the front end, New Game and the first jungle level have
  been played: walking, the war club and the bow, enemies, taking damage,
  dying and restarting, and the in-engine cutscenes. Later levels have had
  little testing.
- 60 fps, held in the level, with short dips while a level loads.
- Higher frame rates. The game keeps time by the clock rather than counting
  frames, so it plays at the right speed at 120, 144 or uncapped.
- Widescreen. The camera is widened rather than stretched, and the HUD stays
  in proportion. The front end and the movies are shown at 4:3 between bars,
  as they were made.
- Rendering at up to 8x the console's 640x480.
- Fullscreen and variable refresh rate (G-SYNC, FreeSync).
- Controllers, keyboard and mouse look, all rebindable.

Tested on Linux (SteamOS, AMD). The port was first brought up on Windows;
macOS has not been tried.

## What you need

Turok: Evolution, the European (PAL) release, extracted from your disc. The
build reads `default.xbe` and warns if it is a different release (see
[Other releases](#other-releases)).

Windows 10 or 11, 64-bit:

- Visual Studio 2022 or 2019 with the Desktop development with C++ workload
- [CMake](https://cmake.org/download/) 3.20 or newer, on PATH
- Python 3.10 or newer with `capstone`: `py -3 -m pip install capstone`
- Git

Linux, x86-64:

- GCC or Clang, CMake 3.20 or newer, Ninja, pkg-config, Git, curl
- Development headers for X11 and Wayland, PulseAudio or PipeWire, OpenSSL,
  FreeType and fontconfig
- Vulkan drivers (Mesa's RADV, ANV, or NVIDIA's own)
- Python 3.10 or newer with `capstone`, in a virtual environment:
  `python3 -m venv ~/.venvs/recomp && ~/.venvs/recomp/bin/pip install capstone`

On Debian or Ubuntu that is

```sh
sudo apt install build-essential cmake ninja-build pkg-config git curl python3-venv \
    libx11-dev libxext-dev libxrandr-dev libxcursor-dev libxi-dev libxss-dev \
    libxfixes-dev libxtst-dev libxkbcommon-dev libwayland-dev wayland-protocols \
    libegl-dev libdrm-dev libgbm-dev libpulse-dev libasound2-dev libudev-dev \
    libssl-dev libfreetype-dev libfontconfig-dev
```

SteamOS has no compiler, and its system is read-only, so build inside the
Freedesktop SDK, which runs as a Flatpak and has all of the above. It is
often installed already, because Flatpak apps such as VS Code use it; if
`flatpak list` does not show `org.freedesktop.Sdk` 25.08, install it with
`flatpak install flathub org.freedesktop.Sdk//25.08`. Open a shell in it from
Konsole:

```sh
flatpak run --filesystem=home --share=network --device=dri \
    --socket=wayland --socket=fallback-x11 --socket=pulseaudio \
    --command=bash org.freedesktop.Sdk//25.08
```

and do everything below in that shell, venv included. The game it builds also
runs outside it, straight from SteamOS.

## Build

```sh
git clone --recursive https://github.com/phobos665/turok-evolution-recomp.git
cd turok-evolution-recomp
```

Copy the extracted disc into a folder called `game`, so that
`game/default.xbe` exists with `Data/`, `Game.txt` and the `supertree*.tre`
archives beside it.

On Linux, fetch the shader compiler first (once; distributions do not package
it):

```sh
bash xboxrecomp/tools/linux/fetch_dxc.sh
```

Then:

```sh
py -3 scripts/build.py                        # Windows
~/.venvs/recomp/bin/python scripts/build.py   # Linux
```

The first run translates the game, which takes a few minutes, then compiles
it, which takes longer. Later runs only recompile. The program ends up in
`build/Release/`:

| | Windows | Linux |
| --- | --- | --- |
| The game | `turok_recomp.exe` | `turok_recomp` |
| Settings window, then the game | `turok_recomp_launcher.exe` | `turok_recomp_launcher` |

If your extracted games live somewhere else, `game` can be a link to the folder
(`ln -s "/path/to/Turok Evolution" game`), or pass `--game DIR` to the build and
set `RECOMP_GAME_DIR` when you play.

If you cloned without `--recursive`, run `git submodule update --init --recursive`.

## Play

Run the launcher to choose settings and play, or the game itself to start
straight away. You can make a shortcut to either from anywhere. On SteamOS,
add the launcher to Steam as a non-Steam game (Steam's "Add a Non-Steam Game"
in desktop mode) to play it from Game Mode.

| Key | |
| --- | --- |
| F9 | Show or hide the frame rate |
| F10 | Change the frame cap: adaptive, 60, 30, 120, 144, off |
| F11 | Save a screenshot and a capture of the frame beside the program, for bug reports |
| Alt+Enter | Switch between fullscreen and a window |

Controllers work as an Xbox pad. The keyboard plays as controller 1:

| Xbox | Key |
| --- | --- |
| D-pad | Arrows |
| A, B, X, Y | Z, X, A, S |
| Start, Back | Enter, Backspace |
| White, Black | Q, W |
| Left, right trigger | E, R |

Change them in the launcher's Input tab, or in `input_bindings.json` (format in
`xboxrecomp/docs/technical/input-binding.md`). The Input tab's mouse look
option turns the mouse into the right stick.

| | Windows | Linux |
| --- | --- | --- |
| Saves | `game\UDATA\41430004\` | `game/UDATA/41430004/` |
| Settings | `%APPDATA%\xboxrecomp\titles\41430004.conf` | `~/.config/xboxrecomp/titles/41430004.conf` |
| Controls | `%APPDATA%\xboxrecomp\input_bindings.json` | `~/.config/xboxrecomp/input_bindings.json` |
| Log, when not started from a terminal | `build\Release\turok_recomp.log` | `build/Release/turok_recomp.log` |

## Settings

These are rows in the launcher's Video tab, and lines in the settings file.
Every line can also be set with the environment variable shown, which wins
over the file.

| Setting | File | What it does |
| --- | --- | --- |
| Resolution | `resolution_scale` (`RECOMP_RES_SCALE`) | Renders at 1 to 8 times 640x480 and filters down. 2 is much sharper and costs little |
| Widescreen | `widescreen` (`RECOMP_WIDESCREEN`) | 16:9, with the wider camera |
| Texture sharpness | `anisotropy` (`RECOMP_ANISO`) | Anisotropic filtering, 1 to 16 |
| Frame rate | `frame_cap` (`RECOMP_FPS_CAP`) | Adaptive (the default, 60 as on the console), 60, 30, 90, 120, 144, 165, 240, or 0 for uncapped |
| Show frame rate | `fps_overlay` (`RECOMP_FPS_OVERLAY`) | Start with the counter on |
| Fullscreen | `fullscreen` (`RECOMP_FULLSCREEN`) | Borderless fullscreen |
| Variable refresh rate | `vrr` (`RECOMP_VRR`) | In fullscreen, frames reach the display as they are ready. It has to be on in the display's driver too |

For smoother motion, raise the frame cap to your display's refresh rate rather
than using frame interpolation, which is not tuned for this game. Uncapped,
the game can run at several hundred frames a second, and the loading screens
at thousands; a cap saves power, which matters on a handheld.

## Other releases

The build identifies the release by the SHA-1 of `default.xbe`:

| Release | `default.xbe` SHA-1 |
| --- | --- |
| Europe (PAL) | `e6510e44ca99b67b9015c27f04bc1b49ce8c0f9a` |

`config/` lists addresses inside that `default.xbe`: function entry points the
translation needs, and the XDK functions the toolkit replaces. The fixes in
`src/recomp_manual.c` attach to addresses in it too. Any other disc, such as
the USA release, needs its own addresses. The build prints a warning with the
SHA-1 and carries on, but the result is unlikely to run; please open an issue
with it.

## Updating

```sh
git pull
git submodule update --init --recursive
py -3 scripts/build.py --relift
```

On Linux, use `~/.venvs/recomp/bin/python` in place of `py -3`. `--relift`
translates the game again, which is needed whenever the toolkit, `config/` or
`src/recomp_manual.c` changes. `--relift --from lift` repeats only the last
stage, which is enough after a change to `src/recomp_manual.c`.

## Problems

- "no default.xbe in ...": the disc's files are not in `game/`.
- Linux, "DXC (the shader compiler) is missing": run
  `bash xboxrecomp/tools/linux/fetch_dxc.sh`, then build again.
- A black window and nothing else: read the log (see [Play](#play)).
- A rendering bug in one place: press F11 there, and attach the `.bmp` and
  `.d3dcap` files it writes to an issue.

## Legal

You need your own copy of Turok: Evolution. Do not share `game/`,
`src/recomp/` (it is the game's translated code) or anything in `build/`.
This repository and xboxrecomp contain no game code or assets.

The code in this repository is licensed under the GNU General Public License
v3.0; see [LICENSE](LICENSE). That covers the files committed here, not the game.
