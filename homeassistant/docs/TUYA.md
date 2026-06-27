# Tuya Local Device Integration Guide

A reference for adding Tuya/Smart Life WiFi devices to Home Assistant with local control
(no cloud dependency at runtime). Written after first device setup in June 2026 — follow
this when you get a new Tuya device and have forgotten how it all works.

---

## The Chain

```
Smart Life app ──── pairs & controls devices via Tuya cloud
       │
       │  (link account to project)
       ▼
Tuya IoT Platform (iot.tuya.com) ──── developer portal, gives access to local keys
       │
       │  (tinytuya wizard downloads keys)
       ▼
tinytuya (Python tool) ──── extracts local device keys from cloud API
       │
       │  (key + device ID + IP → HA config)
       ▼
HA Tuya local integration ──── controls devices directly on LAN, no cloud at runtime
```

---

## One-Time Setup (already done — skip unless starting fresh)

### Tuya IoT Platform account
1. Register at https://iot.tuya.com
2. Cloud → Development → Create Cloud Project:
   - Data Center: **Central Europe** (for UK Smart Life accounts — NOT Western Europe)
   - Industry: Smart Home / Development Method: Smart Home
   - Authorise the default API services (IoT Core)
3. Note your **Access ID** and **Access Secret** from the project overview page
4. Devices tab → **Link App Account** → scan QR code with Smart Life app
   - Smart Life QR scanner: home screen top-right OR Me tab top-right
   - Phone must be on **2.4GHz WiFi** during the scan (Tuya devices don't support 5GHz)
5. Store Access ID and Secret in `.env` (see below) — regenerate Secret if ever exposed

**Important:** Free plan = one project, one data centre. If you need to change data centre,
delete the project and recreate it (you lose nothing — devices re-link via Smart Life).

---

## tinytuya on WSL2

### Installation (already done)
```bash
python3 -m venv ~/.venvs/tinytuya
source ~/.venvs/tinytuya/bin/activate
pip install tinytuya
```

### Running the wizard
```bash
source ~/.venvs/tinytuya/bin/activate   # activate venv
cd ~/projects                            # wizard saves files here
python3 -m tinytuya wizard
```

Wizard prompts:
- **API Key**: Access ID from iot.tuya.com project
- **API Secret**: Access Secret from iot.tuya.com project
- **Device ID**: get from Smart Life app (tap device → `...` → Device Information → Device ID)
  — or enter a known ID from a previous run
- **Region**: `eu` (Central Europe for UK)

### Files created in `~/projects/`
| File | Contents | Commit to git? |
|------|----------|---------------|
| `tinytuya.json` | API credentials + last device ID | **NO** — contains secrets |
| `devices.json` | All linked devices with local keys | **NO** — contains secrets |
| `snapshot.json` | Local scan results (IPs if found) | No |

**WSL2 limitation:** The local network scan always returns 0 devices because WSL2 NAT
blocks UDP broadcast discovery. This is fine — the local keys come from the Tuya cloud
API, not the scan. IPs are found separately (see below).

### Extracting what you need from devices.json
After the wizard completes:
```bash
cat ~/projects/devices.json
```
Note down for each device:
- `id` → Device ID (not secret, can be stored in docs)
- `key` → Local key (secret — store in `.env` or password manager)
- `mac` → MAC address (use to find IP in router DHCP list)

Then save keys to `.env` and delete the files:
```bash
rm ~/projects/devices.json ~/projects/tinytuya.json ~/projects/snapshot.json
```

---

## Finding Device IP Addresses

WSL2 scan won't find IPs. Options:
1. **Router DHCP list**: look up MAC address → IP (and assign a static DHCP reservation)
2. **Smart Life app**: tap device → `...` → Device Information → IP address (sometimes shown)
3. **nmap from WSL2**: `nmap -sn 192.168.1.0/24` then match by MAC

Once found, add a static DHCP reservation in the ASUS router for each Tuya device
(same process as cameras) so the IP never changes.

---

## Adding a Tuya Device to HA

HA Settings → Integrations → Add → **Local Tuya** (requires HACS — install first if needed)
or use the built-in **Tuya** integration (cloud-based, simpler but cloud-dependent).

**For local control** (preferred):
- Integration: `localtuya` (HACS)
- Per device: IP address, Device ID, local key, protocol version (try `3.3` or `3.4`)
- The mapping (dp codes) from `devices.json` → `mapping` section tells you what each
  data point controls (e.g. dp 20 = switch_led, dp 22 = brightness)

---

## Workflow for a New Tuya Device

1. **Pair with Smart Life app**: power on device, open Smart Life → Add Device → follow
   prompts. Phone on 2.4GHz WiFi. Power cycle 3× if it doesn't auto-detect.
2. **Check it appears** in Tuya IoT Platform → your project → Devices (may take a minute)
3. **Run tinytuya wizard** to get the local key:
   ```bash
   source ~/.venvs/tinytuya/bin/activate
   cd ~/projects
   python3 -m tinytuya wizard
   ```
   If config from last time is still in `tinytuya.json`, it'll use that — just confirm.
   Enter the new device's Device ID when prompted (or any linked device ID — wizard
   downloads the full list regardless).
4. **Extract key and MAC** from `devices.json`, find IP from router
5. **Assign static DHCP** reservation in ASUS router for the device MAC
6. **Add to HA** via localtuya integration
7. **Save key to `.env`** and delete `devices.json`
8. **Document in HOUSE.md** (room section) and below

---

## Device Inventory

Keys stored in `.env` (referenced by `TUYA_KEY_<NAME>`). Device IDs are not secret.

| Device | Room | Device ID | MAC | Model | Capabilities |
|--------|------|-----------|-----|-------|-------------|
| Sylstar Smart Ceiling Light | Kitchen | `bfd3f1a1c10821ddf1myge` | `d8:1f:12:6d:f7:d8` | SL-WIFI_R_CCT_24W | On/off, brightness (10–1000), colour temp (0–1000 cold→warm). **CCT only — not RGB** despite "colour" in work_mode |
| Intelligent heater | Main Bedroom | `6781360298f4abe6c8dc` | `98:f4:ab:e6:c8:dc` | GPH-EA/DA | On/off, mode (low/high/antifreeze), set temp (5–50°C), current temp sensor |

### DP code reference — Sylstar
| DP | Code | Type | Notes |
|----|------|------|-------|
| 20 | switch_led | Boolean | On/Off |
| 21 | work_mode | Enum | white/colour/scene/music — use `white` for normal use |
| 22 | bright_value | Integer 10–1000 | Brightness |
| 23 | temp_value | Integer 0–1000 | Colour temp: 0=cool white, 1000=warm white |
| 26 | countdown | Integer 0–86400 | Timer (seconds) |

### DP code reference — Intelligent heater
| DP | Code | Type | Notes |
|----|------|------|-------|
| 1 | switch | Boolean | On/Off |
| 2 | temp_set | Integer 5–50 | Target temperature °C |
| 3 | temp_current | Integer 0–50 | Current temperature °C (sensor) |
| 4 | mode | Enum | low / high / af (antifreeze) |
