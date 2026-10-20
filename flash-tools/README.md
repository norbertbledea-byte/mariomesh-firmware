# mariomesh-flash

Flashing toolkit and the confirmed-working MarioMesh firmware binaries for the
heltec-v3 nodes (Acasa, CN101D, Mobil).

## Current deployed version

**`2.8.2.fcc2261`** — confirmed via `--info` directly against Acasa's own USB
serial port that this is the exact build running on the mesh today. Do not
assume the latest commit on `main` (currently `19fda6b`) is what's deployed —
it may be ahead of what's actually flashed on the physical nodes. Always
verify against a live node's `firmwareVersion` before treating a newer build
as "the right one."

## Flashing a node (full chip erase + factory reset)

```
python -m esptool --chip esp32s3 --port COMx erase_flash
python -m esptool --chip esp32s3 --port COMx write_flash 0x0 firmware-heltec-v3-2.8.2.fcc2261.factory.bin
python -m esptool --chip esp32s3 --port COMx write_flash 0x670000 littlefs-heltec-v3-2.8.2.fcc2261.bin
```

The littlefs/spiffs partition offset for heltec-v3 is **0x670000** (from
`default_8MB.csv` in the firmware repo, `board_build.partitions` set in
`variants/esp32s3/heltec_v3/platformio.ini`) — NOT the generic 4MB-table
default of 0x300000 that `device-install.sh` falls back to without a
`.mt.json`. The `.mt.json` here has the correct offsets if you use that
script instead.

After flashing, re-apply on the device (nothing survives a factory erase):
```
python -m meshtastic --port COMx --set-owner "<Name>" --set-owner-short "<SHORT>"
python -m meshtastic --port COMx --set remote_hardware.enabled true   # only if this node drives relays
```

## Known issue: setting `lora.region` remotely does not persist on a freshly-erased node

Confirmed (2026-10-20) across three different firmware builds (7f00ee3,
fcc2261, and a from-source 19fda6b rebuild) that `--set lora.region EU_868`
over the admin/USB-serial CLI reports success but **the region reverts to
UNSET on the next reboot**, while every other `lora.*` field set the same
way (e.g. `hop_limit`) persists correctly. Acasa (long-provisioned, never
chip-erased) has a correctly-persisted region with this exact same firmware,
so it is not a firmware-version bug — it is specific to a device's first-ever
region assignment after a full `erase_flash`.

**Workaround that works:** set the region through the on-device physical
button menu instead of remotely. Long-press through the "Welcome to
Meshtastic" onboarding banner to reach the region picker, then short-press to
scroll to the desired region and long-press again to confirm. This commits
through `applyLoraRegion()` in `MenuHandler.cpp`, a completely different code
path from the AdminModule remote `set_config` handler, and does persist
correctly across reboots.

Root cause not fully isolated (suspected: first-time region assignment also
triggers PKI identity/keygen side effects in `AdminModule::handleSetConfig`'s
`lora` case that may be failing silently for a never-provisioned node) — not
worth chasing further unless it recurs; the on-device workaround is reliable.

## `stock-firmware/`

Vanilla upstream Meshtastic 2.7.26 builds for every board variant, kept only
as reference/fallback. Not version-controlled here (see `.gitignore`) — too
large and not our own work.
