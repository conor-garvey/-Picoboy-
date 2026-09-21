# PicoBoy

PicoBoy is a Raspberry Pi Pico W handheld project built around an `ILI9341` display and an `LSM6DSL` IMU.

Right now the project includes:

* a custom `Display` class for the ILI9341
* strip-buffer rendering to reduce RAM usage without a full framebuffer
* menu and profile selection flow
* `Plumber Man`, a simple platformer
* `Sky Dodger`, a motion-control based game

## Hardware

### Display Wiring

* `LCD CS` -> `GP17`
* `LCD CLK` -> `GP18`
* `LCD SDI` -> `GP19`
* `LCD RS/DC` -> `GP20`
* `LCD RST` -> `GP21`
* `LCD LED` -> `GP22`

### Buttons

* `UP` -> `GP2`
* `DOWN` -> `GP3`
* `LEFT` -> `GP4`
* `RIGHT` -> `GP5`
* `BUTTON A` -> `GP6`
* `BUTTON B` -> `GP7`
* `SELECT` -> `GP8`
* `START` -> `GP9`

### IMU

Current code configuration is in [src/main.cpp](./src/main.cpp).

The IMU uses:

* `I2C` -> `i2c0`
* bus speed -> `400000`
* address -> auto-detects `0x6A` or `0x6B`

Check [src/main.cpp](./src/main.cpp) for the currently configured SDA and SCL pins.

## Project Layout

* [include/picoboy/display.hpp](./include/picoboy/display.hpp) - display, rendering, text, and frame timing
* [include/picoboy/buttons.hpp](./include/picoboy/buttons.hpp) - button input handling
* [include/picoboy/lsm6dsl.hpp](./include/picoboy/lsm6dsl.hpp) - IMU driver
* [include/picoboy/menu_app.hpp](./include/picoboy/menu_app.hpp) - boot flow and menus
* [include/picoboy/plumber_man_game.hpp](./include/picoboy/plumber_man_game.hpp) - platformer game
* [include/picoboy/sky_dodger_game.hpp](./include/picoboy/sky_dodger_game.hpp) - IMU-based dodging game
* [src/main.cpp](./src/main.cpp) - top-level setup and main loop

## Requirements

You need:

* Raspberry Pi Pico SDK
* CMake
* Ninja or another CMake generator
* ARM GCC toolchain for the Pico

The easiest setup on Windows is usually the Raspberry Pi Pico VS Code extension, because it installs and manages the Pico SDK and toolchain for you.

## Fresh Windows Setup

If your laptop is starting from nothing:

