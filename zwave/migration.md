# Z-Wave Controller Migration — Field Guide

## Context for Claude Code on Pi5

This file is a self-contained working guide for the Z-Wave controller migration procedure.
It is intended to be read by a Claude Code session running on the Pi5 itself.

### What this migration does
Transfers the entire Z-Wave network from the old **Aeotec Z-Stick Gen5** (500-series,
currently on the Pi3B running OpenHAB) to the **Aeotec Z-Stick 10 Pro** (800-series,
already installed on this Pi5 running Z-Wave JS UI).

After the migration, the Z-Stick 10 Pro will have the same home ID and paired devices as
the Gen5, and all devices will be controllable via Home Assistant.

### Key facts

| Item | Value |
|------|-------|
| This machine | Pi5 — simonyoung@192.168.1.222 |
| Z-Wave JS UI | http://192.168.1.222:8091 |
| Z-Wave JS container | `zwave-js-ui` (Docker, running) |
| Z-Wave JS store | `/DockerVolumes/ha/zwave/store/` |
| Settings file | `/DockerVolumes/ha/zwave/store/settings.json` |
| Current stick | Z-Stick 10 Pro (800-series), port: `/dev/zwave` or `/dev/ttyUSB0` |
| Current home ID | `0xcd2fc96a` (Z-Stick 10 Pro's own network — 2 nodes, just test socket) |
| Target home ID | `0xde2f557d` (Gen5 network from Pi3B — all OH devices) |
| HA connection | ws://192.168.1.222:3001 (port 3001 — port 3000 occupied by video-library app) |
| Pi3B (OH) | openhabian@192.168.1.78 — Gen5 plugged in here, running OpenHAB 2.5.10 |
| NAS (HA) | simon@192.168.1.137 — Home Assistant on Docker |
| Gen5 USB issue | Gen5 has non-compliant USB (D+ pull-up). **Cannot plug directly into Pi5.**  Must use a USB 2.0 hub between Pi5 and Gen5. |
| Gen5 firmware req | Must be V1.2 for NVM migration to work — check in Z-Wave JS UI controller info |
| Z-Stick 10 Pro FW | SDK v7.23.2 confirmed — migration prerequisite met |

### Files in this directory
- `node-map.md` — full list of all Z-Wave nodes (derived from OpenHAB items), with
  GWPN1 post-migration configuration notes
- `migration.md` — this file

---

## Pre-flight checks (do these before starting)

- [ ] USB 2.0 hub to hand
- [ ] Gen5 currently plugged into Pi3B and OH is running normally
- [ ] Z-Stick 10 Pro plugged into Pi5, Z-Wave JS UI accessible at http://192.168.1.222:8091
- [ ] Decide where to save the NVM backup .bin file (suggestion: this directory)
- [ ] Note: HA will lose Z-Wave during this procedure — expected and harmless

---

## Step-by-step procedure

### Phase 1 — Prepare Pi5

**1.** Note current Z-Wave JS state (for reference if rollback needed):
```bash
python3 -c "
import json
with open('/DockerVolumes/ha/zwave/store/settings.json') as f:
    s = json.load(f)
print('port:', s['zwave']['port'])
print('serverPort:', s['zwave']['serverPort'])
"
```

**2.** Stop Z-Wave JS container:
```bash
docker stop zwave-js-ui
```

**3.** Unplug Z-Stick 10 Pro from Pi5.

---

### Phase 2 — Connect Gen5 to Pi5 via USB hub

**4.** Connect USB 2.0 hub to Pi5, plug Gen5 into the hub.

**5.** Confirm Gen5 enumerated — look for a new tty device:
```bash
dmesg | tail -20
```
Look for a line like: `cp210x ... ttyUSB0: CP210x UART converter now attached`
Note the device path (e.g. `/dev/ttyUSB0` or `/dev/ttyACM0`).

**6.** Confirm what's available:
```bash
ls /dev/tty{USB,ACM}* 2>/dev/null
```

---

### Phase 3 — Point Z-Wave JS at Gen5

**7.** Update the port in settings.json to the Gen5 device path from step 5:
```bash
# Check current port first
python3 -c "import json; s=json.load(open('/DockerVolumes/ha/zwave/store/settings.json')); print(s['zwave']['port'])"
```

Then update it (replace `/dev/ttyUSB0` with actual path if different):
```bash
python3 - <<'EOF'
import json
path = '/DockerVolumes/ha/zwave/store/settings.json'
with open(path) as f:
    s = json.load(f)
s['zwave']['port'] = '/dev/ttyUSB0'   # ← change to actual Gen5 device path
with open(path, 'w') as f:
    json.dump(s, f, indent=2)
print('port updated to:', s['zwave']['port'])
EOF
```

> Note: if `PermissionError`, the file is owned by the container. Use the Z-Wave JS UI
> Settings → Z-Wave → Serial Port field to change it after starting the container.

**8.** Start Z-Wave JS:
```bash
docker start zwave-js-ui
```

**9.** Open Z-Wave JS UI (http://192.168.1.222:8091) and confirm it's connected to the
Gen5 (should show home ID `0xde2f557d` and all the OH nodes, not `0xcd2fc96a`).

---

### Phase 4 — Check Gen5 firmware (critical prerequisite)

**10.** In Z-Wave JS UI → node 001 (controller) → check firmware version displayed.
- **FW: v1.2** → proceed ✓
- **Not v1.2** → STOP. Firmware update required before proceeding (separate procedure,
  needs Aeotec firmware updater tool). Do not proceed without V1.2.

---

### Phase 5 — NVM Backup

**11.** In Z-Wave JS UI → **Control Panel** → Controller section → **Backup NVM**

**12.** Wait approximately 1 minute. A `.bin` file will download in the browser.

**13.** Save the .bin file. Suggested path on Pi5:
```
/home/simonyoung/projects/home-infra/zwave/nvm_backup_gen5_<date>.bin
```
Keep this file safe — it contains Z-Wave security keys. Do NOT commit to GitHub.

Add to .gitignore if needed:
```bash
echo "zwave/*.bin" >> /home/simonyoung/projects/home-infra/.gitignore
```

---

### Phase 6 — Swap back to Z-Stick 10 Pro

**14.** Stop Z-Wave JS:
```bash
docker stop zwave-js-ui
```

**15.** Unplug Gen5 from hub, remove hub from Pi5.

**16.** Plug Z-Stick 10 Pro back into Pi5 (directly, no hub needed).

**17.** Confirm 10 Pro enumerated:
```bash
dmesg | tail -10
```

**18.** Update settings.json port back to 10 Pro's device path (probably `/dev/zwave` or
`/dev/ttyUSB0` — check `dmesg` output):
```bash
python3 - <<'EOF'
import json
path = '/DockerVolumes/ha/zwave/store/settings.json'
with open(path) as f:
    s = json.load(f)
s['zwave']['port'] = '/dev/zwave'   # ← adjust if needed
with open(path, 'w') as f:
    json.dump(s, f, indent=2)
print('port updated to:', s['zwave']['port'])
EOF
```

**19.** Start Z-Wave JS:
```bash
docker start zwave-js-ui
```

**20.** In Z-Wave JS UI — confirm 10 Pro is connected (home ID `0xcd2fc96a`, 2 nodes).

---

### Phase 7 — NVM Restore to Z-Stick 10 Pro

**21.** In Z-Wave JS UI → **Control Panel** → Controller section → **Restore NVM**
→ select the .bin file from step 13.

**22.** Wait for restore and automatic reinitialisation (may take several minutes).

**23.** After restart, Z-Wave JS UI should now show:
- Home ID: `0xde2f557d` (Gen5's home ID)
- All original OH nodes present

**24.** Cross-reference node list against `node-map.md` — all nodes should be present.
Note: the test GWPN1 test socket (node 002 on old home ID) will be gone — expected.

**25.** HA will pick up the restored network automatically via ws://192.168.1.222:3001.
Allow a few minutes for nodes to appear.

---

### Phase 8 — Return Gen5 to Pi3B and restore OH

**26.** Take Gen5 to Pi3B (openhabian@192.168.1.78).

**27.** On Pi3B, check which port OH's Z-Wave binding uses:
```bash
ssh openhabian@192.168.1.78 "ls /dev/tty{USB,ACM}* 2>/dev/null"
```

**28.** Plug Gen5 into Pi3B on the same port as before.

**29.** Confirm enumerated on Pi3B:
```bash
ssh openhabian@192.168.1.78 "dmesg | tail -10"
```

**30.** OH should reconnect automatically. If not, restart the Z-Wave binding:
```bash
ssh openhabian@192.168.1.78 "sudo systemctl restart openhab2 2>/dev/null || true"
```
Or via Karaf console on Pi3B:
```bash
ssh openhabian@192.168.1.78
openhab-cli console
# then in console:
bundle:restart org.openhab.binding.zwave
```

**31.** Verify OH is working: check OH UI at http://192.168.1.78:8080, toggle a light,
confirm devices responding.

---

## End state

| System | Expected state |
|--------|---------------|
| Pi3B (OH) | Gen5 plugged in, all Z-Wave devices working in OH |
| Pi5 (Z-Wave JS) | Z-Stick 10 Pro has Gen5 network (home ID `0xde2f557d`) |
| HA (NAS) | Z-Wave integration connected, all nodes visible at ws://192.168.1.222:3001 |

OH and HA are both running simultaneously with the same Z-Wave network — this is a
temporary state while migration is being tested. Once HA is fully configured, OH will
be decommissioned.

---

## If something goes wrong

**Rollback to pre-migration state:**
1. `docker stop zwave-js-ui` on Pi5
2. Unplug Z-Stick 10 Pro from Pi5
3. Plug Gen5 back into Pi3B (if it was removed)
4. OH resumes normally (Gen5 remembers all pairings)
5. Start Z-Wave JS with 10 Pro back on Pi5 (its original network `0xcd2fc96a` is gone
   after NVM restore — would need to re-pair devices from scratch, or restore from backup)

**Key safety note:** Keep the NVM backup .bin file safe. If the restore fails or
the 10 Pro needs to be reset, the .bin is the only recovery path short of re-pairing
all devices.
