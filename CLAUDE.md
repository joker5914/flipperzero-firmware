# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

RogueMaster (RM) custom firmware for the Flipper Zero, a fork of the official firmware (OFW, `flipperdevices/flipperzero-firmware`) that pulls in changes from other custom firmwares (mostly Unleashed/`DarkFlippers` and Momentum) and bundles roughly 900 community apps. The code is C for the STM32WB55 (`targets/f7`), built with **fbt**, the Flipper Build Tool, which wraps SCons and lives in `SConstruct`, `firmware.scons`, `site_scons/` and `scripts/fbt_tools/`.

## Build commands

`./fbt` downloads and uses its own ARM toolchain (`scripts/toolchain/`), so you don't install one by hand. Submodules are needed: `git clone --recursive` or `git submodule update --init --recursive`.

- `./fbt`: build the firmware (default target `fw_dist`, output in `dist/` and `build/`)
- `./fbt updater_package`: build a self-update `.tgz` package (what releases ship; see `buildRelease.sh`)
- `./fbt flash_usb_full`: build and install over USB (qFlipper must be closed)
- `./fbt fap_<appid>`: build a single app as a `.fap`
- `./fbt launch APPSRC=<appid>`: build, upload and run one app on a connected Flipper
- `./fbt faps` / `./fbt fap_dist`: build all external apps
- `./fbt dolphin_ext`: build the SD-card animation assets
- `./fbt firmware_cdb`: generate `compile_commands.json` for clangd without building
- `./fbt cli`: open a USB CLI session to the device

CI builds with `./fbt TARGET_HW=7 copro_dist updater_package fap_dist`.

Build options come from `fbt_options.py` (`COMPACT=1`, `DEBUG=0`, `FIRMWARE_APP_SET`, the `FIRMWARE_APPS` sets). Put local overrides in `fbt_options_local.py`, or pass them on the command line, e.g. `./fbt COMPACT=1 DEBUG=0 ...`. `documentation/fbt.md` lists every target and parameter.

## Lint and format

- `./fbt format` / `./fbt lint`: clang-format for C/C++ (`.clang-format`). CI runs `./fbt -s lint`.
- `./fbt format_py` / `./fbt lint_py`: black for Python, SCons files and `application.fam` manifests
- `./fbt format_img` / `./fbt lint_img`: image assets
- `./fbt format_all` / `./fbt lint_all`

## Tests

Unit tests run **on the device**, not on the host. The test app is `applications/debug/unit_tests`, and each suite is a `PLUGIN` app under `tests/`, registered in that app's `application.fam`.

1. Build the test firmware: `./fbt COMPACT=1 DEBUG=0 FIRMWARE_APP_SET=unit_tests updater_package`. `DEBUG=0` is required because a debug image overlaps the radio stack's flash region. `DEBUG=0` also compiles out `furi_assert()`; `furi_check()` still fires.
2. Flash it together with the SD resources from `build/latest/resources`.
3. In the device CLI, run `unit_tests` for all suites, or `unit_tests <test_name>` for one suite (the names are the plugin appids).

Test assets are in `applications/debug/unit_tests/resources/unit_tests/`. `documentation/UnitTests.md` gives the per-protocol formats (for example `.irtest`).

## Architecture

- **`furi/`**: the core OS layer on top of FreeRTOS: threads, message queues, mutexes, the record registry (`furi_record_open("gui")` and similar) and checks (`furi_check`/`furi_assert`).
- **`targets/f7/`**: the hardware target. It contains `furi_hal/` (the HAL that apps call), linker scripts, the BLE glue, and **`api_symbols.csv`**. `targets/f18` is a variant without radio hardware and has its own `api_symbols.csv`.
- **`lib/`**: shared libraries for the protocol stacks (`subghz`, `nfc`, `infrared`, `lfrfid`, `ibutton`), `toolbox`, `flipper_format`, `mjs` (the JS engine) and submodules (FreeRTOS, mlib, nanopb, mbedtls, the STM32 HAL). **`lib/cfw/`** holds RM-specific code: CFW settings, asset packs, name spoofing and the Game Menu.
- **`applications/`**: every app is a directory with an `application.fam` manifest (`App(appid=..., apptype=..., entry_point=..., fap_category=...)`; see `documentation/AppManifests.md`).
  - `services/`: background services (gui, input, storage, cli, rpc, bt, desktop, loader, notification, power, ...) that are always in the firmware.
  - `main/`: built-in apps (subghz, nfc, infrared, lfrfid, ibutton, bad_usb, gpio, archive, u2f) and `cfw_app` (the RM "CFW Settings" app).
  - `system/`: built-in system apps (updater, js_app, hid_app, and the RM additions nice_o_code, seed_capturer and others).
  - `settings/`, `debug/`, `drivers/`, `examples/`.
  - `external/`: about 900 community apps of type `EXTERNAL`, built as `.fap` files for the SD card rather than linked into the firmware. Each keeps its upstream `README.md`/`changelog.md`/`LICENSE`.
- **`applications_user/`**: a place for your own local apps; fbt picks them up automatically.
- **`assets/`**: icons (turned into C sources by the `icons` target), dolphin animations, protobuf definitions (a submodule), and SD-card `resources/`.

### API versioning (`api_symbols.csv`)

Apps loaded from the SD card can call only the symbols marked `+` in `targets/f7/api_symbols.csv`. The file's `Version` row is the API version (currently 88.x, also shown in `CHANGELOG.md`). If you change a header listed in `SDK_HEADERS`, fbt stops the build, updates the CSV, and adds the new entries marked `?`. Change each one to `+` (exported) or `-` (hidden), then rebuild. Keep `targets/f18/api_symbols.csv` in step. Bump the major version by hand only when an existing `+` entry becomes `-`. `documentation/AppsOnSDCard.md` has the details.

### GUI app pattern

Most apps use `ViewDispatcher` + `SceneManager`. A `scenes/` directory holds `*_scene_config.h`, an X-macro list of the scenes, and one `*_scene_<name>.c` per scene, each with `on_enter`, `on_event` and `on_exit` handlers. The app struct owns the views (submenu, popup, text_input, widget and so on), which are registered with the dispatcher by ID. `applications/main/subghz`, `nfc` and `cfw_app` are good references.

## Conventions

- C style (`CODING_STYLE.md`): 4-space indent. Types are `PascalCase`; functions are `snake_case` and start with the file or module name (`subghz_keystore.h`, `SubGhzKeystore`, `subghz_keystore_read`). Constructors and destructors are `*_alloc` / `*_free`. File and directory names must match `^[0-9A-Za-z_]+` (the linter enforces this). Run `./fbt format` before committing.
- Format Python with black.
- Changes pulled from upstream are logged as bullets in the `ReadMe.md` "Latest Updates" section, credited to the author and linked to the source commit or PR. User-facing firmware changes also go in `CHANGELOG.md`. Adding or updating an external app usually means `applications/external/<appid>/` plus a ReadMe bullet.
- Treat `applications/external/*` as vendored upstream code: keep changes small and say in the ReadMe entry what RM changed.
