# Home Infrastructure — Claude Code Project

This is Simon's home infrastructure management project.
It lives in WSL2 on a Windows 11 workstation and is the primary context
file for Claude Code sessions working across the home estate.

This file IS the handoff from an earlier design conversation — read it fully
before acting on any task. It contains both estate topology (facts) and a
task list (things not yet done).

---

## About Simon

- Primary dev environment: VS Code + Claude Code extension on Windows 11 (WSL2)
- Also has M1 MacBook Pro
- Comfortable with Node.js, Docker, PostgreSQL, bash
- Uses DBeaver for DB access, GitHub for version control
- Preference: pragmatic > perfect; start read-only, earn trust before writes

---

## Estate Topology

### Networking
- Virgin Media Hub 5X in DMZ/modem mode
- AiMesh network behind it (multiple nodes)
- Local LAN: 192.168.1.0/24

### Synology NAS (primary Docker host)
- Runs Docker via Portainer Community Edition 2.33.2 LTS
- Portainer URL: https://192.168.1.137:9443
- All primary service stacks run here
- Volume base path: /volume2/docker/

### Home Assistant (on NAS, Docker)
- Container: `ghcr.io/home-assistant/home-assistant:stable`
- Install type: **Home Assistant Core** (Docker container, NOT HAOS/Supervised)
- Therefore: NO add-on store, NO Supervisor API
- Config volume: /volume2/docker/homeassistant/config
- Network: host mode
- Status: vanilla install, migration from OpenHabian in progress
- URL: http://192.168.1.137:8123 (confirm IP)
- HACS: not yet confirmed as installed — check before assuming
- Integrations planned but not yet installed:
  - audi_connect_ha (Audi Q5 TFSIe vehicle status)
  - alexa_devices or alexa_media_player (Alexa announcements)
  - Z-Wave JS integration (pointing at Pi 5 Z-Wave JS instance)

### Linux VPS (Hostinger)
- Runs PM2/systemd services
- nginx for HTTPS termination
- Used for externally-facing services

### Raspberry Pi 5 (on LAN)
- Runs Z-Wave JS UI 11.5.2 (standalone Docker container, not the HA add-on)
- Current stick: **Aeotec Z-Stick 10 Pro** (800-series) — no devices paired yet
- Z-Wave JS UI: http://192.168.1.222:8091
- Z-Wave JS WebSocket port: 3000 — **currently disabled**, must enable before HA integration
- Will be integrated into HA via Z-Wave JS integration (WebSocket)
- Also has Zigbee2MQTT running (~/ha/zigbee2mqtt) — not yet part of migration plan

### Raspberry Pi 3B (OpenHabian)
- Running OpenHabian (OpenHAB-based home automation)
- This is the LEGACY system being replaced by HA on the NAS
- Migration is in progress — not all devices/automations moved yet
- Keep this documented until migration is complete and Pi 3B is decommissioned
- OpenHAB URL: http://192.168.1.78:8080
- SSH: openhabian@192.168.1.78
- OpenHAB config is also mapped as **drive P:** on the Windows host (not accessible from
  WSL2 — use SSH from WSL2, or read directly via P:\ from a Windows context)

### Amazon Echo devices (Alexa)
- Multiple Echo devices around the house
- UK Amazon account (amazon.co.uk — important for Alexa integration region)
- Will be used for voice announcements from HA automations

---

## Z-Wave Controller Migration (Gen 5 → Z-Stick 10 Pro)

### Current state
- **Gen 5** (500-series, Aeotec): on Pi 3B, paired to entire Z-Wave network, working with OH
- **Z-Stick 10 Pro** (800-series, Aeotec): on Pi 5, running Z-Wave JS UI, no devices paired yet
- Z-Wave home ID on Pi 3B network: `0xde2f557d`

### Why Gen 5 can't plug directly into Pi 5
The Gen 5 has a non-compliant USB electrical behaviour: on connect it pulls D+ to +5V
(spec requires 3.3V). Pi 4 and Pi 5 USB host controllers treat this as an invalid SE1
state and never enumerate the device. This is a hardware issue in the stick, not a driver
issue. **Fix: USB 2.0 hub between Pi 5 and Gen 5** — the hub absorbs the non-compliant
pull-up. Users confirmed this works reliably on Pi 4/5.

