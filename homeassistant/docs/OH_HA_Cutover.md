# OH → HA Cutover Plan

Last updated: 2026-07-01
Status: Gen5+ confirmed as permanent Z-Wave controller. OH still live. HA not yet
controlling real devices. This document is the ordered task list to complete the cutover.

**Preference: YAML config over UI wherever practical.**

---

## Context

- **Z-Wave controller**: Aeotec Gen5+ on Pi5 (192.168.1.222), directly connected (no hub)
  Home ID: `0xde2f557d`. Z-Wave JS UI: http://192.168.1.222:8091, ws port 3001.
- **OH still live**: Pi3B (openhabian@192.168.1.78) running OH 2.5.10 with Gen5 (500-series),
  Alexa chain: Alexa → myopenhab.org → OH 3.4.1 (NAS) → OH Bridge → OH 2.5.10 (Pi3B)
- **HA**: http://192.168.1.137:8123, Docker/Core on NAS (no add-on store)
- **Node reference**: `zwave/node-map.md`
- **Alexa name ground truth**: `openhabian/oh341-config/default.items`

---

## Phase 1 — Fix HA Z-Wave JS integration

HA flagged a controller change (Gen5 → Gen5+) and requires reconfiguration.

- [ ] Settings → Devices & Services → Z-Wave JS → **Reconfigure** (or delete and re-add)
      URL: `ws://192.168.1.222:3001`
- [ ] Verify all nodes appear: expected nodes 1, 3–16, 20 (node 16 is dead hardware — ok)
- [ ] Confirm node count in HA matches Z-Wave JS UI

---

## Phase 2 — Configure GWPN1 sockets (LED behaviour)

All 10 GreenWave GWPN1 smart plugs need parameter 4 set to 0 to prevent constant LED
flashing (Z-Wave JS does not poll devices — the default "network error" flash fires
continuously without polling). Do via Z-Wave JS UI → node → Configuration tab.

See `zwave/node-map.md` for the full parameter detail and HA slider workaround.

| Node | Device name | Done |
|------|-------------|------|
| 4 | Conservatory Garden Globe | [ ] |
| 5 | Kitchen Window Plug | [ ] |
| 6 | Conservatory Xmas Tree | [ ] |
| 7 | Moth Trap | [ ] |
| 11 | Cons String Lights | [ ] |
| 12 | Conservatory Kitchen Globe | [ ] |
| 13 | Lounge Std Lamp | [ ] |
| 14 | Lounge Desk Lamp | [ ] |
| 15 | Cons Bird | [ ] |
| 18 | Lounge Mantlepiece | [ ] |

Method: Z-Wave JS UI → node → Configuration tab → `LED for Network Error` → set to `0`.
HA's slider won't go below 1 for param 1 — leave param 1 at 1 (minimum), only param 4 matters.

---

## Phase 3 — Entity naming

HA Z-Wave entities will have generated names. Rename to match the definitive Alexa names
from OH 3.4.1. This is the ground truth for what Alexa (and Julie) currently knows.

For the Lounge Dimmer (node 10): the main entity is the dimmer switch. The binary
switch entity for it can be disabled.

**Entity renaming approach (YAML via customize.yaml):**

Add to `configuration.yaml`:
```yaml
homeassistant:
  customize: !include customize.yaml
```

Then in `customize.yaml` (match entity IDs to actual Z-Wave JS generated IDs — check HA
device page for each node to get the exact entity_id):

