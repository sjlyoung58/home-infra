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
| 16 | **DEAD** Back Garden Relays | Back Garden | pending | Z-Wave switch failed. Ch1 (Garden Floodlight) → being replaced by Aqara T2 #2 Ch1 (Zigbee, decoupled mode) — smart control restored. Ch2 (Side Soffits) → remains wired in series with Rear Soffits (node 20 ch1), no independent control planned. |
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
| Time-of-day calculator | timeofday.rules | Calculates PREDAWN/DAWN/DAYLIGHT/DAYTIME/PREDUSK/SUNSET/DUSK/NIGHTTIME/BEDTIME/SLEEPTIME from astro sunrise/sunset | pending | Not needed as a shared sensor in HA — replace with per-automation sun offset triggers (see notes below) |
| Pre-dusk lights on | timeofday.rules | PREDUSK → Lounge desk lamp, std lamp, mantlepiece ON | pending | `sun.sunset offset: -00:15:00` |
| Sunset conservatory | timeofday.rules | SUNSET → all conservatory + Cons Bird ON | pending | `sun.sunset offset: 00:00:00` |
| Dusk outdoor lights | timeofday.rules | DUSK → Rear soffits, front soffits, porch dusk-dawn, side soffits ON | pending | `sun.sunset offset: +00:15:00` |
| Nighttime moth trap | timeofday.rules | NIGHTTIME → Moth trap ON | pending | `sun.sunset offset: +00:30:00` |
| Daytime Xmas group | timeofday.rules | DAYTIME → Xmas group ON | pending | `time: "10:29:00"` — seasonal, low priority |
| Bedtime off | timeofday.rules | BEDTIME → Bedtime group + Xmas group OFF | pending | `time: "23:25:00"` |
| Dawn moth trap off | timeofday.rules | DAWN → Moth trap OFF | pending | `sun.sunrise offset: 00:00:00` |
| Lounge dimmer presets | virtualSwitch.rules | Sets dimmer level from numeric command | pending | Simple HA script or input_number |
| Siren test | siren.rules | Fires siren via Z-Wave config HTTP PUT | pending | Test/debug — low priority |
| MQTT/dimmer bridge | mqtt.rules | Sync lounge dimmer ↔ MQTT (all commented out / abandoned) | skip | Was experimental; never stable; not migrating |

### Time-of-day migration notes

The OH rule uses a shared `vTimeOfDay` string sensor as a coordination point, with all
lighting automations reacting to state changes on it. **This pattern is not needed in HA.**

HA's Sun integration provides `sun.sun` plus solar event triggers with arbitrary offsets.
Each OH triggered rule maps directly to a standalone HA automation using a sun or time
trigger — no shared sensor required.

The original staging was deliberate (not arbitrary): lights come on in three waves as
it gets progressively darker, avoiding a sudden all-at-once switch:

| OH state | Offset | What turns on | Rationale |
|----------|--------|---------------|-----------|
| PREDUSK | sunset − 15 min | Lounge desk, std lamp, mantlepiece | Indoor warmth before dark |
| SUNSET | sunset | Conservatory group + Cons Bird | Decorative/transitional |
| DUSK | sunset + 15 min | Soffits, porch lights | Outdoor infrastructure as it gets properly dark |
| NIGHTTIME | sunset + 30 min | Moth trap | Nocturnal use |

This staging must be preserved in HA — don't collapse it into a single sunset trigger.

The fixed times (10:29 AM → Xmas on, 23:25 → Bedtime off) are `platform: time` triggers.
All others are `platform: sun` with offsets.

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

## Alexa cutover plan

Current chain: **Alexa → myopenhab.org cloud → OH 3.4.1 (NAS, port 7071) → OH Bridge → OH 2.5.10 (Pi 3B)**

Target: **Alexa → HA Emulated Hue (local, free, no cloud)**

Steps when HA devices are ready:
1. Enable Emulated Hue in HA (`configuration.yaml`: `emulated_hue:`)
2. Expose all light entities and groups — ensure names match what Alexa/Julie currently uses
3. In Alexa app: Add Device → discover new devices (HA lights appear as a Hue bridge)
4. Verify all expected devices found and named correctly
5. In Alexa app: remove the openHAB skill (Smart Home → Your Skills → openHAB → Disable)
6. OH lights disappear from Alexa; HA lights already present — cutover complete
7. Stop OH 3.4.1 NAS container (no longer needed)
8. Decommission OH Bridge binding on Pi 3B (or just leave until Pi 3B is decommissioned)

