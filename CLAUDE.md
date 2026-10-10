# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ESP32 firmware that creates a Wi-Fi access point + HTTP/HTTPS server. When a PS5 connects, it serves a web exploit that installs a payload launcher via a PS5 WebKit vulnerability. Supports PS5 FW 7.00–13.60 via two exploit chains: "Poops" (FW 7.00–12.00) and "Relapse" (FW 7.00–13.60).

The actual project lives in `ps5-webkit-autoloader-esp32/`.

## Build Commands

All commands run from `ps5-webkit-autoloader-esp32/`.

```bash
# Install dependencies (arduino-cli, Node.js minifiers, openssl, etc.)
./install-deps.sh

# Build all targets (pico, s2, s3, c3)
make

# Build a specific ESP32 variant
make pico    # ESP32 Pico — 4 MB flash
make s2      # ESP32-S2
make s3      # ESP32-S3
make c3      # ESP32-C3
make pico-8m # ESP32 Pico — 8 MB flash

# Build with USB serial/JTAG debug enabled
make debug

# Web asset pipeline only
make minimize   # minify to autoloader.mini/
make datadir    # minify + gzip to data/ (LittleFS staging)
make littlefs   # build LittleFS image only

make clean      # remove build/, data/, autoloader.mini/
```

Output binaries: `build/<target>/esp32-arduino.<target>.merged.bin` — a padded 4 MB image (bootloader + partition table + app + LittleFS) ready to flash directly.

## Build Pipeline

Each target runs these steps automatically:
1. Generate self-signed TLS cert for `manuals.playstation.net` via `openssl`
2. Embed cert/key into `server_certs.h` via `embed_cert.py`
3. Patch the Arduino DNSServer library to add a domain allowlist via `prepare-dns-library.py`
4. Minify web assets (JS/CSS/HTML/JSON/SVG) and gzip-compress them into `data/`
5. Build LittleFS image from `data/` via `mklittlefs`
6. Compile `esp32-arduino.ino` via `arduino-cli`
7. Merge all images into a single binary via `esptool.py`

## Architecture

**Firmware** (`esp32-arduino.ino`) — single Arduino sketch with four subsystems in a cooperative loop:

- **Wi-Fi AP** — SSID `ESP32_PORTAL`, password `12345678`, IP `192.168.4.1`
- **DNS Server** (patched DNSServer library, UDP 53) — resolves only `manuals.playstation.net`, `ena.net.playstation.net`, `www.msftconnecttest.com` to `192.168.4.1`; NXDOMAIN for everything else
- **HTTP Server** (Arduino `WebServer`, port 80) — serves connectivity check endpoints, then falls back to LittleFS file serving with gzip support
- **HTTPS Server** (`esp_https_server` / IDF httpd, port 443) — same TLS cert bound to `manuals.playstation.net`; same file serving logic

**Manifest versioning** — on each boot, `esp_random()` generates a 16-hex-digit version string injected dynamically into `.appcache`/`.manifest`/`.cache` files, forcing the PS5 AppCache to re-check assets on every fresh boot without writes to flash.

**Web frontend** (`autoloader/`) — vanilla JS, no framework:
- `index.html` — detects PS5 FW from `navigator.userAgent`, redirects to the appropriate cache trigger page
- `slopkit_cache.html` / `relapse_cache.html` — trigger AppCache install of the selected exploit chain
- `app/0.6.1/` — versioned exploit payloads (`poops.html`, `relapse.html`, `elfldr-ps5.elf`, `kexp-ps5.bin`, `autoloader.elf`)
- AppCache completion is verified via `app/0.6.1/__complete__` marker file

**Python tooling:**
- `embed_cert.py` — PEM → C header for TLS cert/key
- `externalize_autoloader.py` — extracts base64-embedded ELF/BIN payloads from HTML pages into standalone files and updates AppCache manifests
- `prepare-dns-library.py` — patches Arduino DNSServer source at build time to add domain allowlist

**Serial debugging:** `python serial_timestamp.py COM4 115200` (requires `pyserial`) — prefixes every serial line with elapsed time.

## No Automated Tests

There is no test suite. Testing requires physically flashing an ESP32 and connecting a PS5.