```yaml
# Lounge
light.node_10_lounge_dimmer:          # verify entity_id
  friendly_name: "Lounge Dimmer"
switch.node_13_lounge_std_lamp:       # verify entity_id
  friendly_name: "Standard Lamp"
switch.node_14_lounge_desk_lamp:      # verify entity_id
  friendly_name: "Desk Lamp"
switch.node_18_lounge_mantlepiece:    # verify entity_id
  friendly_name: "Lounge Mantlepiece"

# Conservatory
switch.node_4_cons_garden_globe:
  friendly_name: "Garden Globe"
switch.node_12_cons_kitchen_globe:
  friendly_name: "Kitchen Globe"
switch.node_11_cons_string_lights:
  friendly_name: "String Lights"
switch.node_15_cons_bird:
  friendly_name: "Bird"
switch.node_6_cons_xmas_tree:
  friendly_name: "Christmas Tree"
switch.node_20_ch2_cons_centre:       # channel 2 of node 20
  friendly_name: "Centre Lights"

# Outdoor — Front
switch.node_8_ch1_front_soffits:      # channel 1 of node 8
  friendly_name: "Front Soffits"
switch.node_8_ch2_porch_lights:       # channel 2 of node 8
  friendly_name: "Porch Lights"

# Outdoor — Back
switch.node_20_ch1_rear_soffits:      # channel 1 of node 20
  friendly_name: "Rear Soffits"
# node 16 dead — Side Soffits currently wired in series with Rear Soffits, no HA entity needed

# Summerhouse
switch.node_9_ch1_internal_lights:    # channel 1 of node 9
  friendly_name: "Internal Lights"
switch.node_9_ch2_front_light:        # channel 2 of node 9
  friendly_name: "Front Light"

# Not exposed to Alexa (automation-only)
switch.node_7_moth_trap:
  friendly_name: "Moth Trap"
switch.node_5_kitchen_window:
  friendly_name: "Kitchen Window Plug"
```

**Note:** Entity IDs above are illustrative. Z-Wave JS generates them differently — check
actual entity IDs in HA under each device before writing customize.yaml.

---

## Phase 4 — HA areas

Create areas in HA UI (Settings → Areas) matching the groups Alexa uses for group control.
Areas drive Emulated Hue group discovery.

| Area name | Entities to assign |
|-----------|-------------------|
| Lounge | Lounge Dimmer, Standard Lamp, Desk Lamp, Lounge Mantlepiece |
| Conservatory | Garden Globe, Kitchen Globe, String Lights, Bird, Christmas Tree, Centre Lights |
| Summerhouse | Internal Lights, Front Light |
| Front Garden | Front Soffits, Porch Lights |
| Back Garden | Rear Soffits |
| Kitchen | Kitchen Window Plug |

Areas created via HA UI — not configurable in YAML for HA Core.

---

## Phase 5 — Light groups (YAML)

Create explicit HA light/switch groups so Julie can say "Alexa, turn on the lounge lights"
and get the whole room. Groups are exposed to Alexa via Emulated Hue alongside individual
devices.

Add to `configuration.yaml`:
```yaml
light:
  - platform: group
    name: "Lounge Lights"
    entities:
      - light.lounge_dimmer          # node 10 (only dimmable light in lounge)
      - switch.standard_lamp         # node 13
      - switch.desk_lamp             # node 14
      - switch.lounge_mantlepiece    # node 18

switch:
  - platform: group
    name: "Conservatory Lights"
    entities:
      - switch.garden_globe          # node 4
      - switch.kitchen_globe         # node 12
      - switch.string_lights         # node 11
      - switch.bird                  # node 15
      - switch.christmas_tree        # node 6
      - switch.centre_lights         # node 20 ch2

  - platform: group
    name: "Summerhouse Lights"
    entities:
      - switch.internal_lights       # node 9 ch1
      - switch.front_light           # node 9 ch2

  - platform: group
    name: "Bedtime Lights"
    entities:
      - light.lounge_dimmer
      - switch.standard_lamp
      - switch.desk_lamp
      - switch.lounge_mantlepiece
      - switch.garden_globe
      - switch.kitchen_globe
      - switch.string_lights
      - switch.bird
      - switch.centre_lights
      - switch.front_soffits
      - switch.porch_lights
      - switch.rear_soffits
```

**Note:** Verify entity IDs before committing — these are friendly_name-derived slugs.

---

## Phase 6 — Time-of-day automations (YAML)

Replicates `timeofday.rules` from OH. Each OH state change becomes a standalone HA
automation with a sun offset trigger. No shared state sensor needed.

See `openhabian/migration-status.md` for full analysis. The staging was deliberate —
preserve it.

Add to `automations.yaml`:

