<div align="center">

# nRF53-OTA-server

**Cloud Server Repository for Over-The-Air Firmware Delivery (nRF7002 DK / nRF5340)**

[![GitHub Repo](https://img.shields.io/badge/Server%20Repo-nRF53--OTA--server-blue?logo=github)](https://github.com/Prateek-303/nRF53-OTA-server)
[![Firmware Repo](https://img.shields.io/badge/Firmware%20Repo-Firmware--OTA-informational?logo=github)](https://github.com/Prateek-303/Firmware-OTA)
[![SDK](https://img.shields.io/badge/nRF%20Connect%20SDK-v3.2.4-orange)](https://developer.nordicsemi.com)
[![Board](https://img.shields.io/badge/Board-nRF7002%20DK-green)](https://www.nordicsemi.com/Products/Development-hardware/nRF7002-DK)
[![TLS](https://img.shields.io/badge/TLS-1.2%20%2F%20mbedTLS-red)](https://tls.mbed.org)

*by Prateek Baraiya*

</div>

---

## 📌 Overview

This repository is the **cloud backend** for the OTA firmware update system. It uses a plain GitHub repository served via `raw.githubusercontent.com` as a zero-cost, globally available, TLS-secured HTTPS server to deliver signed firmware binaries and JSON manifests to nRF5340-based edge devices. No extra configuration — just push to `main` and the files are instantly live.

> **Firmware Repository:** The source code for the firmware lives at:
> 👉 [github.com/Prateek-303/Firmware-OTA](https://github.com/Prateek-303/Firmware-OTA)

---

## 📁 Repository Structure

```
nRF53-OTA-server (this repo)
 ┣ manifest.json              ← Auto-updated: version, size, SHA256, CRC32
 ┣ authorized_devices.json   ← MAC address whitelist for security
 ┣ app_update_1.0.1.bin      ← V1 signed firmware (LED blink only)
 ┣ app_update_1.0.2.bin      ← V1.1 signed firmware (I2C pin fix)
 ┣ app_update_2.0.0.bin      ← V2 signed firmware (LED + BMP280 + TMP117)
 ┣ LICENSE
 └ README.md                 ← This file
```

---

## ⚙️ System Architecture

### High-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER MACHINE                            │
│                                                                     │
│  ┌──────────────┐    build_ota.bat     ┌──────────────────────────┐│
│  │  VS Code /   │ ─────────────────▶  │   west build             ││
│  │  Source Code │                     │   (Zephyr + nRF SDK)     ││
│  └──────────────┘                     └───────────┬──────────────┘│
│                                                   │               │
│                                         zephyr.signed.bin         │
│                                                   │               │
│                                       ┌───────────▼──────────────┐│
│                                       │  update_manifest.py      ││
│                                       │  • Compute SHA-256        ││
│                                       │  • Compute IEEE CRC-32   ││
│                                       │  • Write manifest.json   ││
│                                       └───────────┬──────────────┘│
│                                                   │               │
│                                          git push origin main     │
└───────────────────────────────────────────────────┼───────────────┘
                                                    │
                                    ┌───────────────▼───────────────┐
                                    │     GITHUB REPOSITORY         │
                                    │  (raw.githubusercontent.com)  │
                                    │                               │
                                    │  ┌─────────────────────────┐  │
                                    │  │  manifest.json           │  │
                                    │  │  app_update_X.Y.Z.bin   │  │
                                    │  │  authorized_devices.json│  │
                                    │  └─────────────────────────┘  │
                                    └───────────┬───────────────────┘
                                                    │  HTTPS / TLS 1.2
                                                    │  (every 24 hours)
                                    ┌───────────────▼───────────────┐
                                    │         nRF7002 DK            │
                                    │       (nRF5340 SoC)           │
                                    │                               │
                                    │  1. Wi-Fi connect             │
                                    │  2. Fetch manifest.json       │
                                    │  3. Compare versions          │
                                    │  4. Stream .bin (1KB chunks)  │
                                    │  5. Live CRC-32 check         │
                                    │  6. Final SHA-256 check       │
                                    │  7. MCUboot image swap        │
                                    │  8. Reboot → new firmware ✅  │
                                    └───────────────────────────────┘
```

---

## 🛠️ Deployment Workflow

Follow these steps every time you want to push an OTA update to the fleet wirelessly.

### Step 1 — Make Code Changes in Firmware Repo
Edit the source files in your local `Firmware-OTA` repository.

### Step 2 — Bump the Version
Open the `VERSION` file in the firmware repo and increment:

```
VERSION_MAJOR = 1       ← Major: breaking/architectural changes
VERSION_MINOR = 0       ← Minor: new features (e.g., V2 sensors)
PATCHLEVEL = 2          ← Patch: bug fixes, pin corrections
```

> ⚠️ The OTA engine only downloads if manifest version > running version.

### Step 3 — Build & Auto-Push
Run the build script in your firmware repo terminal:

```powershell
.\build_ota.bat
```

The script will automatically compile, calculate hashes, update `manifest.json`, and push the new binary to this server repository.

### Step 4 — Wait for GitHub Cache (Important!)
`raw.githubusercontent.com` has a short server-side cache. After every `git push`, wait about **1 to 2 minutes** before resetting the board.

### Step 5 — Trigger OTA on the Board
Press the **RESET** button on the nRF7002 DK. The board will connect to Wi-Fi, poll the manifest, and begin downloading with a live progress bar.

---

## 📋 manifest.json — Field Reference

The manifest is auto-generated by the build script. **Never edit it manually.**

```json
{
    "version": "1.0.2",
    "file_size": 652400,
    "image": "/app_update_1.0.2.bin",
    "sha256": "c4d0505dc6c43e1d11bd4037052c01423ed872f436e9cf6791fa7327b22948c6",
    "crc32": "3c5ce300"
}
```

| Field | Description |
|-------|-------------|
| `version` | Semantic version. Board downloads only if this > running version. |
| `file_size` | Exact byte count. Used to detect incomplete downloads. |
| `image` | URL path relative to the GitHub root. |
| `sha256` | Full SHA-256 hex digest. Verified by MCUboot post-download. |
| `crc32` | IEEE 802.3 CRC-32 hex. Verified live during streaming. |

---

## 🔐 Device Authorization

Only MAC addresses listed in `authorized_devices.json` will receive updates:

```json
{
  "authorized_devices": [
    "bf77015cc222c893"
  ]
}
```

To authorize a new board:
1. Boot the board and open the serial terminal.
2. Find the line: `[OTA] Device MAC: bf77015cc222c893`
3. Add that MAC string to the array in `authorized_devices.json`.
4. Commit and push.

---

## 🧠 Challenges, Error Codes & Engineering Solutions

### ❌ `-113` — EHOSTUNREACH (Host Unreachable)
**Cause:** Board connected to Wi-Fi but DNS/gateway cannot reach `raw.githubusercontent.com`.  
**Fix:** Verify router has internet. The OTA thread retries automatically every 24 hours.

### ❌ `-116` — ETIMEDOUT (Connection Timed Out)
**Cause:** TLS handshake RSA/ECC asymmetric math takes 5–8 seconds on nRF5340. TCP socket default timeout was shorter.  
**Fix:** Extended in firmware `prj.conf` via `CONFIG_NET_MGMT_EVENT_QUEUE_TIMEOUT=5000`. Ensure strong Wi-Fi signal.

### ❌ `-22` — EINVAL (Invalid Argument)
**Cause:** Networking stack tried to open an IPv6 socket when `CONFIG_NET_IPV6=n` was set to save memory.  
**Fix:** Resolved in firmware `prj.conf` with `CONFIG_NET_IPV4=y` and `CONFIG_NET_IPV6=n`.

### ❌ `-12` / `-ENOMEM` (Out of Memory)
**Cause:** mbedTLS allocates large buffers to parse GitHub's full X.509 certificate chain. Default Zephyr heap is too small.  
**Fix:** Resolved with `CONFIG_MBEDTLS_HEAP_SIZE=81920` (80 KB) and `CONFIG_HEAP_MEM_POOL_SIZE=120000` (120 KB).

### ❌ `-0x2700` — mbedTLS X509 Certificate Verify Failed
**Cause:** Embedded devices have no OS trust store. The board rejected GitHub's certificate because it had no trusted root CA.  
**Fix:** Manually extracted the **ISRG Root X1** PEM certificate, hardcoded it in `github_certs.h`, and registered it with `tls_credential_add()`.

### ❌ `region FLASH overflowed` (Linker Error)
**Cause:** Wi-Fi stack + mbedTLS + MCUboot + application exceeded primary slot size.  
**Fix:** Pruned the stack aggressively in `prj.conf` (`CONFIG_SIZE_OPTIMIZATIONS=y`, `CONFIG_NET_IPV6=n`, `CONFIG_WIFI_NM_WPA_SUPPLICANT_WPA3=n`), saving 50+ KB of flash.

### ❌ Net Socket Log Flooding (UI Issue)
**Cause:** Serial terminal was flooded with `net_sock` debug lines, destroying the progress bar display.  
**Fix:** Disabled in `prj.conf` with `CONFIG_NET_LOG=n`. Implemented a carriage-return (`\r`) overwriting `printk` progress bar.

---

## ⚠️ Important Things to Keep in Mind

| Warning | Detail |
|---------|--------|
| ⏱️ **GitHub Cache Delay** | Wait 1–2 minutes after `git push` before resetting the board. `raw.githubusercontent.com` propagates fast but not instantly. |
| 🔒 **Repo Must Be Public** | The firmware fetches files without auth tokens. A private repo returns HTTP 404 and OTA silently fails. |
| 🔐 **TLS Cert Expiry** | The ISRG Root X1 cert in `github_certs.h` must be renewed via OTA before it expires. |
| 🔢 **Version Must Increase** | Pushing the same version number triggers no OTA. Always bump `PATCHLEVEL` at minimum. |
| 🔄 **MCUboot Rollback** | If the new firmware boots but does not confirm itself within 10 seconds, MCUboot automatically reverts to the previous image. |
| 📡 **Poll Interval** | OTA thread polls every 24 hours. Press RESET to trigger an immediate check. |

---

<div align="center">
<em>by <strong>Prateek Baraiya</strong></em>
</div>
