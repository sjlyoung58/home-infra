# House Reference

## Overview
- Type: Bungalow
- Address: 58 Liverpool Road, Walmer, Deal CT14 7LG
- Front faces: **East** (road side)

---

## Floor Plan

```
                               SOUTH (property boundary)
                                        
                 ┌───────────┬──────────────┬──────────────────┐
   C(Front Right)│           │  WIW         |                  |C(Rear)
                 |  Main Bed |  [NAS]┌──────┤    Lounge        |  
                 │           └───────┤En-   |    5×5(m)        │
                 │               G  E│Suite │                  │              ┌────────────┐
EAST             ├─────────────┌─────┴──────┤               R2 │              |            |
                 │             │R3          ├──────────────────┤    WEST      |  Summer-   |
           Zappi │Middle Bed   │   Dining   |                  │              |   House    |
                 │ [Comms]   R1│                Conservatory   │              |            |
                 ├─────────────┤                   (5×5m)      │              └────────────┘
           ┌─────├             |                               |
    Virgin │Porch│   Hall      |            |                  |
           ├─────┴────┬────────┴─┬──────────┴──────────────────┘
           │          │          │          |                 C(Rear Kitchen)
   (Front  │Study/Bed3│  Bath    │ Kitchen  |
     Left)C│          │          │          |
           └──────────┴──────────┴──────────┘
            C(side Alley)
                 NORTH  (side alley along north boundary)

  R1 = RT-AC86u   R2 = RT-AX56U (Mesh)  R3 = RP-AX56 (Mesh)
  G = gas meter   E = elec meter  WIW = Walk-in Wardrobe
  [NAS] = Synology NAS in walk-in wardrobe (south extension)
  [Comms] = comms cupboard in middle bedroom
  Walk-in + En-Suite run along south boundary (later extension)
  Zappi EV charger on outside east (front) wall
  C - Reolink Camera
  Virgin Hub 5x in Porch
```

> Derived from hand-drawn sketch — refine as needed.

---

## Roofs

### Original (pitched/tiled)
- South pitch (narrow end): 3 solar panels
- West pitch: 11 solar panels

### Extensions (flat)
- **Front**: Study/Bed 3 area (front side extension)
- **Rear**: Lounge + Conservatory (each 5m × 5m, contiguous along rear from south boundary)

---

## Solar / Energy

| Device | Location | Notes |
|--------|----------|-------|
| 14× solar panels | Roof (3 south, 11 west) | West-heavy — afternoon generation profile |
| MyEnergi Zappi | Front wall / driveway | EV charger, solar-aware (excess diversion) |
| MyEnergi Harvi | Electric meter cupboard | Clamps for solar and mains monitoring |
| MyEnergi Hub | Comms cupboard | Links Zappi + Harvi to cloud/HA |
| Electric meter | Main bedroom (meter cpbd) | Gen 1 smart meter, not functioning smart |

### zappimon (existing Node.js app)
- Location: `../zappimon` (sibling project, not in this repo)
- Polls MyEnergi API every 30 seconds via `myenergi-api` npm package
- Logs: voltage (calibrated against Fluke meter), solar kW, grid kW, EV charge kW,
  calculated house consumption (solar + grid − charge), min/max voltage session tracking
- Description says "persist data for historical analysis" — persistence not yet built;
  currently logs to console/zappi.log only
- Credentials hardcoded in `src/main.mjs` — fine for private local use, don't expose

**Origin / backstory:** Built ~2 years ago to diagnose intermittent Zappi faults. Voltage
fluctuation was the suspected cause (hence the Fluke calibration factor). MyEnergi
requested an electrician measure voltage inside the unit; the real fault was a loose ribbon
cable between the display PCB and main PCB — almost certainly loose since original
installation and gradually working free over 2-3 years. Reseating the cable fixed it.
Voltage was a red herring. App retained as passive monitoring but no longer diagnostic.