```yaml
# --- PREDUSK (sunset - 15 min): indoor warmth first ---
- alias: "Predusk - Lounge lamps on"
  trigger:
    - platform: sun
      event: sunset
      offset: "-00:15:00"
  action:
    - service: switch.turn_on
      target:
        entity_id:
          - switch.standard_lamp
          - switch.desk_lamp
          - switch.lounge_mantlepiece

# --- SUNSET: conservatory & decorative ---
- alias: "Sunset - Conservatory on"
  trigger:
    - platform: sun
      event: sunset
      offset: "00:00:00"
  action:
    - service: switch.turn_on
      target:
        entity_id:
          - switch.garden_globe
          - switch.kitchen_globe
          - switch.string_lights
          - switch.bird
          - switch.centre_lights

# --- DUSK (sunset + 15 min): outdoor infrastructure ---
- alias: "Dusk - Outdoor lights on"
  trigger:
    - platform: sun
      event: sunset
      offset: "00:15:00"
  action:
    - service: switch.turn_on
      target:
        entity_id:
          - switch.rear_soffits
          - switch.front_soffits
          - switch.porch_lights

# --- NIGHTTIME (sunset + 30 min): moth trap ---
- alias: "Nighttime - Moth trap on"
  trigger:
    - platform: sun
      event: sunset
      offset: "00:30:00"
  action:
    - service: switch.turn_on
      target:
        entity_id: switch.moth_trap

# --- DAWN: moth trap off ---
- alias: "Dawn - Moth trap off"
  trigger:
    - platform: sun
      event: sunrise
      offset: "00:00:00"
  action:
    - service: switch.turn_off
      target:
        entity_id: switch.moth_trap

# --- DAYTIME (fixed 10:29): Xmas group on ---
- alias: "Daytime - Christmas Tree on"
  trigger:
    - platform: time
      at: "10:29:00"
  action:
    - service: switch.turn_on
      target:
        entity_id: switch.christmas_tree

# --- BEDTIME (fixed 23:25): all off ---
- alias: "Bedtime - All off"
  trigger:
    - platform: time
      at: "23:25:00"
  action:
    - service: homeassistant.turn_off
      target:
        entity_id:
          - switch.standard_lamp
          - switch.desk_lamp
          - switch.lounge_mantlepiece
          - switch.garden_globe
          - switch.kitchen_globe
          - switch.string_lights
          - switch.bird
          - switch.centre_lights
          - switch.front_soffits
          - switch.porch_lights
          - switch.rear_soffits
          - switch.christmas_tree

# --- GARDEN FLOODLIGHT: 10-minute reminder ---
# (Node 16 dead — implement when hardware replaced)
# - alias: "Garden Floodlight - 10 min reminder"
#   trigger:
#     - platform: state
#       entity_id: switch.garden_floodlight
#       to: "on"
#       for: "00:10:00"
#   action:
#     - service: notify.alexa_media
#       data:
#         message: "The garden floodlight has been on for 10 minutes"
#         data:
#           type: announce
```

**Note:** Entity IDs in automations must match final entity names from Phase 3. Update
after customization is confirmed.

---

## Phase 7 — Emulated Hue (Alexa integration)

Replaces the OH Alexa skill chain. HA pretends to be a Hue bridge; Alexa discovers
devices via LAN. No cloud required, no Nabu Casa needed.

**Prerequisite:** Hue bridge retired (192.168.1.69) — power it off before enabling
Emulated Hue to avoid conflicts.

Add to `configuration.yaml`:
```yaml
emulated_hue:
  host_ip: 192.168.1.137
  listen_port: 80
  off_maps_to_on_domains:
    - script
    - scene
  expose_by_default: false
  entities:
    # Individual devices — match Alexa names from OH
    light.lounge_dimmer:
      name: "Lounge Dimmer"
      hidden: false
    switch.standard_lamp:
      name: "Standard Lamp"
      hidden: false
    switch.desk_lamp:
      name: "Desk Lamp"
      hidden: false
    switch.garden_globe:
      name: "Garden Globe"
      hidden: false
    switch.kitchen_globe:
      name: "Kitchen Globe"
      hidden: false
    switch.string_lights:
      name: "String Lights"
      hidden: false
    switch.centre_lights:
      name: "Centre Lights"
      hidden: false
    switch.bird:
      name: "Bird"
      hidden: false
    switch.christmas_tree:
      name: "Christmas Tree"
      hidden: false
    switch.internal_lights:
      name: "Internal Lights"
      hidden: false
    switch.front_light:
      name: "Front Light"
      hidden: false
    switch.front_soffits:
      name: "Front Soffits"
      hidden: false
    switch.porch_lights:
      name: "Porch Lights"
      hidden: false
    switch.rear_soffits:
      name: "Rear Soffits"
      hidden: false
    # Groups — enables "Alexa, turn on the lounge lights"
    light.lounge_lights:
      name: "Lounge Lights"
      hidden: false
    switch.conservatory_lights:
      name: "Conservatory Lights"
      hidden: false
    switch.summerhouse_lights:
      name: "Summerhouse Lights"
      hidden: false
```

