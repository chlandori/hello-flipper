---
description: 'Guidance for building, running, and debugging Flipper Zero FAP applications with uFBT.'
applyTo: '**'
---

# Flipper Zero app development (uFBT)

## Toolchain

- Builds use **uFBT** (micro Flipper Build Tool), installed via `pip install ufbt`. Never
  hand-roll Makefiles/CMake — uFBT owns the build.
- All commands run from the app folder root (the folder containing `application.fam`).
- On Windows use PowerShell; if `ufbt` is not on PATH, run `python -m ufbt`.
- First-time setup in a folder: `ufbt create APPID=<app_id>` then `ufbt vscode_dist`.
  `APPID` may only contain `a-z`, `0-9`, and `_`.

### Common commands

| Task | Command |
| --- | --- |
| Scaffold a new app | `ufbt create APPID=hello_flipper` |
| Generate VS Code build/debug config | `ufbt vscode_dist` |
| Build the FAP | `ufbt` (or `Ctrl+Shift+B` in VS Code) |
| Build + upload + launch on device | `ufbt launch` |
| Open the Flipper CLI | `ufbt cli` (then `log` / `log info` to see `FURI_LOG_*` output) |
| Update the SDK | `ufbt update` |
| Flash firmware matching the SDK | `ufbt flash_usb` |
| Clean build artifacts | `ufbt -c` |

## Expected project structure

```
<app_folder>/
├── application.fam      # app manifest read by uFBT
├── <app_id>.c           # entry point
├── <app_id>.png         # 10x10 1-bit app icon
├── images/              # PNGs auto-converted to <app_id>_icons.h
│   └── .gitkeep
└── .github/workflows/build.yml
```

Build output lands in `dist/` as `<app_id>.fap`. Do not commit `dist/` or `.ufbt/`.

## application.fam

```python
App(
    appid="hello_flipper",
    name="Hello Flipper",
    apptype=FlipperAppType.EXTERNAL,
    entry_point="hello_flipper_app",
    stack_size=1 * 1024,
    fap_category="Examples",
    fap_icon="hello_flipper.png",
    fap_icon_assets="images",
)
```

- `entry_point` must exactly match the C function name.
- `apptype=FlipperAppType.EXTERNAL` for FAPs.
- Only set `fap_icon_assets` if the `images/` folder exists; it is what generates
  `<app_id>_icons.h`.
- Declare optional SDK modules with `requires=[...]` when using them (e.g. `"gui"`,
  `"dialogs"`, `"storage"`, `"notification"`).

## C entry point conventions

```c
#include <furi.h>
#include <hello_flipper_icons.h>

int32_t hello_flipper_app(void* p) {
    UNUSED(p);
    FURI_LOG_I("HelloFlipper", "Hello world");
    return 0;
}
```

- Signature is always `int32_t <entry_point>(void* p)`; return `0` on success.
- Use `UNUSED(p)` to silence the unused-parameter warning.
- Only include `<<app_id>_icons.h>` when `images/` contains PNGs; otherwise the header
  does not exist and the build fails.

### Minimal dialog GUI

```c
#include <furi.h>
#include <dialogs/dialogs.h>

int32_t hello_flipper_app(void* p) {
    UNUSED(p);
    DialogsApp* dialogs = furi_record_open(RECORD_DIALOGS);
    DialogMessage* message = dialog_message_alloc();
    dialog_message_set_header(message, "Hello world", 64, 20, AlignCenter, AlignTop);
    dialog_message_set_text(message, "I'm hello_flipper!", 64, 32, AlignCenter, AlignTop);
    dialog_message_show(dialogs, message);
    dialog_message_free(message);
    furi_record_close(RECORD_DIALOGS);
    return 0;
}
```

## Firmware API rules

- Use FURI APIs, not libc/POSIX: `furi_string_*` instead of `char*`/`str*`,
  `furi_delay_ms()` instead of `sleep`, `furi_hal_*` for hardware.
- Memory: `malloc()` on Flipper aborts on failure, so never null-check it; always pair
  every `_alloc()` with its matching `_free()`.
- Every `furi_record_open()` needs a matching `furi_record_close()` before returning.
- Screen is 128x64, 1-bit. Keep `stack_size` small (1–2 KB typical); raise it only if the
  app crashes with a stack overflow.
- Logging macros: `FURI_LOG_E/W/I/D/T(tag, fmt, ...)`. Keep the tag short and constant.

## Troubleshooting

- **SDK version mismatch on launch** — device firmware and uFBT SDK differ. Run
  `ufbt update`, and if needed `ufbt flash_usb` to reflash the device.
- **Device not found** — close qFlipper and any other app holding the serial port, then
  retry.
- **`<app_id>_icons.h` not found** — the `images/` folder is missing/empty or
  `fap_icon_assets` is not set in `application.fam`.
- **App builds but doesn't appear on device** — check `fap_category`; the FAP is installed
  under `SD Card/apps/<fap_category>/`.

## Reference

- Tutorial: https://github.com/flipperdevices/flipperzero-firmware/blob/dev/documentation/app_dev/app_dev_your_first_app_in_c.md
- uFBT: https://github.com/flipperdevices/flipperzero-ufbt