### HA integration (planned)
- `myenergi` integration (built into HA core) — reads Zappi and Harvi as HA entities
- Would give: solar generation sensor, grid import/export sensor, Zappi charge mode,
  surplus solar available → useful for automations (e.g. boost hot water when exporting)
- zappimon could be retired or extended to push to HA via webhook/MQTT once HA is live

### Hive (heating)
- Hive Hub in comms cupboard controls central heating/hot water
- Currently controlled via Alexa skill only ("Alexa, ask Hive to boost the hot water")
- HA integration: `hive` (available via HACS) — add later, not a migration priority
- No change needed to Alexa Hive skill — it's independent of the OH→HA migration

---

## Infrastructure / Comms

### Broadband
- FTTP (Nexfibre/Virgin Media) cable buried in lawn, enters building in Porch
- Hub 5X in Porch, wired RJ45 to Comms Cupboard, running in DMZ mode to main ASUS router
- Old FTTC cable available (no longer in use) — wired into comms cupboard

### Comms Cupboard (in Middle Bedroom)
- 10-port unmanaged switch (main LAN distribution)
- 4-port PoE switch (4 × Reolink RLC-811A — full)
- Single PoE injector (RLC-1224A "Front Right" — repurposed from replaced Hikvision)
  Plan: move to Mercusys PoE switch once location is decided (loft ruled out — see below)
- Many ethernet cables from loft entering through ceiling to RJ45 sockets in rooms
- Hive Hub (heating control)
- MyEnergi Hub
- Philips Hue Bridge (to be retired — see [migration status](openhabian/migration-status.md))
- RT-AC86U — primary AiMesh router (thick west wall requires 2 satellite nodes at rear: RT-AX56U + RP-AX56)

### NAS location
- Synology NAS in walk-in wardrobe off main bedroom, with APC UPS backup

---

## Rooms

### Porch / Entrance Hall
**Description:** FTTP entry point. Hub 5X located here.

#### Lighting
- Porch: 2 × GU10 downlights ("Porch" — Z-Wave node 8 ch2, Flush 2 Relays)

#### Smart devices
- Virgin Media Hub 5X (broadband, DMZ mode)

---

### Study / Bedroom 3
**Description:** Front-facing room, flat-roof extension.

#### Lighting
- TBC

#### Smart devices
- TBC

---

### Middle Bedroom
**Description:** Adjacent to main bedroom. Houses the comms cupboard.

#### Lighting
- TBC

#### Smart devices
- See Comms Cupboard section above

---

### Main Bedroom
**Description:** Full depth of original house, south end. Formerly a single garage,
converted and extended ~1.3m to the south property boundary. Door to dining room.
Contains walk-in wardrobe (NAS location), ensuite bathroom/shower.

#### Sub-rooms
- Walk-in wardrobe: Synology NAS + APC UPS
- Ensuite: bath and shower
- Electric meter cupboard: consumer unit, Gen 1 smart meter, MyEnergi Harvi
- Gas meter cupboard

#### Lighting
- TBC

#### Smart devices
- MyEnergi Harvi (clamps monitoring solar and mains)

---

### Kitchen
**Description:** North-west area, windows face rear garden (west).

#### Lighting
- Conservatory Kitchen Globe: Z-Wave node 12 (Qubino smart plug, "Kitchen Globe" in Alexa)
  — note: despite the name, this plug is physically in/near the kitchen
- **Sylstar 24W Smart LED Ceiling Light**: Tuya/Smart Life compatible, not yet integrated
  Currently on a simple on/off switch. **Confirmed WiFi (2.4GHz)**. Now paired to Smart Life.
  Pairing: 3× on/off cycles → light flashes → pair in Smart Life app. Phone must be on
  2.4GHz WiFi during pairing (device does not support 5GHz).
  - HA integration: Tuya local integration (requires local device key extraction) or
    cloud-based Tuya integration as simpler first step
  - Either way: needs always-powered (smart light must not lose power)
  - Switch options: T2 channel (decoupled mode) OR Sonoff ZBMINI-L2 behind existing switch
    (ZBMINI-L2 simpler if no other relay channel needed in kitchen; no neutral required)