**Note:** `expose_by_default: false` means only explicitly listed entities are exposed.
This prevents Z-Wave sensors, binary sensors, etc. from cluttering Alexa's device list.

---

## Phase 8 — Alexa cutover

**Do this only after Phase 7 is configured and tested in HA.**

- [ ] Power off Hue bridge (192.168.1.69)
- [ ] Restart HA to load Emulated Hue config
- [ ] In Alexa app: Devices → Add Device → discover (HA devices should appear)
- [ ] Verify each device appears by name, test a few voice commands
- [ ] In Alexa app: More → Skills & Games → openHAB → Disable skill
- [ ] OH devices disappear from Alexa; HA devices remain
- [ ] Test Julie's commands: "Alexa, set the lounge lights to 15", etc.
- [ ] Test group commands: "Alexa, turn on the conservatory lights"
- [ ] Stop OH 3.4.1 Alexa bridge container on NAS (Portainer → stack openhab3-4-1 → stop)

---

## Phase 9 — Parallel verification

Run OH and HA simultaneously for a period before decommissioning OH.

- [ ] All time-of-day automations fire correctly (verify at dusk on first evening)
- [ ] Alexa voice control working for all named devices
- [ ] Group control working ("lounge lights", "conservatory lights", "summerhouse lights")
- [ ] Lounge dimmer brightness control working ("Alexa, set the lounge dimmer to 40")
- [ ] Power monitoring visible in HA for GreenWave sockets
- [ ] Bedtime automation fires at 23:25 and turns everything off
- [ ] Monitor for at least 3–5 days — catch any sunrise/sunset edge cases

---

## Phase 10 — Decommission OH

Only after Phase 9 confidence period.

- [ ] Stop OH 3.4.1 NAS container permanently (already stopped in Phase 8)
- [ ] Stop OH 2.5.10 on Pi3B: `sudo systemctl stop openhab2`
- [ ] Unplug Gen5 from Pi3B (it's now spare — keep as backup)
- [ ] Gen5+ remains on Pi5 as permanent Z-Wave controller
- [ ] Pi3B can be decommissioned or repurposed

---

## Deferred items (not part of this cutover)

These are handled in separate phases after core cutover is stable:

| Item | Where documented |
|------|-----------------|
| Hue dining room dimmer (via Zigbee2MQTT) | Zigbee setup — parked |
| Reolink cameras | `cameras/device-inventory.md`, Phase 3a |
| Ring doorbell | `cameras/device-inventory.md`, Phase 3a |
| Audi Q5 integration | `homeassistant/docs/AUDI.md` |
| Alexa announcements (for automations) | `homeassistant/docs/AUDI.md` Component 3 |
| MyEnergi / Zappi | `HOUSE.md` Solar / Energy section |
| Hive heating | `HOUSE.md` Hive section |
| Cloudflare Tunnel (remote access) | `CLAUDE.md` Remote access section |
| iPhone Companion app / geofencing | `CLAUDE.md` iPhones section |
| Garden Floodlight reminder automation | Blocked pending node 16 hardware replacement |

---

## Quick reference — Alexa name map

From `openhabian/oh341-config/default.items` — **these names must be preserved exactly.**

| Alexa name | Z-Wave node | Location |
|-----------|-------------|----------|
| Lounge Dimmer | 10 | Lounge |
| Standard Lamp | 13 | Lounge |
| Desk Lamp | 14 | Lounge |
| Garden Globe | 4 | Conservatory |
| Kitchen Globe | 12 | Conservatory |
| String Lights | 11 | Conservatory |
| Centre Lights | 20 ch2 | Conservatory |
| Bird | 15 | Conservatory |
| Christmas Tree | 6 | Conservatory |
| Internal Lights | 9 ch1 | Summerhouse |
| Front Light | 9 ch2 | Summerhouse |
| Front Soffits | 8 ch1 | Front Garden |
| Porch Lights | 8 ch2 | Front Garden |
| Rear Soffits | 20 ch1 | Back Garden |
| Side Soffits | 16 ch2 | Back Garden (node dead — wired with Rear Soffits) |
| Garden Floodlight | 16 ch1 | Back Garden (node dead — manual switch) |
