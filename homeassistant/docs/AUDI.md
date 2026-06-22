# Audi Q5 → Home Assistant Vehicle Status Monitoring

Status: planning / not yet implemented
Owner: Simon
Last updated: 2026-06-20

## Vehicle

- Audi Q5 TFSIe (plug-in hybrid), Vorsprung Competition, registered late 2021
- Currently monitored via the myAudi phone app
- App's vehicle status screen exposes (confirmed by inspection): **Lock, Doors, Windows,
  Sunroof, Bonnet, Tailgate, Lights**
- Being a PHEV, this Q5 runs the same backend tier as e-tron models, which tends to expose
  richer per-component status via the API than pure ICE Q5s on older MIB2 stacks. Per-window
  granularity is plausible but not yet confirmed against the actual API payload — verify once
  integration is live (see Open Questions).

## Goal

1. Programmatic retrieval of vehicle status (lock state, doors, windows, sunroof, bonnet,
   tailgate, lights) and other available data (mileage, range/fuel, battery SoC, last-seen
   location/timestamp).
2. Warn if any window or the sunroof has been left open.
3. Warn if the car is unlocked.
4. Integrate with Home Assistant so warnings surface as HA entities/automations that Simon
   handles notification routing for himself (not asking the integration/automation to push
   notifications directly).
5. **Alexa announcement**: as part of the alert logic, have an Alexa device speak an
   announcement, e.g. *"Your car sunroof is open"*, when the relevant condition is detected.

## Environment

- Home Assistant already running (install method TBC — HAOS/Supervised vs Docker/Container;
  confirm before implementation, as it changes the HACS install steps)
- HACS install status TBC — confirm before implementation
- Synology NAS available as a Docker host if a standalone/companion service is ever preferred
  over a pure HA custom integration

## Architecture (planned)

```
myAudi cloud API (unofficial, same backend the phone app uses)
        │
        ▼
audi_connect_ha  (HACS custom integration for Home Assistant)
        │
        ▼
HA entities: lock_state, doors, windows, sunroof, bonnet, tailgate,
             lights, mileage, range, battery SoC, last update, location
        │
        ▼
HA Automation: "Vehicle Security Check"
   - Trigger: any relevant entity changes state (or on poll update)
   - Condition: ignore transient states (e.g. don't fire while driving /
     within N minutes of last lock event, to avoid nagging mid-unlock)
   - Action:
       1. Set a persistent notification / input_boolean / HA "issue" for
          Simon's own downstream notification handling
       2. Trigger Alexa announcement (see below) for at least the
          sunroof/window-open and unlocked cases
```

## Component 1: Audi data source

- **Integration**: `audi_connect_ha` (community-maintained fork; the actively maintained
  fork as of June 2026 lives under `audiconnect/audi_connect_ha` on GitHub — verify this is
  still the maintained one at implementation time, as forks have churned historically:
  originally `arjenvrh`, then various community continuations including `sMau`, `acdcnow`,
  `timgursky`).
- **Install method**: HACS custom repository (preferred) or manual copy into
  `custom_components/audiconnect/`.
- **Auth**: same myAudi username/password as the phone app. Optional S-PIN for actions
  requiring it (lock/unlock, climatisation, etc. — not needed for read-only status).
- **Region setting**: DE (Europe).
- **Known fragility**: this is a reverse-engineered API, not an official Audi developer
  product. It has broken before when Audi/VW Group changed their identity/OAuth backend
  (e.g. a 2021/2022-era breakage requiring a manual endpoint patch in
  `audi_services.py`). Expect occasional integration updates to be needed; don't treat this
  as "install once and forget."
- **Polling**: minimum interval is typically 15 minutes for passive cloud-data polls (doesn't
  wake the car). There's also a "force refresh" service that pings the vehicle directly —
  use sparingly, as frequent direct vehicle pings can affect 12V battery drain over time on
  some models.

## Component 2: HA automation logic

Two trigger conditions, ideally combined into one automation:

1. **Unlocked warning**
   - Trigger: lock-state entity changes to "unlocked"
   - Debounce: require it to persist for ~10 minutes before alerting (avoid false positives
     while Simon is mid-unlock / loading the car)

2. **Window / sunroof open warning**
   - Trigger: any window entity or the sunroof entity ≠ "closed"
   - Condition: combine with a "not currently driving" signal if available (e.g. ignition/
     speed state), so a window cracked open while driving doesn't trigger a false alarm
   - Debounce: similar grace period (e.g. 15 minutes) before alerting

Output of both: set a shared `input_boolean` or persistent notification that Simon's own
downstream logic can consume, plus fire the Alexa announcement action below.

## Component 3: Alexa announcement

Requirement: when the sunroof/window-open or unlocked condition fires, an Alexa device in
the house should **speak** an announcement (not just chime) — e.g. *"Your car sunroof is
open."*

Two viable approaches in HA, with a maintenance trade-off to decide on at implementation
time:

1. **Official `alexa_devices` core integration** (built into HA since ~2025.6)
   - Action: `notify.send_message` targeting `notify.<echo_device>_announce`, or the
     `alexa_devices.send_announcement`-style action if available at implementation time
   - Pros: officially maintained by HA core team, no HACS dependency, more stable across HA
     updates
   - Cons: newer, narrower feature set — confirm at implementation time whether
     all-device/broadcast announcements and full TTS phrasing are supported, or whether it's
     still catching up to Alexa Media Player's capabilities

2. **Alexa Media Player (HACS custom integration, `alandtse/alexa_media_player`)**
   - Action: `notify.alexa_media` (or `notify.send_message` in newer syntax) targeting one or
     more `media_player.<device>` entities, with `data: { type: announce }` for the
     intercom-style "ding, announcement" behaviour rather than playing as ordinary TTS
   - Pros: mature, more flexible, supports targeting multiple/all Echo devices, well
     documented community blueprints exist for this exact use case
   - Cons: unofficial, has a history of breaking after HA or Amazon-side changes; requires
     Amazon account login through HACS integration (same caution as Audi: regional Amazon
     domain matters — sign in to the amazon.co.uk domain, not amazon.com, for UK accounts)

**Recommendation to revisit at implementation time**: try the official `alexa_devices`
integration first since it's already built into HA; fall back to Alexa Media Player only if
its announcement/broadcast capabilities turn out to be insufficient.

Example shape of the final automation action step (illustrative, not tested):

```yaml
action: notify.send_message
data:
  message: "Your car sunroof is open"
target:
  entity_id: notify.kitchen_echo_announce
```

## Open questions / to confirm during implementation

- [ ] Confirm HA install method (HAOS/Supervised vs Docker) — changes HACS install steps
- [ ] Confirm whether HACS is already installed
- [ ] Confirm `audi_connect_ha` maintained fork is still current at implementation time
      (check GitHub activity — this ecosystem has forked multiple times)
- [ ] Once connected, inspect actual entity list for this VIN to confirm: per-window
      open/closed granularity vs aggregate-only, sunroof reported separately, bonnet/
      tailgate/lights entity names
- [ ] Decide debounce thresholds for both alert conditions (avoid nagging during normal
      use)
- [ ] Decide official `alexa_devices` vs Alexa Media Player for announcements, based on
      what announcement features are available in HA at implementation time
- [ ] Confirm Amazon account region/domain for Alexa devices (UK: amazon.co.uk)
- [ ] Decide where the non-Alexa "warning" output lands, since Simon is handling that
      himself (dashboard card? separate notify service? logbook entry?)

## References

- `audi_connect_ha` (HACS integration): https://github.com/audiconnect/audi_connect_ha
- Original API discovery / underlying library: https://github.com/davidgiga1993/AudiAPI
- HA official Alexa integration: https://www.home-assistant.io/integrations/alexa_devices/
- Alexa Media Player (HACS): https://github.com/alandtse/alexa_media_player
- HACS: https://hacs.xyz
