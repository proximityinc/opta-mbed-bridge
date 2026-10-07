# opta-mbed-bridge

Mbed-to-Zephyr OTA bridge images for the Proximity OPTA access panels.

## Why this repo exists

The Mbed firmware (2025.2.0 and earlier) downloads OTA packages with mbedTLS
against a 48-root bundle that lacks Amazon Root CA 1, so it cannot fetch from
S3 or CloudFront (opta-cp `docs/audit/09`, finding B0). The bundle does hold
ISRG Root X1, and `raw.githubusercontent.com` serves a Let's Encrypt chain
cross-signed by it. That makes this the one host the deployed Mbed fleet can
update from. Proven 2026-07-30: a Zephyr image delivered through the Mbed OTA
path over Ethernet.

The Zephyr firmware never uses this repo. It streams images from the platform
over the connection it already trusts (opta-cp `docs/rebuild/11`), so every
release after the bridge rides the normal pipeline.

## What is here

| File | Carries | Size | md5 |
|---|---|---|---|
| `firmware-2026.1.6-opta-cp.ota` | `2026.1.6-opta-cp` (opta-cp tag `5dab5c6`), LZSS-compressed, Arduino OTA header with the compressed flag | 430,695 B | `36e3488ab94d9465dc455dac90da6176` |

The payload decompresses to the exact `zephyr.bin` inside the published
`access/2026.1.6-opta-cp/firmware.ota` (532,020 B), verified by round trip.

## Running a bridge update

1. Make the repo public for the window. Raw URLs need it: the Mbed client
   sends no token and follows no redirect, so Releases do not work.
2. Confirm the chain still verifies against the Mbed bundle. ISRG Root X1 must
   appear as an issuer:

   ```bash
   echo | openssl s_client -connect raw.githubusercontent.com:443 -servername raw.githubusercontent.com 2>/dev/null | grep -E "^ *[0-9]* *[si]:"
   ```

3. Set the panel's `OTA_URL` preference to

   `https://raw.githubusercontent.com/proximityinc/opta-mbed-bridge/main/firmware-2026.1.6-opta-cp.ota`

   through the platform's Access preference action (`SetPreferenceOnAccess`;
   the message is `{"action":"setPreferencesKeyValue","key":"OTA_URL","value":...}`
   on `access/{serial}/config`). The Mbed router persists the key. The
   platform's `open:instruct-firmware-update` cannot do this: it insists on a
   platform path and an S3-published version.
4. Send `{"action":"command","command":"REBOOT"}`. The OTA check runs on the
   next MQTT connect: it deletes the key, downloads, decompresses to the QSPI
   staging volume and hands off to the Arduino bootloader.
5. The first Zephyr boot imports identity from the Mbed filesystem read-only
   (opta-cp decision D5), so the panel comes back as the same device with no
   wizard. Readers must be OSDP; the legacy keypad protocol is gone (D7).
   Enrolling one is opta-cp `docs/readers.md`.
6. Make the repo private again. Raw content stays cached for a few minutes.

## Building a bridge image

From opta-cp with the release tag checked out and `builds/release` built by
`build-release.sh`:

1. LZSS-encode `builds/release/zephyr/zephyr.bin` with `tools/lzss.py` from
   the `mbed-maintenance` branch. It loads `./tools/lzss.dylib` relative to the
   working directory, so run it from a directory that has that file.
2. Wrap it: `python3 tools/bin2ota.py --compressed OPTA <lzss file> firmware-<version>.ota`
3. Verify: decode the payload back with `lzss.py --decode` and compare it to
   `zephyr.bin`; the header CRC covers everything after the 8-byte prefix.

`tools/otacheck.py` reports the compressed flag as a defect because the Zephyr
firmware refuses such a package. For a bridge image that is the point.