---

### Dining Room
**Description:** Adjacent to main bedroom (door between them). Contains Hue bulb.

#### Lighting
- Dining room dimmer: Philips Hue dimmable bulb (ecb5fa2ca718, 192.168.1.69)
  — to be migrated to Zigbee2MQTT direct when Hue bridge is retired

#### Smart devices
- TBC

---

### Lounge
**Description:** Rear extension, flat roof, 5m × 5m. Contiguous with conservatory.
No loft access (solid foam insulation).

#### Lighting (Z-Wave, all via OH group `GF_Lounge`)
| Node | Alexa name | Device | Notes |
|------|-----------|--------|-------|
| 10 | Lounge Dimmer | Qubino Flush Dimmer Plus | 5-bulb LED candle ceiling light. To be replaced with Candeo RD1 Zigbee rotary dimmer |
| 13 | Standard Lamp | Qubino smart plug | Floor lamp |
| 14 | Desk Lamp | Qubino smart plug | Desk lamp |
| 18 | Lounge Mantlepiece | Qubino smart plug | Automation-only (not Alexa) |

#### Smart devices
- TBC (presence sensor planned — Sonoff SNZB-06P or similar)

#### Planned automations
- Pre-dusk (sunset − 15 min): desk lamp, standard lamp, mantlepiece ON
- Bedtime (23:25): Bedtime group OFF

---

### Conservatory
**Description:** Rear extension, flat roof, 5m × 5m, adjoining lounge. Glass walls.
Features: Ventec 100 window controller (to be replaced with Shelly 2PM Gen 4).
**Planned MR4 location** — glass walls ideal for Zigbee/Thread signal in all directions.

#### Lighting — current (Z-Wave, via OH group `GF_Conservatory`)
| Node | Alexa name | Device | Notes |
|------|-----------|--------|-------|
| 4 | Garden Globe | Qubino smart plug | |
| 11 | String Lights | Qubino smart plug | |
| 12 | Kitchen Globe | Qubino smart plug | Near kitchen side |
| 15 | Bird | Qubino smart plug | Decorative |
| 20 ch2 | Centre Lights | Qubino Flush 2 Relays ch2 | Wall switch input |

#### Lighting — planned (8 × IKEA GU10 Thread bulbs)
8 GU10 ceiling spots to be replaced with **IKEA multicolour GU10 Thread bulbs** (Matter-over-Thread).
Will pair via HA Matter integration using MR4 as Thread Border Router.
Powered Thread bulbs act as Thread mesh routers — extend coverage from conservatory outward.

**Smart bulb model — important:**
With Thread GU10s, "off" = brightness 0% (bulb still powered, still on Thread network).
Relays must stay permanently closed. Cutting power = bulbs drop off Thread network (bad).
Z-Wave toggle behaviour (relay cuts power on switch press) is incompatible with this.

**Current wiring** (3-gang wall switch plate):
- Switch 1: 2 GU10 bulbs (physical only)
- Switch 2: 2 GU10 bulbs (physical only)
- Switch 3: 4 GU10 bulbs via existing Qubino Flush 2 Relays (Z-Wave, node TBC)

**Planned wiring — 2 × Aqara T2 Dual Relay (all channels in decoupled mode):**

| T2 unit | Channel | Controls | Switch input |
|---------|---------|----------|-------------|
| T2 #1 | Ch1 | 2 × Thread GU10 | 3-gang switch 1 |
| T2 #1 | Ch2 | 2 × Thread GU10 | 3-gang switch 2 |
| T2 #2 | Ch1 | Garden Floodlight | 2 × 2-way switches (T2 S1 sees the 2-way output — transparent, either switch works) |
| T2 #2 | Ch2 | 4 × Thread GU10 | 3-gang switch 3 (replaces Qubino Z-Wave relay) |

All 3 switches in the group use the same T2 type → consistent decoupled mode behaviour
across the whole plate. Garden floodlight regains smart control (lost when node 16 died).

