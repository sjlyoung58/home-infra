# New Hardware Inventory
# Sourced from March 2026 planning session (gemini_chat.txt)

Five distinct projects emerged. Items marked **[bought]** were confirmed ordered in the
chat; **[planned]** means decided on but purchase not confirmed; **[discussed/rejected]**
means considered but ruled out.

---

## Project 1 — Zigbee/Thread Coordinator Infrastructure

**Goal:** Replace the Pi 5's Z-Stick 10 Pro Zigbee path with a purpose-built, loft-mounted
Zigbee + Thread coordinator with better signal coverage across the bungalow.

### Hardware

| Item | Status | Notes |
|------|--------|-------|
| SMLIGHT SLZB-MR4 | **[bought]** | Dual-radio LAN/PoE coordinator. Zigbee (CC2674P10) + Thread (EFR32MG26) simultaneously on separate radios. Up to 750 devices. Web UI (SLZB-OS) for firmware updates. |
| Mercusys MS105GP | **[bought]** | 5-port Gigabit PoE+ switch, 65W, ~£18. TP-Link sub-brand. Used to power MR4 and anything else in the loft. |

### Plan
- Mount MR4 in loft, centrally placed for line-of-sight coverage through ceilings
- PoE via Mercusys switch (MR4 draws only ~1.5W)
- Caution: foil-backed loft insulation can block Zigbee — mount MR4 **below** the foil
- HA integration: Zigbee2MQTT (preferred for Aqara compatibility) or ZHA
- MR4 also acts as Thread Border Router for future Matter-over-Thread devices

### HA config (when ready)
```yaml
# Zigbee2MQTT port setting
port: tcp://192.168.1.XX:6638   # replace XX with MR4's assigned IP
```

---

## Project 2 — Loft Smart Relay Switches (ceiling rose wiring)

**Goal:** Replace existing dumb wiring at ceiling roses in loft with smart relays that keep
the existing wall switches working even if HA/Zigbee is offline.

### Key requirement
Existing wall switch wiring sends 230V back up to the loft (same as Qubino Z-Wave modules).
Most cheap Zigbee modules (Sonoff MINI DUO, Sonoff ZBMINI-L2) use **low-voltage dry contact**
inputs — these would be destroyed by 230V. Only modules supporting **wet contact (230V) switch
input** are suitable.

### Hardware

| Item | Status | Notes |
|------|--------|-------|
| Aqara Dual Relay T2 (DCM-K01) | **[bought]** | Zigbee 3.0. Wet contact mode: red jumper bridges L→LIN, S1/S2 accept 230V from wall switches. 2 channels. DIN rail mount (ideal for loft joist). Energy monitoring. ~£27.99 (20% off). |

### Wiring (wet contact mode)
```
L + N → Module power
Red jumper: L → LIN (enables wet contact)
COM: from wall switch common
S1, S2: 230V switched-live returns from 2-gang wall switch
L1, L2: out to ceiling roses
```

### HA behaviour
- Appears as Zigbee switch in Zigbee2MQTT/ZHA
- Acts as Zigbee router (mesh extender) — important for battery sensors nearby
- Wall switch works locally even if Zigbee/HA is offline (relay firmware handles it)

### Rejected alternatives
- **Sonoff MINI DUO Zigbee** — S1/S2 are 3.3V dry contact only; 230V would fry it
- **Candeo Zigbee 2-Gang** (~£39) — works but more expensive than T2 at £27.99
- **Moes/Tuya budget modules** — cheaper but small terminals, fiddly with 1.5mm T&E

---

## Project 3 — Lounge Dimmer Replacement

**Goal:** Replace Qubino Flush Dimmer Plus (Z-Wave, momentary switch) with a Zigbee
rotary dimmer. Momentary switch is poor UX; user wants push click on/off + rotate to dim.

### Location
Extension room with **flat solid roof** — no loft access. Access to ceiling rose from inside
the room. Existing 25mm backbox (likely needs a 10mm spacer or deepening to 35mm).
5 × LED candle bulbs (~4W each = 20W total). **No neutral wire at switch**.

### Hardware

| Item | Status | Notes |
|------|--------|-------|
| Candeo RD1 Zigbee Rotary Dimmer | **[bought]** | Push click on/off + rotate to dim. Energy monitoring (W/V/A/kWh). No-neutral capable (min 10W load — 5 bulbs at 20W is fine). Zigbee 3.0, pairs with MR4. ~£54. |
| 10mm single-gang spacer | **[planned]** | ~£2. Needed to fit RD1 in 25mm backbox while testing. |
| Candeo LED Bypass (C130-S) | **[planned, conditional]** | ~£12. Only needed if bulbs glow faintly at night when switch is off. Wire in parallel at ceiling rose. Try without first. |

### No-neutral notes
- RD1 "steals" tiny leakage current through bulb circuit when off
- 5 bulbs spreads the load — less likely to glow than a single bulb setup
- 70% chance it works out of the box; have bypass ready as backup

### 2-gang situation (outside light on same plate)
A second switch on the same plate controls a **non-dimmable LED floodlight**. Solution:
- Use 2-gang RD1 (or RD1-Pro)
- Set **minimum brightness to 99%** on the floodlight gang in HA → knob becomes on/off only
- RD1 is rated 250W LED; floodlight draws ~30-50W actual — well within limits
- Consider RD1-Pro for "decouple" mode which handles this more elegantly

### 2-way dimming (different location — hall/landing)
Separate 1-gang backbox currently has 2×2-way switches. Plan:
- **Circuit A** (dimmer): Candeo RD1 master + **RD1C companion** at other end
  - Uses existing traveller wires for digital link (not conventional L1/L2)
  - Varilight 2-module plate: frame **83362**, faceplate **40552**, adapter **YGAP**
