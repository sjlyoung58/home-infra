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
| 1 | RLC-1224A  | Front Right   | 192.168.1.19  | EC:71:DB:64:A9:36 | v3.1.0.2174_23050816  | TBD              | 12MP |
| 2 | RLC-811A   | Front Left    | 192.168.1.84  | EC:71:DB:60:B5:BB | v3.1.0.764_21121708   | TBD              | ⚠ Dec 2021 — update all RLC-811As |
| 3 | RLC-811A   | Side Alley    | 192.168.1.117 | EC:71:DB:AD:74:47 | v3.1.0.764_21121708   | TBD              | |
| 4 | RLC-811A   | Rear Kitchen  | 192.168.1.118 | EC:71:DB:BD:C2:5D | v3.1.0.764_21121708   | TBD              | |
| 5 | RLC-811A   | Rear          | 192.168.1.159 | EC:71:DB:E6:B6:F8 | v3.1.0.764_21121708   | TBD              | |

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
| 1 | E1 Pro | Indoor1  | 192.168.1.141 | 60:FB:00:9A:1A:5C | v3.0.0.716_21112404  | TBD              | ⚠ Nov 2021 firmware — update |
| 2 | E1 Pro | Indoor2  | 192.168.1.168 | 60:FB:00:9A:2B:A0 | v3.0.0.716_21112404  | TBD              | ⚠ update firmware |

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