### Migration path (500-series → 800-series via Z-Wave JS UI)
Aeotec have an official guide for this using the Z-Wave JS UI NVM backup/restore.
Reference: https://aeotec.freshdesk.com/support/solutions/articles/6000279670

Prerequisites:
- Gen 5 firmware must be **V1.2** (check in Z-Wave JS UI controller info when connected)
- Z-Stick 10 Pro firmware must be **V7.23.2** or later (check in Z-Wave JS UI now)

Steps (when ready to cut over):
1. Connect Gen 5 to Pi 5 via USB 2.0 hub; stop/swap Z-Wave JS from Z-Stick 10 Pro
2. Z-Wave JS UI → Controller → NVM Backup (wait ~1 min, .bin file downloads)
3. Swap back to Z-Stick 10 Pro
4. Z-Wave JS UI → Controller → NVM Restore with the .bin file
5. Enable WebSocket server, connect HA Z-Wave JS integration
6. Verify all nodes appear in HA; cross-reference against zwave/node-map.md

### Strategy: keep Gen 5 on Pi 3B until ready to cut over
- **Do NOT put Z-Stick 10 Pro on Pi 3B** — OH 2.5.10 predates 800-series support,
  compatibility unknown and not worth the risk
- Gen 5 stays on Pi 3B running OH until HA is ready to go live
- Migration is a single cutover event, not a gradual swap

### Gen 5 battery / post-migration
- Gen 5 has an internal Li-ion battery (for standalone inclusion/exclusion mode)
- Battery auto-powers-down when unplugged and idle — won't act as a rogue controller
- After migration, keep Gen 5 physically away from the network while verifying HA
- Once stable: factory reset Gen 5 (hold button until LED flashes) to clear home ID
- Gen 5 can then be kept as a spare