- **Circuit B** (conventional): Varilight toggle module **83626** — wiring unchanged
- **Screwfix parts:** 83362 (frame), 40552 (faceplate), 83626 (toggle), YGAP (adapter)
- RD1C needed for other end — buy as pair

---

## Project 4 — Conservatory Window Automation

**Goal:** Replace failed Ventec 100 controller (230V, 3-wire actuator: brown=open,
black=close, blue=neutral) with smart window control + presence + rain sensing.

### Backbox details
- Location: corner of conservatory, ~1.5m high
- Depth: **55mm** (deep box flush with brick + 10mm plaster) — plenty of room
- Ventec was double-width; replacing with 4-module plate

### Hardware

| Item | Status | Notes |
|------|--------|-------|
| Shelly Plus 2PM Gen 4 | **[bought]** | WiFi 6 + Zigbee (native). Cover mode with hardware interlock (prevents open+close simultaneously). Power monitoring for calibration. Gen 4 chosen for Zigbee mesh capability. |
| Tuya ZY-M100 Zigbee mmWave sensor | **[bought]** (Amazon B0CST2MQ43) | 24GHz mmWave, 100° FOV, detects still presence (breathing). Micro-USB powered. Will be mounted angled 45° inside wall box on wedge (polystyrene recommended). |
| HLK-PM01 5V PSU | **[planned]** | Compact AC→5V module. Powers ZY-M100 from mains in the wall box. Wire +5V/GND to ZY-M100 Micro-USB (cut cable). |
| Tiardey "Sunflower" Zigbee rain+light sensor | **[bought]** (Amazon B0DGG5FR3Q) | Solar-powered, IPX6. Capacitive rain detection. Also reports Lux. Mount outside conservatory tilted 15-20° for rain runoff. South/SW-facing for solar charging. |
| Varilight 4-module faceplate | **[planned]** | Screwfix **40558** (white) |
| Varilight 4-module grid frame | **[planned]** | Screwfix **90137** |
| Varilight 2-way centre-off retractive switch | **[planned]** | Screwfix **27120** — single rocker, tilt up=open, tilt down=close, springs to centre |
| Varilight blank modules ×3 | **[planned]** | Screwfix **78350** |

### Wiring (Shelly 2PM)
```
L  → Permanent live (also to switch COM and HLK-PM01 L)
N  → Neutral (also to HLK-PM01 N and motor blue)
O1 → Motor brown (Open)
O2 → Motor black (Close)
S1 → Varilight switch L1 (Open)
S2 → Varilight switch L2 (Close)
```
If window opens when you press Close: toggle "Reverse Directions" in Shelly app — don't
re-wire.

### Presence sensor mounting
- Remove ZY-M100 PCB from its housing
- Mount bare PCB on 45° polystyrene wedge inside wall box, behind one of the blanks
- Polystyrene is ~100% transparent to mmWave; white plastic faceplate ~0% signal loss
- Illuminance sensor disabled/ignored (lux values are attenuated behind faceplate;
  sunrise/sunset from HA Sun integration used instead)
- Set Radar Sensitivity to 7 initially; adjust if glass reflections cause ghost presence

### HA automation logic (conservatory window)
```yaml
# Open: presence detected AND temp >22°C (after sunrise, before sunset)
# Close triggers (any one):
#   - Rain sensor detects rain (immediate, overrides all)
#   - After sunset
#   - Presence clear for 15+ minutes
```

### ZY-M100 Zigbee chattiness
Known to send updates every few seconds. If Zigbee network gets sluggish, set minimum
report interval for illuminance to 3600s in Zigbee2MQTT device settings.

---

## Project 5 — Presence Sensing (general)

### Conservatory — ZY-M100 (covered above)

### Other rooms — future
- **Aqara FP1E** (~£45): Higher-end 24GHz mmWave, AI filtering, good for rooms with
  moving fans/curtains. 120° FOV, 6m range — covers 5×5m room from corner at chest height.
  Considered as alternative to ZY-M100.
- **Sonoff SNZB-06P** (~£18): Budget 24GHz mmWave. USB-C powered. Good Zigbee mesh citizen
  (less chatty than Tuya). Recommended if adding more presence sensors later.
- **Avoid:** £10 Knadgbft/generic Tuya 5.8GHz sensors — network-spamming, prone to ghosting

---

## Rejected / not pursued

| Item | Reason |
|------|--------|
| Sonoff MINI DUO (Zigbee or WiFi) | S1/S2 dry contact only — incompatible with 230V switch wiring |
| Aqara H2 EU Thread/Zigbee switch | Discussed for loft ceiling switches but T2 relay chosen instead |
| SwitchBot | BLE only — doesn't talk to MR4; would need separate BT proxy |
| Aqara FP2 | Overkill and expensive for conservatory use case |
| Shelly Plus 2PM Gen 3 | Gen 4 chosen for native Zigbee support |

---

## Integration notes for HA setup

- MR4: add via Zigbee2MQTT or ZHA as network coordinator
- Aqara T2: pairs as Zigbee device; configure "wet contact" mode via app or jumper before
  pairing; verify S1/S2 polarity matches expected open/close
- Shelly 2PM: set to Cover mode + Button input + run calibration before automations
- ZY-M100: disable Illuminance entity in HA (Settings → Devices → entity → Disable)
- Tiardey sunflower: Tuya-ecosystem Zigbee; may need Z2M external converter if unsupported;
  disable onboard siren (120dB) in device settings
- Candeo RD1: pairs as Zigbee dimmer; set minimum brightness floor via HA device settings
  if LEDs flicker at low end
