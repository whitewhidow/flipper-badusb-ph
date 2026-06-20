# Bad USB (placeholders)

A Flipper Zero BadUSB app, forked from **RogueMaster** firmware's BadUSB, that adds
**`[placeholder]` substitution** and **per-payload configs**. Write payloads with
tokens like `[host]`, `[url]`, `[username]`; when you run one, the app asks you to
fill those tokens in — either by picking a saved **config** or by **typing values
manually** — and substitutes them before the DuckyScript executes.

It installs as a **separate app** (`appid = badusb_ph`) and does not touch the stock
BadUSB app. It is completely independent of the Android/Wear projects in this repo
(different language, different build system).

> **Firmware note.** This is built on RogueMaster's BadUSB, because that's the
> firmware on the target device. RogueMaster rewrote the HID layer (its own
> `BadUsbHidConfig`, a custom `ble_profile_hid_ext`, USB unlock logic), so a FAP
> built from *official* BadUSB sources compiles against RM but **HardFaults at HID
> init on an RM device**. Building on the RM base avoids that. As a consequence the
> code uses RM-specific APIs and is **not** buildable against official firmware /
> submittable to the official catalog as-is. RM firmware is GPLv3; this fork
> inherits that.

## Placeholder syntax

Anywhere in a `.txt` payload, write `[name]` where `name` is letters, digits or
underscores:

```
DELAY 1500
STRING Connecting to [host] on port [port]
ENTER
STRING Logged in as [username]
```

- Each distinct `[name]` is asked for once; the same `[name]` used multiple times
  gets the same value everywhere.
- A token that isn't a valid name (e.g. `[ ]`, `[a b]`) is left untouched.
- Payloads with **no** placeholders run exactly like stock BadUSB (no extra prompts).

`[...]` is not DuckyScript syntax, so existing payloads are unaffected.

## Run flow

1. Open the app, pick a payload from `/ext/badusb`.
2. If it contains placeholders you get a menu:
   - **a saved config** for this payload (if any), or
   - **Manual entry**.
3. Any placeholder not supplied by the chosen config is requested via the on-screen
   keyboard.
4. After manual entry you're offered **Save** — name it to store a config for reuse,
   or **Run** to launch without saving.
5. The script runs with every `[name]` replaced. Press **OK** to start/stop as usual.

## Configs

- Per payload (keyed by the payload's file name without extension).
- Stored on the SD card at:
  `/ext/apps_data/badusb_ph/configs/<payload_name>/<config_name>.cfg`
- Plain `FlipperFormat` files (`name: value` per line) — editable by hand if you like.

## Build & install

A prebuilt FAP for RogueMaster (target `f7`, API 87.8) is provided at
`dist/badusb_ph.fap` — just copy it to `/ext/apps/USB/` on the SD card (qFlipper or
the storage helper) and it appears under **Apps → USB**.

To rebuild from source you build it **in-tree against the RogueMaster firmware**
(not `ufbt`, because it depends on RM's HID layer):

```bash
# 1. Clone the RM firmware tag matching your device (see device_info: firmware_commit)
git clone --depth 1 -b <RM-tag> --filter=blob:none \
    https://github.com/RogueMaster/flipperzero-firmware-wPlugins.git rm-fw

# 2. Drop this app in as a user app
cp -r flipper_badusb_ph rm-fw/applications_user/badusb_ph

# 3. Build (reuses RM's toolchain; ufbt's v39 toolchain can be symlinked to skip a download:
#    ln -s ~/.ufbt/toolchain rm-fw/toolchain )
cd rm-fw && ./fbt fap_badusb_ph
#    -> build/f7-firmware-C/.extapps/badusb_ph.fap
```

Install over USB with the SDK's storage helper (replace `<PORT>`, e.g. `/dev/ttyACM0`):

```bash
python3 ~/.ufbt/current/scripts/storage.py -p <PORT> send \
    build/f7-firmware-C/.extapps/badusb_ph.fap /ext/apps/USB/badusb_ph.fap
```

> An earlier iteration of this app was `ufbt`-based and built against the official
> SDK — it runs on official firmware but **crashes on RogueMaster**. This tree is the
> RM-based version that runs on the target device.

### Keyboard layouts

The app reuses the layouts that official firmware already installs at
`/ext/badusb/assets/layouts`. If a layout file is missing it falls back to the
built-in en-US map.

## Sample payload

`sample_payloads/ph_demo.txt` is a benign demo (types text only) that uses
`[host]`, `[port]`, `[url]` and `[username]`. Copy it to `/ext/badusb/` to try the
flow:

```bash
# with the official storage helper bundled in the SDK
python3 ~/.ufbt/current/scripts/storage.py -p <PORT> send \
    sample_payloads/ph_demo.txt /ext/badusb/ph_demo.txt
```

## What was changed vs RM's stock BadUSB

- `helpers/placeholder.{c,h}` — new: line-based scanning (skips `REM` comment
  lines), per-line substitution, and config load/save via `FlipperFormat`.
- `helpers/ducky_script.c` — applies substitution to each line before it is parsed
  (`placeholder_apply` in `ducky_script_execute_next`); added
  `bad_usb_script_set_placeholders` + a `placeholder_map` field on the script.
- `scenes/bad_usb_scene_ph_config.c`, `..._ph_input.c`, `..._ph_save.c` — new scenes
  for choosing a config, entering values, and saving (reuse RM's `TextInput`; add a
  `Submenu` view for the config picker).
- `scenes/bad_usb_scene_file_select.c` — scans the chosen payload and branches into
  the placeholder flow when tokens are present (otherwise behaves exactly like stock).
- `scenes/bad_usb_scene_work.c` — Back returns straight to the file browser, and
  falls through to normal exit when launched directly on a file.
- `application.fam` — `appid=badusb_ph`, `EXTERNAL`; otherwise RM's BadUSB verbatim
  (its own `images/`, `ble_profile`, 4 KB stack).
