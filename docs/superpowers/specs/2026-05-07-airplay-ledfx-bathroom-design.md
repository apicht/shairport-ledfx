# Audio-Reactive WLED in the Bathroom — Design

Date: 2026-05-07 (revised 2026-05-08)
Status: Approved; updated to reflect MQTT/HA integration and env-driven config rendering

## Goal

Drive audio-reactive WLED effects in the bathroom in sync with whatever is playing
on the bathroom speaker, with all software running in Docker on the QNAP NAS.

## Context

- Bathroom has a WLED-controlled LED setup.
- Bathroom speaker is wired to a zone of a multi-zone amplifier in the AV rack.
  The amp is fed by an Arylic LP10 network streamer (AirPlay 2, DLNA, Spotify
  Connect, Bluetooth).
- An ODROID N2+ runs Home Assistant and Music Assistant 2.8.6.
- A QNAP NAS hosts Docker via Container Station and is the deployment target.

The user's original framing — emulate a Chromecast receiver and pipe its audio
into LedFx — was rejected after weighing the alternatives. Cast receiver
emulation is fragile (no maintained open-source AP2-grade Cast receiver; Google
periodically breaks the unofficial ones) and still requires a separate path to
the LP10. AirPlay 2 is supported natively on both endpoints and offers tight
multi-room sync, so the design uses AirPlay 2 throughout.

## Architecture

```
                    ┌────────────────────────────────────┐
                    │  Home Assistant + Music Assistant  │
                    │  + Mosquitto add-on (on ODROID)    │
                    └────┬─────────────────────────┬─────┘
                         │ AirPlay 2 (sender)      ▲ MQTT (subscribe)
                  ┌──────┴────────┐                │
                  ▼               ▼                │
   ┌─────────────────┐  ┌──────────────────┐       │
   │ shairport-sync  │  │   Arylic LP10    │       │
   │  (Docker, NAS)  │  │   (AV rack)      │       │
   │  AP2 + nqptp    │  └────────┬─────────┘       │
   │  → Pulse socket │           │ analog out      │
   │  → MQTT events  ├───────────┼─────────────────┘
   └────────┬────────┘           ▼
            │ unix socket   ┌─────────┐
            ▼               │  amp    │
   ┌─────────────────┐      └────┬────┘
   │ LedFx (Docker)  │           │
   │ Pulse server    │           ▼
   │ FFT → DDP       │     bathroom speaker
   └────────┬────────┘
            │ DDP (UDP/4048)
            ▼
        WLED (bathroom)

   ┌─────────────────────────────┐
   │ shairport-config-render     │   one-shot init container.
   │  (Docker, NAS, exits)       │   Renders shairport-sync.conf
   │  bash + render-config.sh    │   from .env into shared volume.
   └─────────────────────────────┘
```

Three Docker containers on the NAS. **shairport-config-render** is a
one-shot init container that runs at each `docker compose up`: it reads
env vars from `.env`, renders `shairport-sync.conf` to a named volume, and
exits. **shairport-sync** depends on it (`service_completed_successfully`)
and reads the rendered config. **LedFx** hosts a PulseAudio server in
server mode and exposes its socket via a bind-mount. shairport-sync writes
audio into that socket and (when MQTT is enabled) publishes session
events to a Mosquitto broker that Home Assistant subscribes to.

## Components

### LedFx (audio-reactive engine)

- Image: `ghcr.io/ledfx/ledfx:v2.1.8`
- Runs in PulseAudio **server mode** (a Pulse instance lives inside the
  container). The official LedFx Docker docs document this pattern.
- Exposes the Pulse socket to siblings via a bind-mount of
  `~/ledfx/pulse:/home/ledfx/.config/pulse` (UID:GID 1000:1000, rw).
- `network_mode: host` — required for WLED DDP and mDNS discovery.
- Web UI on `http://<nas>:8888`.
- Sends DDP (UDP/4048) to the WLED device by IP. DDP is preferred over E1.31
  for single-controller setups: unicast, low overhead.

### shairport-sync (AirPlay 2 receiver)

- Image: `mikebrady/shairport-sync:5.0.4`
- The `:latest` / non-`-classic` tag bundles nqptp internally and is built
  with `--with-mqtt-client`; AP2 mode and MQTT publishing work out of the
  box. `:latest-classic` is AP1 only — do not use.