1. Install [Visual Studio Code](https://code.visualstudio.com/).
2. Open VS Code and install the `Raspberry Pi Pico` extension published by `Raspberry Pi`.
3. Get this project onto your laptop:

   * install Git and clone it, or
   * download the repository ZIP from GitHub and extract it
4. In VS Code, open the `PicoBoy` project folder itself.
5. Let the Pico extension install the Pico SDK, toolchain, CMake, and Ninja if prompted.
6. Restart VS Code once setup finishes.

After that, open a PowerShell terminal in the project folder and use the build commands below.

## Build Setup

### PowerShell

From the project root:

```powershell
cmake -S . -B build -G Ninja -DPICO_BOARD=pico_w
cmake --build build
```

If the Pico SDK is not already configured, set `PICO_SDK_PATH` first:

```powershell
$env:PICO_SDK_PATH="C:\path\to\pico-sdk"
cmake -S . -B build -G Ninja -DPICO_BOARD=pico_w
cmake --build build
```

### Clean Reconfigure

```powershell
Remove-Item -Recurse -Force build
cmake -S . -B build -G Ninja -DPICO_BOARD=pico_w
cmake --build build
```

### Git Clone Example

```powershell
git clone https://github.com/YOUR-USERNAME/PicoBoy.git
cd PicoBoy
cmake -S . -B build -G Ninja -DPICO_BOARD=pico_w
cmake --build build
```

## Build Outputs

After a successful build, the important files are usually:

* `build/PicoBoy.uf2`
* `build/PicoBoy.elf`
* `build/PicoBoy.bin`

To flash the board:

1. Hold the Pico W `BOOTSEL` button while plugging it in.
2. Copy `build/PicoBoy.uf2` to the Pico mass-storage device.

## Current Runtime Configuration

The main app setup is in [src/main.cpp](./src/main.cpp).

The display currently uses a `260 x 218` viewport and supports two SPI clock configurations:

* `QUIET` mode -> `10 MHz` SPI
* `ORIGINAL` mode -> `20 MHz` SPI

The code also contains a software frame-timing system through `setTargetFps()`. These values are **frame-rate targets/caps**, not measurements of the actual achieved frame rate.

Actual display performance has not yet been formally benchmarked on the hardware.

### SPI and Frame Rate

Reducing the SPI clock from `20 MHz` to `10 MHz` halves the available display-transfer bandwidth. It does **not**, by itself, prove that the actual frame rate changed from `60 FPS` to `30 FPS`.

The current renderer uses RGB565, which requires `16 bits` per pixel. With the current `260 x 218` viewport, one complete viewport contains:

```text
260 x 218 = 56,680 pixels

56,680 x 16 bits = 906,880 bits per frame
```

Ignoring all command, rendering, and SPI overhead, the theoretical transfer limits are therefore approximately:

```text
20 MHz / 906,880 bits = 22.1 frames/second

10 MHz / 906,880 bits = 11.0 frames/second
```

These are theoretical upper limits for transferring the complete viewport, not measured FPS figures. Real performance will be lower because the Pico also has to render each strip, send display commands, run game logic, process input, and perform other work.

The renderer's `present()` function currently processes and transfers the complete configured viewport each frame using strip buffers. The strip buffer reduces RAM requirements, but it does not reduce the total number of pixel bytes that must be transferred for a complete frame.

For this reason, older references to `60 FPS` and `30 FPS` should be understood as software timing targets rather than verified display frame rates.

A hardware benchmark is still required to determine the actual achieved FPS in each display mode.

To use the full landscape viewport:

```cpp
display.setViewport(320, 240);
```

A full `320 x 240` RGB565 frame requires even more SPI bandwidth, so its theoretical maximum frame rate is lower than the current `260 x 218` viewport.

## Controls

### Menu

* `UP` / `DOWN` move selection
* `A` or `START` confirm
* `B` or `SELECT` go back

### Settings

* open from the `SETTINGS` item in the game-select screen
* switch `DISPLAY MODE` between `QUIET` and `ORIGINAL`
* toggle `GAME AUDIO` on or off
* run `NOISE DIAGNOSTIC` from the settings screen

### Plumber Man

* `LEFT` / `RIGHT` move
* `A` or `UP` jump
* `B` run
* `START` restart
* `SELECT` pause
* pause menu: return to menu or change volume

### Sky Dodger

* `A` or `START` steps through calibration
* tilt the device to steer
* `START` restart after crashing
* `SELECT` pause
* pause menu: return to menu or change volume

## Noise Diagnostic

Open the `SETTINGS` item from the game-select screen, then choose `NOISE DIAGNOSTIC`.

Inside the diagnostic:

* `UP` / `DOWN` select an item
* `A`, `LEFT`, or `RIGHT` toggle the selected item
* `START` exits back to the normal menu

If `GAME AUDIO` is enabled in settings, the diagnostic keeps the audio playing while you toggle:

* `SCREEN LOAD` to turn continuous display SPI updates on or off
* `BACKLIGHT` to turn the display backlight on or off
* `IMU POLL` to turn repeated IMU reads on or off

Menu screens stay silent. Audio is used in the games and in the diagnostic only.

## Notes

* The display renderer uses strip buffers instead of a full framebuffer to keep RAM usage reasonable on the Pico W.
* The default display SPI clock is reduced to `10 MHz` to reduce display-related audio noise.
* The original display mode uses a `20 MHz` SPI clock.
* The configured FPS values are software timing targets and should not be interpreted as measured display frame rates.
* The actual achieved frame rate at `10 MHz` and `20 MHz` has not yet been formally measured.
* `Sky Dodger` calibrates at startup and auto-detects the steering axis.
* The codebase is still growing, so the API and game structure may continue to change as PicoBoy develops.
