# OpenHabian → Home Assistant Migration Status

OpenHAB version: 2.5.10
OpenHAB URL: http://192.168.1.78:8080
Pi 3B SSH: openhabian@192.168.1.78

## Status key
- `migrated` — live in HA, tested, OpenHabian version retired
- `pending` — not yet started
- `blocked` — waiting on something (note reason)
- `skip` — not migrating (note reason)

---

## Z-Wave Devices

Z-Wave controller has already moved to Pi 5 (Z-Wave JS UI).
Devices will appear in HA once Z-Wave JS integration is configured (Phase 5).
WebSocket server must be enabled first — see zwave/node-map.md.

| Node | Name | Location | Status | Notes |
|------|------|----------|--------|-------|
| 3 | Siren/Alarm | ? | pending | NAS-AB01Z; test rule in OH fires it via HTTP PUT config |
| 4 | Conservatory Garden Globe | Conservatory | pending | |
| 5 | Kitchen Window Plug | Kitchen/Porch | pending | |
| 6 | Conservatory Xmas Tree | Conservatory | pending | |
| 7 | Moth Trap | Back Garden | pending | |
| 8 | Porch Relays (Front Soffits, Porch Dusk-Dawn) | Front Garden | pending | 2-channel relay |
| 9 | Summerhouse Relays (Internal Lights, Front Light) | Summerhouse | pending | 2-channel relay |
| 10 | Lounge Dimmer | Lounge | pending | Dimmer + binary switch |
| 11 | Cons String Lights | Conservatory | pending | |
| 12 | Conservatory Kitchen Globe | Conservatory | pending | |
| 13 | Lounge Std Lamp | Lounge | pending | |
| 14 | Lounge Desk Lamp | Lounge | pending | |
| 15 | Cons Bird | Conservatory | pending | |
| 16 | Back Garden Relays (Garden Floodlight, Side Soffits) | Back Garden | pending | 2-channel relay |
| 17 | (Decommissioned) | — | skip | Replaced by node 20 |
| 18 | Lounge Mantlepiece | Lounge | pending | |
| 20 | Rear Soffits / Cons Centre | Back Garden/Conservatory | pending | 2-channel relay |

---

## Non-Z-Wave Devices

| Device | Protocol | OH integration | Status | Notes |
|--------|----------|---------------|--------|-------|
| Dining Room Dimmer | Philips Hue | Hue binding (ecb5fa2ca718) | pending | HA has built-in Hue integration |
| Sun rise/set | Astro binding | astro:sun:home | pending | HA has built-in Sun integration |

---

## Automations / Rules

| Rule | File | Description | Status | Notes |
|------|------|-------------|--------|-------|
| Time-of-day calculator | timeofday.rules | Calculates PREDAWN/DAWN/DAYLIGHT/DAYTIME/PREDUSK/SUNSET/DUSK/NIGHTTIME/BEDTIME/SLEEPTIME from astro sunrise/sunset | pending | HA sun integration + time-based automations can replace this |
| Pre-dusk lights on | timeofday.rules | PREDUSK → Lounge desk lamp, std lamp, mantlepiece ON | pending | |
| Sunset conservatory | timeofday.rules | SUNSET → all conservatory + Cons Bird ON | pending | |
| Dusk outdoor lights | timeofday.rules | DUSK → Rear soffits, front soffits, porch dusk-dawn, side soffits ON | pending | |
| Nighttime moth trap | timeofday.rules | NIGHTTIME → Moth trap ON | pending | |
| Daytime Xmas group | timeofday.rules | DAYTIME → Xmas group ON | pending | Seasonal — low priority |
| Bedtime off | timeofday.rules | BEDTIME → Bedtime group + Xmas group OFF | pending | |
| Dawn moth trap off | timeofday.rules | DAWN → Moth trap OFF | pending | |
| Lounge dimmer presets | virtualSwitch.rules | Sets dimmer level from numeric command | pending | Simple helper |
| Siren test | siren.rules | Fires siren via Z-Wave config HTTP PUT | pending | Test/debug rule; low priority |
| MQTT/dimmer bridge | mqtt.rules | Sync lounge dimmer ↔ MQTT (all commented out / abandoned) | skip | Was experimental; never stable; not migrating |

---

## MQTT Bridge approach — decision record

An MQTT bridge between OH and HA was attempted as a transition strategy (HA consuming OH
device state via MQTT before full migration). Abandoned — proved difficult and flaky in
practice. Root causes: OH 2.5.x MQTT binding reliability issues, retained-message state
drift, and the feedback-loop problem on the lounge dimmer (required a mutex workaround that
still wasn't stable).

Not worth revisiting. Z-Wave devices migrate cleanly via Z-Wave JS → HA direct integration,
with no OH involvement. Hue migrates via native HA Hue integration. No MQTT bridge is
needed at any stage of the migration.

---

## Cleanup tasks

| Task | Status | Notes |
|------|--------|-------|
| Remove MQTT integration from HA (bridge remnant) | pending | The MQTT integration was configured solely for the OH→HA bridge. Safe to remove once confirmed nothing else in HA currently depends on it. Do NOT remove if/when Zigbee2MQTT is migrated — that will need MQTT. |

## Blockers

- Z-Wave JS WebSocket server must be enabled on Pi 5 before any Z-Wave devices appear in HA (Phase 5 prerequisite)
- Hue bridge IP/credentials needed for HA Hue integration setup
