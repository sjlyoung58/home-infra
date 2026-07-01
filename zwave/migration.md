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
| Currently connected to Pi5 (2026-07-01) | Spare Gen5+ (via USB hub) — see "Current status" section below for why |
| Z-Stick 10 Pro home ID | `0xcd2fc96a` (its own network — 2 nodes, just test socket; currently unplugged, set aside) |
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
- `xxxxGen5_NVM_20260701095038.bin` — **stale/pre-upgrade** NVM backup taken off the
  Gen5 while still at v1.1/SDK 6.51.10 (2026-07-01, pre-firmware-update). SDK too old
  to restore anywhere (see Phase 4 note) — kept for historical/rollback reference only,
  renamed with `xxxx` prefix to mark it as superseded/do-not-use.
- `gen5_1-2_NVM_20260701124605.bin` — fresh NVM backup taken off the Gen5 post-firmware-
  update (v1.2/SDK 6.81.6). This is the one actually used for the Phase 7 restore onto
  the Z-Stick 10 Pro — see Phase 7 outcome section for what happened.
  Both `.bin` files gitignored (`zwave/*.bin`), contain Z-Wave security keys, do not commit.

---

## Current status (2026-07-01) — read this first if resuming

**Where things physically stand right now:**
- The live Gen5 (v1.1, SDK 6.51.10, home ID `0xde2f557d`, all OH nodes) is back on the
  Pi3B running OH, **untouched** — never firmware-flashed, never restored-to.
- A spare, never-used **Gen5+** is currently plugged into the Pi5 via the USB hub, with
  `zwave-js-ui` running against it. It reports **FW v1.2, SDK v6.81.6** (Aeotec's
  `V1.02` release) — i.e. already at the target firmware, out of the box.
- `Gen5_NVM_20260701095038.bin` (backup of the live Gen5) is saved in this directory.
- Device mapping note: the Gen5 and Gen5+ both enumerate identically —
  `idVendor=0658, idProduct=0200`, no serial number in the USB descriptor, both show up
  as `/dev/serial/by-id/usb-0658_0200-if00` → `ttyACM0`. **The two sticks are visually
  indistinguishable at the OS/Docker level** — the only way to tell them apart is by
  what Z-Wave JS UI reports once connected (home ID / node count / firmware version).
  No Portainer stack device-mapping change was needed to swap between them.

**What we tried and found:**
- Test-restored the Gen5 backup onto the (already v1.2) Gen5+ to see if it would just
  work. It failed:
  ```
  Error while calling restoreNVM: Failed to convert NVM to target format:
  Did not find a matching NVM 500 parser implementation! Make sure that the
  NVM data belongs to a controller with Z-Wave SDK 6.61 or higher. (ZW0280)
  ```
  This confirms Z-Wave JS has no parser at all for NVM data below SDK 6.61 — a backup
  taken from firmware below that line **cannot be restored anywhere**, full stop. There
  is no converter, and no way to route around this via a different destination stick.
  (Full detail in the Phase 4 note below.)
