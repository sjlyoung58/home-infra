# Camera Device Inventory

## Reolink POE Cameras (local API)

HA integration: built-in Reolink integration (no HACS). Local API, no cloud dependency.
Add each camera individually: Settings → Integrations → Add → Reolink.

| # | Model       | Location     | IP             | HA Entity prefix         | Notes                        |
|---|-------------|--------------|----------------|--------------------------|------------------------------|
| 1 | RLC-811A    | TBD          | TBD            | TBD                      | 4K 8MP, assign static IP     |
| 2 | RLC-811A    | TBD          | TBD            | TBD                      | 4K 8MP, assign static IP     |
| 3 | RLC-811A    | TBD          | TBD            | TBD                      | 4K 8MP, assign static IP     |
| 4 | RLC-811A    | TBD          | TBD            | TBD                      | 4K 8MP, assign static IP     |
| 5 | RLC-1224A   | TBD          | TBD            | TBD                      | 12MP, assign static IP       |

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

| # | Model       | Location     | IP             | HA Entity prefix | Notes                     |
|---|-------------|--------------|----------------|------------------|---------------------------|
| 1 | E1 Pro      | Not deployed | TBD (WiFi)     | TBD              | 5MP, pan/tilt, WiFi only  |
| 2 | E1 Pro      | Not deployed | TBD (WiFi)     | TBD              | 5MP, pan/tilt, WiFi only  |

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
