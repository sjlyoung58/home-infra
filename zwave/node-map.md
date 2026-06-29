# Z-Wave Node Map

Z-Wave JS UI: http://192.168.1.222:8091
Controller home ID: 0xcd2fc96a
Z-Wave JS UI version: 11.5.2

Node names/locations not yet configured in Z-Wave JS UI (nodes.json is empty).
Table derived from OpenHAB items (controller was migrated from Pi 3B).

**Note:** WebSocket server enabled on port 3001. HA connected at ws://192.168.1.222:3001.

## GWPN1 post-migration configuration (apply to all GreenWave sockets)

After including each GWPN1 in the new Z-Wave network, set these parameters:

| Parameter | Name | Value | Effect |
|-----------|------|-------|--------|
| 1 | No Communication Light | **0** | Disable constant flash when controller silent |
| 4 | LED for Network Error | **1** | Enable warning flash if network error occurs |

**Result:** Solid LED in normal operation; flashes only on genuine network error.

**HA gotcha:** HA's slider for Parameter 1 won't go below 1. Workaround:
1. Disable the "No Communication Light" entity in HA (three-dot menu → Disable)
2. Set Parameter 1 = 0 in Z-Wave JS UI → node → Configuration tab
3. The 0 value will then stick

| Node | Device / OH name | Type | Location | Notes |
|------|-----------------|------|----------|-------|
| 3 | Siren/Alarm (ZWaveNode003NASAB01ZSirenAlarm) | NAS-AB01Z Siren | ? | switch_binary + siren mode config |
| 4 | Conservatory Garden Globe | Qubino Smart Plug | Conservatory | |
| 5 | Kitchen Window Plug | Qubino Smart Plug | Kitchen/Porch | Originally "Porch Window"; now general-purpose |
| 6 | Conservatory Xmas Tree | Qubino Smart Plug | Conservatory | Xmas group |
| 7 | Moth Trap | Qubino Smart Plug | Back Garden | Originally "Porch Window" |
| 8 | Porch Relays | Qubino Flush 2 Relays | Front Garden | Ch1: Front Soffits; Ch2: Porch Dusk-Dawn |
| 9 | Summerhouse Relays | Qubino Flush 2 Relays | Summerhouse | Ch1: Internal Lights; Ch2: Front Light |
| 10 | Lounge Dimmer | Qubino Flush Dimmer Plus | Lounge | switch_dimmer + switch_binary |
| 11 | Cons String Lights | Qubino Smart Plug | Conservatory | Bedtime group |
| 12 | Conservatory Kitchen Globe | Qubino Smart Plug | Conservatory | |
| 13 | Lounge Std Lamp | Qubino Smart Plug | Lounge | Bedtime group |
| 14 | Lounge Desk Lamp | Qubino Smart Plug | Lounge | Bedtime group |
| 15 | Cons Bird | Qubino Smart Plug | Conservatory | Originally "Xmas Icicles"; Bedtime group |
| 16 | **DEAD** — Back Garden Relays | Qubino Flush 2 Relays (ZMNHBD) | Back Garden | Z-Wave switch failed. Ch1 (Garden Floodlight) now on manual 2-way switch. Ch2 (Side Soffits) wired in series with Rear Soffits (node 20 ch1). Plan to replace/revive. |
| 17 | (Decommissioned) | Qubino Flush 2 Relays | — | Was Cons Centre/Rear Soffits; replaced by node 20 |
| 18 | Lounge Mantlepiece | Qubino Smart Plug | Lounge | Bedtime + Xmas groups |
| 20 | Rear Soffit / Cons Centre Relays | Qubino Flush 2 Relays | Back Garden/Conservatory | Ch1: Rear Soffits; Ch2: Cons Centre |

## Non-Z-Wave devices on OpenHAB (for migration reference)

| Device | Protocol | OH channel | Location |
|--------|----------|-----------|----------|
| Dining Room Dimmer (HueDimmableBulb) | Hue | hue:0100:ecb5fa2ca718:1:brightness | Dining Room |
| Sun rise/set | Astro binding | astro:sun:home / astro:sun:local | Virtual |