- `network_mode: host` — required for AirPlay 2 (mDNS, PTP on UDP/319-320).
- `cap_add: [SYS_NICE]` — needed for nqptp scheduling priority.
- Connects to LedFx's Pulse socket via bind-mount.
- Reads its config from a named volume (`shairport-config`) populated by
  the render container; runs with `-c /cfg/shairport-sync.conf`.
- All operator-tunable settings (`AIRPLAY_NAME`, MQTT broker, etc.) come
  from `.env`. Fixed settings (PA backend, `ignore_volume_control`) live
  in the render script.

### shairport-config-render (init container)

- Image: `bash:5.2-alpine3.22`
- One-shot: runs at each `docker compose up`, executes
  `shairport-sync/render-config.sh`, writes `shairport-sync.conf` to the
  `shairport-config` named volume, and exits.
- Why it exists: shairport-sync's libconfig format has no env-var
  substitution and conditional MQTT-auth lines need real shell logic. A
  small init container keeps `docker compose up` as the single deploy
  command (no host-side `make config` step).
- `restart: "no"` — failure here blocks `shairport` (which `depends_on`
  it with `service_completed_successfully`), so a misconfigured `.env`
  fails fast at deploy time rather than producing a malformed config.

### Music Assistant (existing, no changes to deployment)

- 2.8.6 already running on the HA box.
- Adds the LP10 via the AirPlay provider (already discoverable on LAN).
- Discovers `Bathroom-Sync` automatically via mDNS once shairport-sync is up.
- A user-defined sync group "Bathroom Reactive" pairs them.

## Audio data flow

1. MA decodes the source (Spotify, local file, etc.).
2. MA streams over AirPlay 2 to both members of the sync group: LP10 and
   shairport-sync.
3. LP10 → amp → speaker. shairport-sync → Pulse → LedFx.
4. LedFx samples its default Pulse source, runs FFT (size 2048 default), maps
   to its active effect, and emits DDP frames to the WLED IP at the effect's
   configured rate (typically 60 fps).

Sample rate: shairport-sync delivers 44.1 kHz / 16-bit PCM (AirPlay native).
LedFx accepts this directly.

## Playback paths

The all-AirPlay design supports **two independent ways to drive playback**:

1. **Via Music Assistant** (HA automations, voice via HA assistant, MA UI,
   playlists). MA addresses its own sync group "Bathroom Reactive."
2. **Direct AirPlay from any sender** (iPhone/iPad/Mac, Spotify, Apple Music,
   browser AirPlay, Apple TV). Both endpoints are added to a group in the
   Apple Home app; the group then appears in every AirPlay picker on the LAN.

These mechanisms are independent and can both be used. **Caveat:** AirPlay
groups are Apple-only on the sender side; Android phones and Google Home
speakers cannot target them. The user has accepted this trade-off.

## Home Assistant integration via MQTT

shairport-sync's built-in MQTT publisher emits AirPlay session events and
per-track metadata to a Mosquitto broker. Combined with HA-MQTT
autodiscovery (`enable_autodiscovery = "yes"`), a `media_player` entity
appears in HA automatically — no manual `configuration.yaml` work.

**Topics published** (under `<MQTT_TOPIC>/...`, defaulting to `<AIRPLAY_NAME>/...`):

| Topic | When |
|---|---|
| `play_start` | An AirPlay session begins |
| `play_end` | An AirPlay session ends |
| `play_flush` | Sender flushes the buffer (seek/skip) |
| `play_resume` | Sender resumes after pause |
| `title`, `artist`, `album`, `genre`, `format`, `songalbum`, `volume`, `client_ip` | Per-track metadata at session start |

These are HA automation triggers (`platform: mqtt`, `topic:
Bathroom-Sync/play_start`). Source-agnostic — fires for any AirPlay
sender (Music Assistant, iPhone, Mac, Spotify, Apple Music, browser).

**Toggle:** `MQTT_ENABLED=no` in `.env` omits the entire `mqtt = {…}`
block from the rendered config and removes the `MQTT_HOST` requirement.
The default is `yes`.

**Broker placement:** the HA Mosquitto add-on lives inside HAOS. From
the QNAP (a separate host), reach it via the HA host's LAN
hostname/IP (e.g., `homeassistant.local:1883`), not the in-HAOS Docker
name `core-mosquitto`.

## Sync characteristics

- Two AirPlay 2 endpoints in a sync group use AirPlay 2's native multi-room
  sync (PTP-based, ~10–50 ms tolerance) when grouped via Apple Home.
