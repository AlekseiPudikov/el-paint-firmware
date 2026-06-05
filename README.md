# EL-Paint firmware (encrypted OTA)

Public host for **encrypted** EL-Paint block firmware. The phone app polls
[`manifest.json`](manifest.json), compares the listed version with the connected
device, and downloads the `.elpfw` package when an update is available.

## Why public hosting is safe
Packages are **AES-128-CBC encrypted** with a per-product key:
- **Confidentiality** — without the key the firmware can't be read, so the
  anti-clone property holds even in public.
- **No malware injection** — to be installed, a package must be encrypted with
  the correct key. A tampered/substituted package decrypts to garbage on the
  block; the CRC-gated `VERIFY` rejects it (no magic written) and the block stays
  recoverable in its bootloader. So even a fully compromised mirror can't flash
  foreign firmware.

Serve over HTTPS (GitHub does). The app refuses downgrades (won't install a
version ≤ the device's current one).

> ⚠️ **Never publish a dev-key package.** The dev key is the public FIPS-197 AES
> test vector — a dev `.elpfw` is trivially decryptable. Only publish packages
> built with the real per-product key (`provision.py encrypt`).

## `manifest.json`
```json
{
  "schema": 1,
  "block": {
    "version": 5,                 // integer firmware version (matches device report)
    "name": "EL-Paint Monoblock (STM32F103)",
    "url": "https://raw.githubusercontent.com/EL-paint/OTA-fw/main/firmware/block_v5.elpfw",
    "size": 11788,                // bytes (download sanity check)
    "sha256": "<hex>",            // app verifies after download (BL CRC32 is the final gate)
    "min_app": 4,                 // minimum compatible app version
    "notes": "Release notes shown in the app."
  }
}
```
A `version` of `0` / empty `url` means "no update available".

## Publishing a new firmware
1. Build the production app (Keil target `Monoblock`) and encrypt it with the
   **real** key: `provision.py encrypt Monoblock.hex -o block_v<N>.elpfw`.
2. Upload `block_v<N>.elpfw` as a GitHub **Release** asset (keeps binaries out of
   git history) — or commit it under `firmware/`.
3. Bump `block.version` and set `url` / `size` / `sha256` in `manifest.json`,
   then commit. The app picks it up on the next update check.