- Investigated Z-Wave's built-in Controller Replication / "Learn Mode" as an alternative
  to NVM backup/restore entirely (it's exposed as a raw button in zwave-js-ui). Ruled
  out: per the zwave-js maintainer (GitHub discussion #7268, Oct 2024), full
  secondary-controller-promotion migration is an acknowledged but **unimplemented**
  feature — the exposed "Learn Mode" button only does a low-level network join, not full
  node-table replication + primary promotion. NVM backup/restore remains the only
  actually-supported migration path in zwave-js today.

**Decided plan (superseded — see outcome below) — transplant the network onto the
Gen5+ instead of firmware-flashing the live Gen5:**
1. Downgrade the **Gen5+** to v1.01 (SDK ~6.51, matching the backup) using Aeotec's
   `Z_Stick_G5_V1_01_DFU.zip` (requires a free Aeotec support-portal account to download)
2. Restore `Gen5_NVM_20260701095038.bin` onto the now-v1.01 Gen5+ — same SDK family, no
   format conversion needed, should succeed where the v1.2 test failed
3. Verify the restore: Gen5+ should now show home ID `0xde2f557d` and all OH nodes
4. **Do the risky v1.02 upgrade on the Gen5+ instead of the live Gen5** — if this bricks
   the Gen5+, nothing is lost; the live Gen5 is still sitting untouched on the Pi3B
5. Once the Gen5+ is confirmed at v1.2 with the full network restored, continue this
   guide from **Phase 5 onward** (NVM backup off the Gen5+ → restore onto the Z-Stick
   10 Pro) exactly as originally written — that hop (v1.2 500-series → 800-series) is
   the one Aeotec's own migration guide is designed for
6. The live Gen5 is never touched by any firmware operation in this plan — it remains
   the permanent rollback the whole way through

**Outcome (2026-07-01, actual path taken):** no `V1.01` downgrade image was available
for the **Gen5+** specifically (only for the original Gen5), so the Gen5+-transplant
plan above was abandoned. Simon instead used **Phase 4a-fallback**: flashed the
**live Gen5** directly to v1.02 via the Windows Aeotec updater.

Verified successful, thoroughly, at every layer:
- Driver log on reconnect: `home ID: 0xde2f557d`, firmware `1.2`, SDK `6.81.6`
- Controller reported `Z-Wave Classic nodes: 1, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, ...`
  directly from its own NVM — i.e. the update preserved the network in place as designed
- Every node (3–16, 20) pinged successfully and came back "ready to be used"
- Raw persisted cache file `de2f557d.jsonl` on the Pi5 confirmed full data for nodes
  1, 3–16, 20 — checked independently of any UI
- (There was a false alarm: zwave-js-ui's web UI briefly showed only node 001 in one
  browser tab after the stick reconnected — turned out to be a stale/cached browser
  session on that specific tab, not a data issue. Confirmed fine in a fresh
  Firefox session at http://localhost:8091/#/control-panel. Hard-refresh or a new
  browser/tab fixes this if it recurs.)

**The live Gen5 is now at v1.2/SDK 6.81.6** — i.e. it has itself crossed the SDK 6.61
threshold. The Gen5+ was never touched further and is not part of the plan going
forward. Took a fresh backup, `gen5_1-2_NVM_20260701124605.bin`, and proceeded with
Phase 5 onward — see next section for the outcome.

---

## Phase 7 outcome (2026-07-01) — cross-generation restore did NOT actually re-key the mesh

**What happened:** Excluded the GWPN1 test socket from the 10 Pro's own network first
(clean practice), then restored `gen5_1-2_NVM_20260701124605.bin` onto the Z-Stick 10
Pro. The restore/conversion **completed without error** this time (unlike the earlier
SDK<6.61 failure) and produced a **new home ID, `0xccf8fef4`** (different from both the
Gen5's `0xde2f557d` and the 10 Pro's old `0xcd2fc96a` — this new-home-ID behaviour is
the documented, expected way a proper 500→800-series conversion works, generating a
fresh ID and — supposedly — reprogramming it across the mesh to every node).

The controller's own node table (`ccf8fef4.jsonl` cache file) showed entries for all
expected nodes: **1, 3–16, 20**, plus an unexpected extra, **node 227** (very likely a
virtual/reserved node artifact of the cross-generation conversion — high ID near the
classic Z-Wave ceiling of 232, not a real device).

**But this did not mean real communication was established.** Digging into the driver
log for the full post-restore session (all 621 lines from reconnect to idle) showed:
- Every node got a **local** `GetNodeProtocolInfo` lookup (this only asks the
  controller chip what it has cached in its own NVM table — it is NOT a live RF
  message to the physical device, and proves nothing about reachability)
- **Zero** ping/alive/ready results for any of nodes 3–16, 20, 227 — the interview
  queue did the local lookups then fell straight into idle `GetBackgroundRSSI` polling
  and never actually tried to contact any real device
- Confirmed directly: node 4 ("Conservatory Garden Globe") showed **device type
  unknown** in the UI, and a manual ping **failed**

**Conclusion:** despite meeting the documented SDK≥6.61 prerequisite and completing
without error, this cross-generation (500-series → 800-series) NVM restore created
phantom table entries on the 10 Pro without actually reprogramming the physical
devices' stored home ID. This matches a known zwave-js community-reported failure mode
for this exact kind of restore (500-series firmware 1.2 → newer-generation controller
resulting in "all nodes dead"). The SDK≥6.61 prerequisite is necessary but evidently
**not sufficient** for a working migration.

**Safety check performed — production system confirmed intact:** since the real
devices never got the new home ID, they should still be listening for the old one.
Swapped the Gen5 back into the Pi3B/OH: **confirmed working** — controls the
conservatory lights correctly (initial response was slow, expected right after a
stick replug while OH re-syncs and the mesh re-settles routes). This proves the
physical mesh was never actually touched by the failed 10 Pro migration attempt — no
harm done to the production system at any point.

**Where this leaves the migration — decision point, not yet resolved:**
1. **Full re-inclusion**: put the 10 Pro into inclusion mode and physically re-pair
   each of the ~13 real devices from scratch (Qubino modules can be re-paired via
   their wired switch, 3 toggles — see the "Worst-case recovery note" above; GreenWave
   is easy physical access). Downside: loses old node IDs, any automations referencing
   them need rebuilding.
2. **Fall back to keeping the Gen5 as the permanent HA controller** via the USB hub on
   the Pi5, skipping the Z-Stick 10 Pro migration entirely. Sidesteps this
   cross-generation restore problem completely. Trade-off: permanent hub dependency,
   no Z-Wave Long Range or improved S2 security (the 800-series' main advantages this
   whole migration was chasing).

Not yet decided — pick up here next session.

**Worst-case recovery note (Gen5 or Gen5+ bricked beyond use, no matching-SDK spare
left to restore onto):** the physical Z-Wave devices themselves are unaffected — they're
independent radios waiting for a controller with the right home ID/keys. Recovery would
mean fresh inclusion of every node onto whichever controller is left working, losing
old node IDs (automations need rebuilding). One relevant detail for that scenario: the
Qubino modules (some in a loft, hard to reach physically) do **not** require physical
access to re-pair — they support inclusion/exclusion via 3 quick toggles of their wired
wall switch (On-Off-On-Off-On within ~5 seconds) while the controller is in
inclusion/exclusion mode. GreenWave sockets are not a concern (easy physical access).

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

> **settings.json's port does NOT need to change — only the Docker device mapping does.**
> The container always sees the stick at the same fixed internal path, `/dev/zwave`,
> regardless of which physical stick is plugged in. Docker remaps whichever host
> `/dev/serial/by-id/...` path is configured onto that fixed name. So `settings.json`
> should stay `"port": "/dev/zwave"` permanently — never a host path like `/dev/ttyUSB0`
> or `/dev/ttyACM0`, which don't exist inside the container's namespace at all.
>
> This container is managed as a **Portainer stack** (project `zwave`), and Portainer
> itself runs on the NAS (https://192.168.1.137:9443) — the Pi5 only runs a Portainer
> agent, there is no local compose file to hand-edit. The device mapping is hardcoded
> to the *current* stick's `/dev/serial/by-id/...` path in the stack's compose YAML, e.g.:
> ```
> /dev/serial/by-id/usb-Silicon_Labs_CP2105_Dual_USB_to_UART_Bridge_Controller_<serial>-if01-port0:/dev/zwave
> ```
> When you swap sticks, that by-id path no longer exists on the host, so `docker start`
> fails at the daemon level (`error gathering device information ... no such file or
> directory`) — **before** the container or settings.json are even read.

**7.** Confirm settings.json still has the fixed internal path (should be unchanged):
```bash
python3 -c "import json; s=json.load(open('/DockerVolumes/ha/zwave/store/settings.json')); print(s['zwave']['port'])"
```
Expected output: `/dev/zwave`. If it's anything else, fix it (needs `sudo`, file is owned
by the container):
```bash
sudo python3 - <<'EOF'
import json
path = '/DockerVolumes/ha/zwave/store/settings.json'
with open(path) as f:
    s = json.load(f)
s['zwave']['port'] = '/dev/zwave'
with open(path, 'w') as f:
    json.dump(s, f, indent=2)
print('port set to:', s['zwave']['port'])
EOF
```

**7a.** Update the Docker device mapping, **before** starting the container:
1. Find the Gen5's stable by-id path: `ls -la /dev/serial/by-id/`
2. Portainer UI → NAS (https://192.168.1.137:9443) → Stacks → `zwave` → edit compose YAML
3. Change the `devices:` line to point at the Gen5's by-id path (still mapped to
   `/dev/zwave` inside the container)
4. Redeploy the stack — this recreates the container with the corrected device mapping

**8.** Start Z-Wave JS (if not already started by the Portainer redeploy in 7a):
```bash
docker start zwave-js-ui
```

**9.** Open Z-Wave JS UI (http://192.168.1.222:8091) and confirm it's connected to the
Gen5 (should show home ID `0xde2f557d` and all the OH nodes, not `0xcd2fc96a`).

---

### Phase 4 — Check Gen5 firmware (critical prerequisite)

**10.** In Z-Wave JS UI → node 001 (controller) → check firmware version displayed.
- **FW: v1.2** → proceed ✓
- **Not v1.2** → STOP. Firmware update required before proceeding — go to Phase 4a below.
  Do not proceed without V1.2.

> Note: Z-Wave JS UI displays the Gen5's firmware without a leading zero, e.g. Aeotec's
> `V1.02` release shows as `v1.2`, and `V1.01` shows as `v1.1`. Confirmed on
> 2026-07-01: this stick reported `v1.1` — i.e. one release behind (`V1.01`), needs the
> `V1.02` update.

> **Confirmed 2026-07-01 — no way to route around this, verified safely on a spare stick:**
> Took an NVM backup of the live Gen5 (v1.1 / SDK 6.51.10) and test-restored it onto a
> spare, never-used Gen5+ that was already at v1.2 (SDK 6.81.6). The restore failed with:
> ```
> Error while calling restoreNVM: Failed to convert NVM to target format:
> Did not find a matching NVM 500 parser implementation! Make sure that the
> NVM data belongs to a controller with Z-Wave SDK 6.61 or higher. (ZW0280)
> ```
> This confirms the limitation is in the **source backup's SDK version**, not the
> destination stick — Z-Wave JS has no parser at all for NVM data below SDK 6.61, so a
> backup taken from the live Gen5 *before* updating its firmware cannot be restored
> anywhere (Gen5+ or 10 Pro), no exceptions. The live Gen5 must be updated to v1.2 first;
> there is no safe way to skip Phase 4a. (Test cost nothing — spare stick only, backup
> is read-only, live Gen5/network untouched throughout.)

---

### Phase 4a — Transplant the network onto Gen5+ (decided plan, supersedes flashing the live Gen5 directly)

**Do not firmware-flash the live Gen5.** Per the "Current status" section above, the
decided plan routes all firmware-flash risk onto the spare Gen5+ instead. This phase
moves the Gen5+ back and forth between the **Windows PC** (firmware flashing — Aeotec's
tool needs real Windows, not a VM, not the Pi5/WSL2) and the **Pi5** (NVM restore, which
only happens through the Z-Wave JS UI web interface). Each step below is tagged with
which machine it happens on.

**10a. [Windows]** Create a free account at https://aeotec.freshdesk.com/support/login
if you don't already have one (required to download firmware). Download:
- Downgrade firmware: `Z_Stick_G5_V1_01_DFU.zip`
- Upgrade firmware (needed later in this same phase): `DFU of Z-Stick_G5_EU_V1_02.zip`
  ([reference article](https://aeotec.freshdesk.com/support/solutions/articles/6000252294-z-stick-gen5-v1-02-firmware-update))
- Driver: `ZW050x_USB_Programming_Driver.zip`

**10b. [Windows]** Plug the Gen5+ directly into the Windows PC. Unlike on the Pi5, a
USB hub is probably not needed — the non-compliant D+ behaviour is specifically a
Raspberry Pi USB host controller issue (see Key facts / CLAUDE.md), and typical
Windows PC USB controllers are more tolerant of this kind of quirk. If it doesn't
enumerate, try adding a hub as a fallback.

**10c. [Windows]** Install the driver (pick one):
- Easy: unzip driver package, right-click `zw05xxprg.inf` → Install
- Manual: Device Manager → plug in stick → Ports → right-click `UZB (COMX)` →
  Update Driver → browse to extracted folder
- Alternative: run `CP210xVCPInstaller_x86.exe` or `_x64.exe` from the same package

**10d. [Windows] — Downgrade to v1.01:** Close any other software touching the stick.
Unzip `Z_Stick_G5_V1_01_DFU.zip`, run the updater exe inside, Settings → select the
Gen5+'s COM port → Update. Wait for completion, close, wait ~10s, unplug/replug.

**10e. [Windows/Pi5]** Confirm the downgrade. If the updater tool shows a version
readout, check it there; otherwise move the Gen5+ back to the Pi5 (Phase 2 style, via
hub) and check node 001 in Z-Wave JS UI — should now read **v1.1 / SDK ~6.51**, own
blank network (its previous v1.2 test network, if any, may be reset by the downgrade —
irrelevant, it's about to be overwritten anyway).

**10f. [Pi5] — Restore the real network:** With the now-v1.01 Gen5+ connected to the
Pi5 and `zwave-js-ui` running: Control Panel → purple floating "Advanced actions"
button → **General actions** → **NVM Management** → **Restore** → select
`Gen5_NVM_20260701095038.bin` from this directory. This should succeed now (same SDK
family as the backup, no format conversion needed — this is the exact operation that
failed in the Phase 4 test when the Gen5+ was still at v1.2).

**10g. [Pi5]** Verify: Gen5+ should now show home ID `0xde2f557d` and all the OH nodes
(cross-reference against `node-map.md`).

**10h. [Windows] — Upgrade to v1.02 (the "risky" step, now on the disposable stick):**
Move the Gen5+ (now holding the real network) back to the Windows PC. Unzip
`DFU of Z-Stick_G5_EU_V1_02.zip`, double-click `Z_Stick_G5_EU_V1_02.exe`, Settings →
select COM port → Update. Wait for completion, close, wait ~10s, unplug/replug.

> **Critical warnings (from Aeotec, unchanged):**
> - The update **can brick the stick** — no official recovery, voids warranty
> - Stick must be a 2018-or-later unit (age/hardware revision requirement)
> - Do not interrupt the update once started
> - **If this bricks the Gen5+:** nothing is actually lost — the live Gen5 is still
>   sitting untouched on the Pi3B running OH. You'd just need another spare stick and
>   to redo 10a–10g. This is exactly why the risk was moved here instead of onto the
>   live Gen5.

**10i. [Pi5]** Return the Gen5+ to the Pi5 (via hub), confirm Z-Wave JS UI shows
firmware `v1.2`, home ID `0xde2f557d`, and all OH nodes still present, then continue to
**Phase 5 below — using the Gen5+ as the source stick** (the live Gen5 is not involved
in Phase 5 onward at all; it stays on the Pi3B as the permanent rollback).

---

### Phase 4a-fallback — Flash the live Gen5 directly (not the decided plan — reference only)

If the Gen5+ transplant plan above turns out not to be viable (e.g. no downgrade file
available, restore fails for some other reason), the direct route is to firmware-update
the live Gen5 itself. This carries the bricking risk directly to the production stick —
only do this if Phase 4a above is not an option, and take a fresh NVM backup of the
Gen5 immediately beforehand regardless (extra insurance beyond the one already taken).

Steps are identical to Phase 4a's 10a/10c/10h (skip the downgrade/restore parts, this
stick is already v1.1 and already holds the real network) — download the same
`DFU of Z-Stick_G5_EU_V1_02.zip` + driver, install driver, run the updater directly
against the Gen5, same warnings apply. Once done, return the Gen5 to the Pi5 via the
USB hub and continue to Phase 5 as originally written.

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

**17.** Confirm 10 Pro enumerated, and note its by-id path:
```bash
dmesg | tail -10
ls -la /dev/serial/by-id/
```

**17a.** Same two-part update as Phase 3, step 7a — **do this before starting the
container**:
1. Portainer UI → NAS (https://192.168.1.137:9443) → Stacks → `zwave` → edit compose YAML
2. Change the `devices:` line back to the 10 Pro's by-id path (mapped to `/dev/zwave`)
3. Redeploy the stack

**18.** Confirm settings.json still has the fixed internal path (should not need to
change — see note in Phase 3):
```bash
python3 -c "import json; s=json.load(open('/DockerVolumes/ha/zwave/store/settings.json')); print(s['zwave']['port'])"
```
Expected output: `/dev/zwave`. If it's anything else, fix it (needs `sudo`):
```bash
sudo python3 - <<'EOF'
import json
path = '/DockerVolumes/ha/zwave/store/settings.json'
with open(path) as f:
    s = json.load(f)
s['zwave']['port'] = '/dev/zwave'
with open(path, 'w') as f:
    json.dump(s, f, indent=2)
print('port set to:', s['zwave']['port'])
EOF
```

**19.** Start Z-Wave JS (if not already started by the Portainer redeploy in 17a):
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