**Name matching is critical** — Alexa voice commands use the device/group names.

Known voice command patterns:
- Simon (specific): "Alexa, set the **lounge dimmer** to 15" → target entity by exact name
- Julie (vague): "Alexa, set the **lounge lights** to 15" → Alexa resolves via its own device
  grouping (OH groups are NOT exposed to Alexa — resolution is Alexa-side)

Source of truth: `openhabian/oh341-config/default.items` (OH 3.4.1 Alexa bridge config)

**Definitive Alexa device name map** (name Alexa knows → HA friendly name to use):

| Alexa name | OH item label | Node | HA area | Notes |
|-----------|--------------|------|---------|-------|
| Lounge Dimmer | Lounge Dimmer | 10 | Lounge | Dimmer (brightness) |
| Standard Lamp | Lounge Std Lamp | 13 | Lounge | |
| Desk Lamp | Lounge Desk Lamp | 14 | Lounge | |
| Garden Globe | Conservatory Garden Globe | 4 | Conservatory | |
| Kitchen Globe | Conservatory Kitchen Globe | 12 | Conservatory | |
| String Lights | Cons String Lights | 11 | Conservatory | type: Switch in OH |
| Centre Lights | Conservatory Centre | 20 ch2 | Conservatory | |
| Bird | Cons Bird | 15 | Conservatory | type: Switch in OH |
| Christmas Tree | Cons Xmas Tree | 6 | Conservatory | type: Switch in OH |
| Internal Lights | Internal Lights | 9 ch1 | Summerhouse | |
| Front Light | Front Light | 9 ch2 | Summerhouse | |
| Front Soffits | Front Soffits | 8 ch1 | Front Garden | |
| Porch Lights | Porch Dusk-Dawn | 8 ch2 | Front Garden | |
| Rear Soffits | Rear Soffits | 20 ch1 | Back Garden | |
| Side Soffits | Side Soffits | 16 ch2 | Back Garden | **Node 16 dead** — currently wired in series with Rear Soffits (node 20 ch1). Not independently controllable until node 16 is replaced. |
| Garden Floodlight | Garden Floodlight | 16 ch1 | Back Garden | **Node 16 dead** — currently on manual 2-way switch. Expose as entity when node 16 replaced. **AUTO-OFF replaced by notification** (see below) |

**Not exposed to Alexa** (automation-only or not voice-controlled):
- Lounge Mantlepiece (node 18) — time-of-day automation only
- Moth Trap (node 7) — time-of-day automation only
- Kitchen Window (node 5) — not used
- Siren (node 3) — obviously
- Dining Room Dimmer (Hue) — never voice-controlled; Hue bridge can be retired

**Garden Floodlight — notification instead of auto-off (via Aqara T2 #2 Ch1):**
OH had `expire="3m,command=OFF"` — use case is letting the dog out at night and forgetting
to turn the light off. Preferred HA approach: notification after N minutes rather than
auto-off, so there's a reminder without the light cutting out unexpectedly.
Options (decide at implementation time):
- Alexa announcement: "The garden floodlight has been on for 10 minutes" (needs Alexa integration live)
- HA mobile app push notification (needs HA Companion app installed)
- Both: Alexa speaks it, phone also gets a notification

HA automation shape:
  trigger: garden_floodlight turns on
  action: wait N minutes (configurable — suggest 10 min)
  condition: garden_floodlight still on
  action: Alexa announce + push notify
  (repeat every N minutes while still on, or just once)

**Alexa group resolution:**
OH groups (GF_Conservatory etc.) have no Alexa metadata — they are NOT exposed.
"Conservatory lights", "Lounge lights" etc. are resolved by Alexa's own device grouping.
In HA, assign entities to **areas** (Lounge, Conservatory, Summerhouse, Front Garden,
Back Garden) — Alexa will auto-group by area on discovery, preserving the vague-command
behaviour Julie relies on.

## Blockers

- Z-Wave JS WebSocket server must be enabled on Pi 5 before any Z-Wave devices appear in HA (Phase 5 prerequisite)
- Confirm device/group names Julie uses with Alexa before naming HA entities
