# Camera Device Inventory

## Reolink POE Cameras (local API)

HA integration: built-in Reolink integration (no HACS). Local API, no cloud dependency.
Add each camera individually: Settings → Integrations → Add → Reolink.

IPs and details confirmed via Reolink app (2026-06-23).
Static DHCP reservations to be set in ASUS router before HA integration.
⚠ Firmware on all cameras is old — update via Reolink app before integrating with HA.

| # | Model      | Location      | Current IP    | Firmware              | HA Entity prefix | Notes |
|---|------------|---------------|---------------|-----------------------|------------------|-------|
| # | Model      | Location      | IP (static)   | MAC               | Firmware              | HA Entity prefix | Notes |
|---|------------|---------------|---------------|-------------------|-----------------------|------------------|-------|
| # | Model      | Location      | IP (static)   | MAC               | Firmware              | HA Entity prefix | Notes |
|---|------------|---------------|---------------|-------------------|-----------------------|------------------|-------|
| 1 | RLC-1224A  | Front Right   | 192.168.1.19  | EC:71:DB:64:A9:36 | v3.1.0.2174_23050816  | TBD              | 12MP; auto-update on; confirmed latest |
| 2 | RLC-811A   | Front Left    | 192.168.1.84  | EC:71:DB:60:B5:BB | v3.1.0.764_21121708   | TBD              | auto-update on; confirmed latest |
| 3 | RLC-811A   | Side Alley    | 192.168.1.117 | EC:71:DB:AD:74:47 | v3.1.0.764_21121708   | TBD              | auto-update on; confirmed latest |
| 4 | RLC-811A   | Rear Kitchen  | 192.168.1.118 | EC:71:DB:BD:C2:5D | v3.1.0.764_21121708   | TBD              | auto-update on; confirmed latest |
| 5 | RLC-811A   | Rear          | 192.168.1.159 | EC:71:DB:E6:B6:F8 | v3.1.0.764_21121708   | TBD              | auto-update on; confirmed latest |

### AI Detection binary sensors (per camera)
Once integrated, each camera exposes binary sensors for:
- `binary_sensor.<name>_motion`
- `binary_sensor.<name>_person`
- `binary_sensor.<name>_vehicle`
- `binary_sensor.<name>_pet`
- `binary_sensor.<name>_package`

Exact entity IDs to be filled in after HA integration (Phase 3a).

### Firmware
Check all cameras are on current firmware before integrating.
Reolink app → Device Settings → System → Firmware Upgrade.

---

## Reolink Indoor WiFi Cameras (local API)

HA integration: same built-in Reolink integration as POE cameras.
Currently not deployed — previously used as baby monitors for grandchildren visits.

| # | Model  | Location | IP (static)   | MAC               | Firmware             | HA Entity prefix | Notes |
|---|--------|----------|---------------|-------------------|----------------------|------------------|-------|
| # | Model  | Location | IP (static)   | MAC               | Firmware             | HA Entity prefix | Notes |
|---|--------|----------|---------------|-------------------|----------------------|------------------|-------|
| 1 | E1 Pro | Indoor1  | 192.168.1.141 | 60:FB:00:9A:1A:5C | v3.0.0.716_21112404  | TBD              | No auto-update; manual update needed before use |
| 2 | E1 Pro | Indoor2  | 192.168.1.168 | 60:FB:00:9A:2B:A0 | v3.0.0.716_21112404  | TBD              | No auto-update; manual update needed before use |

**E1 Pro status: low priority / deferred**
Not planned for initial HA integration. Potential future use: room presence detection
via person detection binary sensor (`binary_sensor.<name>_person`). If pursued, update
firmware manually via Reolink app first (no auto-update available on these models).
Note: no two-way audio in HA — Reolink app required for that.

---

## Frigate NVR (future consideration)

Frigate is a local NVR with AI object detection that integrates natively with HA.
More capable than Reolink's built-in detection — can do reliable person/vehicle
detection, clip recording triggered by detection, and more precise room presence.
Would run as a Docker container on the NAS alongside HA.

**Resource caveat — NAS has no hardware ML accelerator:**
NAS CPU: AMD Ryzen Embedded V1500B (4 cores, Zen, decent CPU — but AMD, no QuickSync).
Frigate's object detection inference is GPU/accelerator-intensive. Without hardware
acceleration, CPU-only inference across 5 cameras would be very heavy and likely
too slow for real-time detection (typically 1-5 FPS per camera on CPU vs 70+ FPS
with a Coral).

**Synology USB passthrough problem:**
Google Coral USB Accelerator on Synology is a known workaround situation — the DSM
Docker GUI doesn't expose USB passthrough at all. Workarounds require CLI container
startup, custom Docker images, or privileged containers, and have broken across DSM
updates. Not suitable for stable infrastructure. Don't use this approach.

**Better hosting options for Frigate:**

1. **Dedicated Intel N100 mini-PC (~£150)** — Intel QuickSync built in, Frigate supports
   it natively, no Coral needed, no USB hacks, dedicated host with no shared risk.
   Cleanest long-term solution.

2. **Pi 5 with Coral USB** — Pi 5 has proper USB Docker passthrough (no Synology caveats).
   But Pi 5 is already critical HA infrastructure (Z-Wave, Zigbee, nginx) — adding
   live video streams from 5 cameras is a significant extra load. Risky.

3. **Pi 5 with HAILO-8L AI HAT+ (~£70)** — Raspberry Pi's M.2 AI accelerator connects
   via PCIe (not USB), Frigate added HAILO support in 2024. However Pi 5 already has an
   NVMe HAT occupying the PCIe slot — HAILO would require removing it or a PCIe splitter
   solution. Not practical.

**Recommendation: park until after OH→HA migration is stable.** Then assess whether
room presence / clip recording use case justifies the investment. If yes, a dedicated
Intel N100 mini-PC is the cleanest path.

### Capabilities in HA
- `camera.<name>` — RTSP stream (viewable in dashboard)
- `binary_sensor.<name>_motion` — motion detected
- `binary_sensor.<name>_person` — person detected
- Pan/tilt control via HA (PTZ supported by integration)

### Limitations vs POE cameras
- **No two-way audio** in HA — use Reolink app for that (key omission for baby monitor use)
- No vehicle/pet/package AI detection (person + motion only)
- WiFi only — assign DHCP reservations when deploying rather than static IPs

### Usage notes
- Plug in and add to Reolink app first to configure WiFi, then add to HA by IP
- Useful if deployed: stream on a dashboard card, person-detection automations
- Two-way audio requires the Reolink app directly — HA cannot substitute for this

---

## Ring Doorbell (cloud-based)

HA integration: built-in Ring integration (no HACS). Cloud-relayed — requires Ring account.
Setup: Settings → Integrations → Add → Ring. Needs 2FA approval on phone during setup.

| Model                    | Location  | Status         | Ring account       |
|--------------------------|-----------|----------------|--------------------|
| Ring Video Doorbell Pro  | Front door | Not yet installed (arrived 2026-06-22) | TBD |

### Entities (after integration)
- `binary_sensor.<name>_ding` — doorbell press
- `binary_sensor.<name>_motion` — motion detected
- `camera.<name>` — live view (cloud-relayed RTSP)

### Notes
- Wired model — no battery management needed
- Motion zones configured in Ring app, respected by HA integration
- Stream has some latency due to cloud relay — not suitable for low-latency use
- Ring integration token refreshes automatically; re-auth needed if Ring account password changes