- When grouped via MA, the airplay provider streams to each independently;
  drift is typically <100 ms.
- Both are perceptually fine for visualization. Light-vs-sound offsets in this
  range are not noticeable to humans (films routinely run ~40 ms off).

## Networking

- All containers: `network_mode: host`. This is required by both shairport-sync
  (PTP, mDNS) and LedFx (DDP, WLED discovery), and is the simplest path on
  QNAP Container Station.
- No inbound ports from the internet. LedFx web UI (`:8888`) is LAN-only.
- The NAS, ODROID, LP10, and WLED are all on the same flat LAN — no routing
  or mDNS reflector needed.

## Conflict checks

- **PTP ports 319/320:** nqptp inside shairport-sync needs exclusive access.
  QTS does not run a PTP daemon. Other consumers on the same host would be:
  another shairport-sync instance, another nqptp, or a HomeKit hub that does
  PTP. Verify none are present before deploying.
- **mDNS service collisions:** shairport-sync advertises `_raop._tcp` and
  `_airplay._tcp`. The `general.name` setting must be unique on the LAN.
- **PulseAudio socket ownership:** LedFx runs as UID 1000 and writes the
  Pulse socket into the bind-mounted `volumes/ledfx-pulse/` host dir. The
  dir must be `chown 1000:1000` before first boot or LedFx can't create
  the socket. (Authentication is anonymous — LedFx's bundled Pulse uses
  `auth-anonymous=1`, so no cookie needs to be shared between containers.)

## Configuration touchpoints

Files the user will edit during install:

1. `.env` (copied from `.env.example`) — single source of truth for
   operator settings: `AIRPLAY_NAME`, `MQTT_*`, `MQTT_ENABLED`, image
   tags. The render container picks up `AIRPLAY_NAME` and `MQTT_*`;
   compose's auto-`.env` discovery picks up image tags and anything used
   in the YAML's `${VAR}` substitutions. The WLED IP is configured
   directly in the LedFx web UI, not via `.env`.
2. `shairport-sync/render-config.sh` — the source of truth for *fixed*
   shairport-sync settings (PA backend, `ignore_volume_control`,
   `publish_parsed`, autodiscovery prefix). Edit only when you want to
   change behavior that isn't `.env`-tunable.
3. `docker-compose.yml` — three services (`ledfx`, `shairport`,
   `shairport-config-render`). Edit only to change image versions, add
   another LedFx instance for another room, etc.
4. LedFx web UI / `PUT /api/audio/devices` — select the Pulse default source
   as the active audio device.
5. Music Assistant (HA UI) — add LP10 (AirPlay), confirm Bathroom-Sync was
   auto-discovered, create sync group "Bathroom Reactive."
6. Apple Home app (optional) — add Bathroom-Sync as an AirPlay 2 device,
   then create an AirPlay group containing it and the LP10. Enables direct
   AirPlay from any Apple device.
7. Home Assistant (when `MQTT_ENABLED=yes`) — verify the autodiscovered
   `media_player.bathroom_sync` entity appears under MQTT, then write
   automations against `Bathroom-Sync/play_start` / `play_end` topics.

## Failure modes

| Failure | Behavior | Recovery |
|---|---|---|
| LedFx container dies | Pulse socket gone; shairport-sync errors | `restart: unless-stopped` on both; shairport-sync reconnects when Pulse returns |
| shairport-sync dies | AirPlay receiver disappears; LP10 still plays | Auto-restart; reappears on LAN within ~30s |
| WLED offline | LedFx logs and keeps running | Reconnects when WLED returns |
| LP10 unreachable | MA marks it offline; group plays only on Bathroom-Sync (LEDs react, no sound) | Power-cycle LP10 |
| MA unreachable | No audio source for either path; direct AirPlay still works | HA/MA recovery |
| NAS reboot | Container Station auto-starts compose stack | MA rediscovers Bathroom-Sync via mDNS within ~30s |
| nqptp port conflict | shairport-sync logs "PTP port unavailable" | Identify and disable the conflicting service |
| Render container fails (e.g., `MQTT_HOST` missing while `MQTT_ENABLED=yes`) | `shairport` never starts (`depends_on: service_completed_successfully`) | Fix `.env` or set `MQTT_ENABLED=no`; rerun `docker compose up` |
| MQTT broker unreachable (with `MQTT_ENABLED=yes`) | shairport-sync runs and serves AirPlay normally; logs MQTT connection errors and retries | Bring broker back; no manual intervention needed |