### Immediate next step
Connect Gen 5 to Pi 5 via USB 2.0 hub → confirm it enumerates in Z-Wave JS UI →
check firmware version → check Z-Stick 10 Pro firmware. Do NVM backup only when
ready to cut over to HA (don't do it prematurely while OH still needs to be live).

---

## Key Project: Audi Q5 Vehicle Monitoring

See `homeassistant/docs/AUDI.md` for the full design document.

### Vehicle
- Audi Q5 TFSIe PHEV, Vorsprung Competition, registered late 2021
- App shows: Lock, Doors, Windows, Sunroof, Bonnet, Tailgate, Lights
- PHEV stack likely gives richer per-component status than pure ICE Q5s

### Goals
1. Pull vehicle status into HA as entities
2. Alert if windows or sunroof left open (with debounce — not while driving)
3. Alert if car left unlocked (with debounce — not mid-load)
4. Alexa announces specific condition e.g. "Your car sunroof is open"
5. Simon handles his own notification routing from HA entities/input_booleans

### Integration approach
- Data source: `audi_connect_ha` HACS custom integration
  - Repo: https://github.com/audiconnect/audi_connect_ha (verify still current)
  - Auth: myAudi username/password, region = DE
  - Fragile: reverse-engineered API, has broken before on Audi auth changes
  - Poll interval: ~15 min passive; avoid frequent force-refresh (12V battery)
- Alert output: HA persistent notification + input_boolean + Alexa announcement
- Alexa: try official `alexa_devices` integration first; fall back to
  `alexa_media_player` (HACS, alandtse) if announcement features insufficient
- Automation must NOT call Claude/AI at runtime — pure deterministic HA logic

---

## Claude Integration Strategy

### Philosophy
- Claude is a **config/maintenance copilot** (role 1), not a live automation
  component (role 2). Claude acts when Simon asks it to, not unattended.
- Start **read-only** across all MCP connections. Open up writes once trust
  is established.
- For irreversible or high-consequence changes: Claude advises, Simon executes.

### MCP Servers (to be configured in Claude Code)

#### 1. Home Assistant — ha-mcp
- Repo: https://github.com/homeassistant-ai/ha-mcp
- NOTE: Cannot run as a HA add-on (Core/Docker install, no Supervisor)
- Must run as a standalone container in Portainer, or via uvx locally
- Read-only mode: YES (initial phase)
- Requires: HA long-lived access token
- Config for Claude Code (~/.claude.json or .mcp.json in this project):
  ```json
  {
    "mcpServers": {
      "homeassistant": {
        "command": "uvx",
        "args": ["ha-mcp"],
        "env": {
          "HA_URL": "http://192.168.1.137:8123",
          "HA_TOKEN": "<long-lived-token>",
          "READ_ONLY": "true"
        }
      }
    }
  }
  ```
- Alternatively: add as a sidecar container in the HA Portainer stack

#### 2. Portainer — official portainer/portainer-mcp
- Repo: https://github.com/portainer/portainer-mcp
- Works with Portainer CE (confirmed — CE/EE both supported)
- Portainer version: 2.33.2 LTS — use MCP version pin `~=2.33.0`
- Requires: Portainer API key (My Account → Access tokens in Portainer UI)
- Self-signed cert on NAS: set PORTAINER_TLS_VERIFY=0 if needed
- Config for Claude Code:
  ```json
  {
    "mcpServers": {
      "portainer": {
        "command": "uvx",
        "args": ["--from", "mcp-portainer~=2.33.0", "mcp-portainer"],
        "env": {
          "PORTAINER_URL": "https://192.168.1.137:9443",
          "PORTAINER_API_KEY": "<api-key>"
        }
      }
    }
  }
  ```
  Version confirmed: 2.33.2 LTS → pin `~=2.33.0`.

#### 3. Z-Wave JS (no MCP server exists)
- No dedicated Z-Wave JS MCP server available as of June 2026
- Strategy: Z-Wave devices will appear as HA entities once Z-Wave JS
  integration is configured in HA — access them via ha-mcp instead
- Document the Z-Wave node map in zwave/node-map.md for Claude context

#### 4. OpenHabian (no MCP server)
- Legacy system, being decommissioned — no MCP integration warranted
- Document migration status in openhabian/migration-status.md

---

## Repository Structure

```
home-infra/
├── CLAUDE.md                          ← you are here
├── .mcp.json                          ← Claude Code MCP server config (gitignored if contains secrets)
├── .env.example                       ← template for secrets (never commit .env)
│
├── homeassistant/
│   ├── docs/
│   │   └── AUDI.md                   ← Audi Q5 integration design doc
│   ├── config-snapshots/             ← periodic snapshots of key HA config files
│   │   ├── automations.yaml
│   │   ├── configuration.yaml
│   │   ├── scripts.yaml
│   │   └── scenes.yaml
│   └── README.md
│
├── portainer/
│   ├── stacks/                       ← copies of Portainer stack compose files
│   │   ├── homeassistant.yml
│   │   ├── zwave-js.yml              ← if Z-Wave JS moves to NAS
│   │   └── ...
│   └── README.md
│
├── zwave/
│   ├── node-map.md                   ← Z-Wave node ID → device name → location
│   └── README.md
│
└── openhabian/
    ├── migration-status.md           ← what's migrated, what's pending, blockers
    └── README.md
```

Secrets (tokens, passwords, IPs if preferred private) go in `.env`, which is
gitignored. Reference them in `.mcp.json` via environment variable substitution.

---

## Task List

Tasks are roughly sequenced. Complete prerequisites before dependents.

### Phase 0 — Project setup (WSL2, first session)
- [x] Create the directory structure above in WSL2
- [x] `git init`, create `.gitignore` (ignore `.env`, `*.env`, `secrets/`)
- [x] Create `.env.example` with all required variable names but no values
- [x] Push to GitHub (private repo) — https://github.com/sjlyoung58/home-infra
- [x] Confirm NAS IP, Pi 5 IP, Pi 3B IP — fill in this CLAUDE.md
- [x] Confirm Portainer version number — 2.33.2 LTS, MCP pin ~=2.33.0

### Phase 1 — Populate initial context
- [x] Copy current Portainer stack YAML for HA container into
      `portainer/stacks/homeassistant.yml`
- [x] Document Z-Wave node map in `zwave/node-map.md`
- [x] Begin `openhabian/migration-status.md`
- [x] Copy `AUDI.md` into `homeassistant/docs/AUDI.md`

### Phase 2 — MCP setup (Claude Code)
- [ ] Check whether `uvx` is available in WSL2; install if not (`pip install uv`)
- [ ] Check Portainer version, install matching `mcp-portainer` version via uvx
      and test it connects: `uvx --from "mcp-portainer~=2.33.0" mcp-portainer`
- [ ] Generate Portainer API key: Portainer UI → My Account → Access tokens
- [ ] Create `.mcp.json` in project root with Portainer MCP config (read-only)
- [ ] Generate HA long-lived access token: HA UI → Profile → Long-lived tokens
- [ ] Add ha-mcp to `.mcp.json` (read-only, HA_URL + HA_TOKEN)
- [ ] Test both MCP connections from a Claude Code session:
      "List my Portainer stacks" / "What HA entities are available?"
- [ ] Add secrets to `.env`, reference from `.mcp.json` via env vars
- [ ] Commit `.mcp.json` (no secrets), `.env.example`, confirm `.env` gitignored

### Phase 3 — HACS and Audi integration
- [ ] Confirm whether HACS is installed in HA (check HA → Settings → Integrations)
- [ ] If not: install HACS manually (Core/Docker install — must use the manual
      script method, not the add-on; see https://hacs.xyz/docs/setup/download)
- [ ] Add `audi_connect_ha` as custom HACS repository:
      https://github.com/audiconnect/audi_connect_ha
- [ ] Install and configure with myAudi credentials, region = DE
- [ ] Once connected, inspect entity list for this VIN — note in AUDI.md:
      - Are windows reported per-window or aggregate?
      - Is sunroof a separate entity?
      - Exact entity IDs for: lock, doors, windows, sunroof, bonnet, tailgate
- [ ] Update AUDI.md open questions section with findings

### Phase 4 — HA automations (vehicle alerts)
- [ ] Build "Vehicle Security Check" automation in HA
      (Claude can draft YAML once entity IDs are known from Phase 3)
  - Trigger: lock state or window/sunroof entity changes
  - Debounce: 10–15 min before alerting
  - Condition: exclude "currently driving" if ignition/speed entity available
  - Action: set input_boolean / persistent notification
- [ ] Set up Alexa integration:
  - Try official `alexa_devices` integration first (built into HA core)
  - Confirm it supports `announce` type (spoken announcement not just chime)
  - If insufficient: install Alexa Media Player via HACS (alandtse/alexa_media_player)
  - UK account: use amazon.co.uk domain during auth
- [ ] Add Alexa announcement action to vehicle alert automation
      e.g. "Your car sunroof is open" spoken on kitchen Echo

### Phase 5 — Z-Wave migration to HA
- [ ] Install Z-Wave JS integration in HA (Settings → Integrations → Z-Wave JS)
  - Point at Pi 5 Z-Wave JS WebSocket: ws://192.168.1.222:3000
  - **PREREQUISITE**: WebSocket server is currently DISABLED in Z-Wave JS UI settings
    Must enable it first: Z-Wave JS UI → Settings → Z-Wave → Enable WS Server
  - Do NOT migrate Z-Wave JS to NAS yet — keep it on Pi 5 for now
- [ ] Verify all Z-Wave devices appear as HA entities
- [ ] Cross-reference against node-map.md — flag any missing devices
- [ ] Rename/area-assign entities in HA to match OpenHabian device names
      (makes migration easier to track)

### Phase 6 — OpenHabian migration completion
- [ ] Work through migration-status.md systematically
- [ ] For each pending device: migrate to HA, test, mark done
- [ ] For each pending automation: recreate in HA, test, mark done
- [ ] When all items are migrated and verified:
  - Decommission Pi 3B / OpenHabian
  - Update this CLAUDE.md to remove OpenHabian references

### Phase 7 — Expand Claude access (when ready)
- [ ] Review ha-mcp read-only restriction — open up specific write tools
      (suggested order: helpers first, then automations, then config)
- [ ] Review Portainer MCP — enable write access for stack management
- [ ] Document which tools are enabled and any approval-gate preferences

---

## Open Questions / Decisions Pending

- Whether HACS is already installed
- Z-Wave JS WebSocket port confirmed: 3000 — but server currently DISABLED, must enable before Phase 5
- Alexa integration choice: official `alexa_devices` vs Alexa Media Player
  (re-evaluate at Phase 4 based on what announcement features are available)
- Whether to eventually move Z-Wave JS from Pi 5 to NAS Docker container

---

## Key References

- ha-mcp: https://github.com/homeassistant-ai/ha-mcp
- portainer/portainer-mcp (official): https://github.com/portainer/portainer-mcp
- audi_connect_ha: https://github.com/audiconnect/audi_connect_ha
- Alexa Media Player: https://github.com/alandtse/alexa_media_player
- HA Alexa devices integration: https://www.home-assistant.io/integrations/alexa_devices/
- HACS (manual install for Core/Docker): https://hacs.xyz/docs/setup/download
- Z-Wave JS UI: https://github.com/zwave-js/zwave-js-ui
