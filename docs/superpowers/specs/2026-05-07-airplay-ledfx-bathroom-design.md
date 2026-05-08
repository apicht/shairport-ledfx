# Audio-Reactive WLED in the Bathroom — Design

Date: 2026-05-07
Status: Draft pending user review

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
                    │       (existing, on ODROID)        │
                    └─────────────────┬──────────────────┘
                                      │ AirPlay 2 (sender)
                  ┌───────────────────┼───────────────────┐
                  ▼                                       ▼
       ┌─────────────────────┐                 ┌────────────────────┐
       │ shairport-sync      │                 │   Arylic LP10      │
       │  (Docker, NAS)      │                 │   (AV rack)        │
       │  AP2 + nqptp        │                 └─────────┬──────────┘
       │  → Pulse socket     │                           │ analog out
       └──────────┬──────────┘                           ▼
                  │ unix socket                  ┌──────────────┐
                  ▼                              │ Multi-zone   │
       ┌─────────────────────┐                   │   amp        │
       │ LedFx (Docker, NAS) │                   └──────┬───────┘
       │ Pulse server mode   │                          │
       │ FFT → DDP           │                          ▼
       └──────────┬──────────┘                   bathroom speaker
                  │ DDP (UDP/4048)
                  ▼
              WLED (bathroom)
```

Two Docker containers on the NAS share the LedFx-hosted PulseAudio socket via
a bind-mount. shairport-sync writes audio into the socket; LedFx reads its
default Pulse source, runs FFT, and pushes effects to WLED.

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
- The `:latest` / non-`-classic` tag bundles nqptp internally; AP2 mode works
  out of the box. `:latest-classic` is AP1 only — do not use.
- `network_mode: host` — required for AirPlay 2 (mDNS, PTP on UDP/319-320).
- `cap_add: [SYS_NICE]` — needed for nqptp scheduling priority.
- PulseAudio backend: `command: -o pulseaudio` (or set in `shairport-sync.conf`).
- Connects to LedFx's Pulse socket via bind-mount.
- Configuration file (`shairport-sync.conf`) sets:
  - `general.name = "Bathroom-Sync"` — visible name in AirPlay pickers and MA
  - `general.ignore_volume_control = "yes"` — keeps Pulse loopback at unity so
    LedFx FFT sees consistent levels regardless of sender volume
  - `general.output_backend = "pulseaudio"`
  - HomeKit/AP2 config block enabled (default)

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
- **PulseAudio cookie:** LedFx generates a Pulse cookie on first run. The
  shairport-sync container must mount the same cookie path so authentication
  succeeds. Documented in the LedFx Docker docs.

## Configuration touchpoints

Files the user will edit during install:

1. `docker-compose.yml` — two services (`ledfx`, `shairport`) on host
   networking, with the shared Pulse bind-mount.
2. `shairport-sync.conf` — set `name`, output backend, ignore volume.
3. LedFx web UI / `PUT /api/audio/devices` — select the Pulse default source
   as the active audio device.
4. Music Assistant (HA UI) — add LP10 (AirPlay), confirm Bathroom-Sync was
   auto-discovered, create sync group "Bathroom Reactive."
5. Apple Home app (optional) — add Bathroom-Sync as an AirPlay 2 device, then
   create an AirPlay group containing it and the LP10. Enables direct AirPlay
   from any Apple device.

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
8. **Reboot.** Restart the NAS; confirm the stack comes up cleanly and MA
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