## Testing strategy

In order, gating each step before the next:

1. **Pulse socket reachable.** Bring up LedFx alone. From a shell, exec into
   the LedFx container and run `pactl info` — confirm a default sink exists.
2. **Shairport-sync visible.** Start shairport-sync. Confirm "Bathroom-Sync"
   appears in the AirPlay picker on a Mac/iPhone on the same LAN. AirPlay a
   short clip directly to it.
3. **LedFx reactive.** While step 2 is playing, open the LedFx web UI. Confirm
   the audio meter moves and the active effect responds to the audio.
4. **WLED responsive.** Confirm the bathroom LEDs animate to the audio with
   no visible packet loss.
5. **MA discovery.** In MA, confirm Bathroom-Sync appears under the AirPlay
   provider. Play a track to it solo; verify LEDs react.
6. **MA group.** Create the sync group "Bathroom Reactive" with LP10 +
   Bathroom-Sync. Play a track to the group; confirm both speaker and LEDs
   are in audible/visible sync.
7. **Apple Home group (optional).** Add Bathroom-Sync to Apple Home, group
   with LP10, AirPlay from a phone to the group; same expectations as step 6.
8. **MQTT autodiscovery (when `MQTT_ENABLED=yes`).** Subscribe to
   `Bathroom-Sync/#` on the broker; play and stop a track; confirm
   `play_start` and `play_end` arrive. In HA, confirm the autodiscovered
   `media_player.bathroom_sync` entity appears under the MQTT integration.
9. **Reboot.** Restart the NAS; confirm the stack comes up cleanly and MA
   rediscovers within 60 s.

## Out of scope

- Google Cast / Chromecast group membership. Original idea, dropped after
  user confirmed any-way-to-start-playback was acceptable.
- Replacing the LP10 with a NAS-driven audio path to the amp. Possible but
  requires hardware changes; not warranted given the current setup works.
- Multi-room expansion (other rooms with reactive lighting). Architecture
  scales — add another shairport-sync container with a different name and
  another LedFx instance — but is not part of this spec.
- Custom WLED effects. We use stock LedFx effects.

## Component versions (pinned)

| Component | Version | Released |
|---|---|---|
| shairport-sync | 5.0.4 | 2026-04-27 |
| nqptp (bundled in shairport-sync image) | 1.2.x | per image build |
| LedFx | 2.1.8 | 2026-04-21 |
| Music Assistant | 2.8.6 | 2026-04-23 |
| bash (render init container) | 5.2-alpine3.22 | upstream Docker official |

The shairport-sync 5.0.4 image was built before nqptp 1.2.7 released; the
bundled nqptp version is whatever was current when the image was built. This
is unlikely to affect AP2 sync quality — verified in testing if any issues.

## References

- LedFx Docker docs (Pulse server mode + shairport-sync sibling pattern):
  https://github.com/ledfx/ledfx/blob/main/docs/installing.md
- shairport-sync Docker docs:
  https://github.com/mikebrady/shairport-sync/tree/master/docker
- shairport-sync AIRPLAY2.md (PTP port requirements):
  https://github.com/mikebrady/shairport-sync/blob/master/AIRPLAY2.md
- Music Assistant docs (player providers, sync groups):
  https://music-assistant.io/

## Revision history

- **2026-05-07** — Initial design approved. Two containers (LedFx,
  shairport-sync) with static `shairport-sync.conf`.
- **2026-05-08** — Added `.env`-driven config rendering via a
  `shairport-config-render` init container; added MQTT publisher with HA
  autodiscovery and an `MQTT_ENABLED` toggle. Architecture, components,
  configuration touchpoints, failure modes, testing strategy, and
  component versions updated accordingly.
- **2026-05-08 (audit follow-up)** — Bumped render image off EOL Alpine
  3.19 to `bash:5.2-alpine3.22`. Removed `WLED_IP` and `NAS_HOST` from
  `.env.example` (informational-only, not consumed by the stack).
  Removed vestigial `PULSE_COOKIE` from compose. Corrected Pulse-cookie
  troubleshooting guidance to reflect LedFx's anonymous-Pulse model.
  Documented MQTT credential storage. Deleted the now-stale
  implementation plan; README + spec are the canonical docs.