**Physical fallback (design decision):**
In decoupled mode, switches work whenever HA is up. Acceptable because Alexa has the
same HA dependency. Emergency fallback if HA is down: cut circuit at fuse — bulbs return
to default-on at full brightness when power restored. HA on NAS with UPS = very reliable.

#### Smart devices (existing)
- Ventec 100 window controller (230V, 3-wire actuator — brown=open, black=close, blue=N)
  Rain sensor on this unit failed ~2 years ago

#### Smart devices (planned — [new kit inventory](hardware/new-kit-inventory.md))
- SLZB-MR4: Zigbee + Thread coordinator (PoE, placed in conservatory)
- 2 × Aqara Dual Relay T2 (DCM-K01): see wiring table above (all in decoupled mode)
- Shelly Plus 2PM Gen 4: replacing Ventec 100 for window control (cover mode)
- Tuya ZY-M100 mmWave presence sensor: mounted angled in wall box (55mm deep)
- HLK-PM01 5V PSU: powers presence sensor from mains in wall box
- Tiardey "Sunflower" Zigbee rain+light sensor: on conservatory roof, south/SW facing
- Varilight 4-module plate: 1 × centre-off switch (open/close) + 3 blanks

#### Planned automations
- Open if: presence detected AND temp >22°C (daytime)
- Close if: rain detected (immediate) OR after sunset OR presence clear 15 min

---

## Outdoor

### Front Garden / Driveway
**Description:** 10m driveway from road. Block paved, parking for 3 cars, lawn/flowerbeds.

#### Lighting (Z-Wave, OH group `OU_FrontGarden`)
| Node | Alexa name | Device | Notes |
|------|-----------|--------|-------|
| 8 ch1 | Front Soffits | Qubino Flush 2 Relays | 7 × GU10 under front soffits |
| 8 ch2 | Porch Lights | Qubino Flush 2 Relays | Dusk-dawn porch light |

#### Other devices
- MyEnergi Zappi EV charger
- Reolink RLC-1224A "Front Right" (192.168.1.19) — under right-hand soffits
- Reolink RLC-811A "Front Left" (192.168.1.84) — under left-hand soffits

#### Planned automations
- Arrival home after dark: turn on front soffits + porch lights (geofence trigger)

---

### Back Garden
**Description:** West-facing rear garden.

#### Lighting (Z-Wave, OH group `OU_BackGarden`)
| Node | Alexa name | Device | Notes |
|------|-----------|--------|-------|
| 20 ch1 | Rear Soffits | Qubino Flush 2 Relays | Also controls Side Soffits (wired in series — node 16 dead) |
| 16 ch2 | Side Soffits | ~~Qubino Flush 2 Relays~~ | **Node 16 dead** — currently wired in series with Rear Soffits |
| 16 ch1 | Garden Floodlight | ~~Qubino Flush 2 Relays~~ | **Node 16 dead** — on manual switch pending replacement |

#### Cameras
- Reolink RLC-811A "Side Alley" (192.168.1.117)
- Reolink RLC-811A "Rear Kitchen" (192.168.1.118)
- Reolink RLC-811A "Rear" (192.168.1.159)

#### Planned automations
- Dusk (sunset + 15 min): rear soffits ON
- Garden floodlight: notification after 10 min on ("The garden floodlight has been on for 10 minutes")

---

### Summerhouse
**Description:** Outbuilding in garden.

#### Lighting (Z-Wave, OH group `OU_Summerhouse`)
| Node | Alexa name | Device | Notes |
|------|-----------|--------|-------|
| 9 ch1 | Internal Lights | Qubino Flush 2 Relays | |
| 9 ch2 | Front Light | Qubino Flush 2 Relays | External light on summerhouse |

---

## Seasonal / Occasional Groups

| OH Group | Alexa name | Members | Notes |
|----------|-----------|---------|-------|
| Bedtime | — | Lounge lamps, conservatory lights, soffits | Turned OFF at 23:25 |
| Xmas | Christmas Tree | Cons Xmas Tree + others | Seasonal use |
