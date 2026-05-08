# shairport-ledfx

Audio-reactive WLED lighting in any room, driven by AirPlay 2.

A two-container Docker stack that exposes an AirPlay 2 receiver and pipes its
audio into [LedFx](https://github.com/LedFx/LedFx) for real-time WLED
visualization. Designed to run alongside an existing networked speaker
(e.g., Arylic, HomePod, Sonos) so audio plays on the speaker AND drives the
LEDs in sync.

## Why

LedFx wants a local audio source. Most "Cast to LedFx" projects try to
emulate a Chromecast receiver — a fragile path, since Google Cast's
receiver-side protocol is closed and unofficial emulators get broken
periodically.

This project takes the opposite approach: stand up an **AirPlay 2 receiver**
([shairport-sync](https://github.com/mikebrady/shairport-sync)) as a sibling
container to LedFx, sharing a PulseAudio socket. Anything that AirPlays —
Music Assistant, iOS Control Center, macOS, Spotify, Apple Music, browser
AirPlay — can target it. Group it with another AirPlay 2 speaker (in
Apple Home or Music Assistant) and the same stream drives both audio and
LEDs.

## Architecture

```
                    ┌────────────────────────────────────┐
                    │  AirPlay sender                    │
                    │  (Music Assistant, iPhone, Mac …)  │
                    └─────────────────┬──────────────────┘
                                      │ AirPlay 2
                  ┌───────────────────┴───────────────────┐
                  ▼                                       ▼
       ┌─────────────────────┐                 ┌──────────────────────┐
       │ shairport-sync      │                 │  Existing AirPlay 2  │
       │  (Docker container) │                 │  speaker (LP10,      │
       │  AP2 + nqptp        │                 │  HomePod, Sonos …)   │
       │  → Pulse socket     │                 └──────────┬───────────┘
       └──────────┬──────────┘                            │ amp/output
                  │ unix socket                           ▼
                  ▼                                 physical speaker
       ┌─────────────────────┐
       │ LedFx               │
       │  (Docker container) │
       │  Pulse server mode  │
       │  FFT → DDP          │
       └──────────┬──────────┘
                  │ DDP (UDP/4048)
                  ▼
              WLED controller → LED strip
```

Two containers on the same host. LedFx hosts a PulseAudio server in server
mode and exposes its socket via a bind-mount. shairport-sync writes audio
into that socket. LedFx samples its default Pulse source, runs FFT, and
emits DDP frames to the WLED device.

Both containers run with `network_mode: host` because:
- shairport-sync needs UDP 319/320 for PTP (AirPlay 2 timing) and mDNS
  to advertise on the LAN.
- LedFx needs unfettered access to the LAN for DDP and WLED discovery.

## Components and pinned versions

| Component | Version | Notes |
|---|---|---|
| [shairport-sync](https://github.com/mikebrady/shairport-sync) | `5.0.4` | AirPlay 2 receiver. The `:latest` (non-`-classic`) tag bundles `nqptp` internally. |
| [LedFx](https://github.com/LedFx/LedFx) | `2.1.8` | Audio-reactive engine. Runs PulseAudio in server mode in the official Docker image. |
| [Music Assistant](https://music-assistant.io/) | `2.8.6` | Optional. Used as a sender to drive sync groups. |

Override image tags in `.env` to upgrade.

## Requirements

- A Linux host with Docker and Docker Compose v2 (designed and tested for
  QNAP Container Station; works on any Linux box).
- WLED-flashed controller on the same LAN.
- An existing networked speaker for the audio output (or just the LEDs if
  you don't care about audible playback).
- UDP ports 319/320 free on the host. No other PTP daemon, HomeKit hub,
  or shairport-sync instance binding them.
- An AirPlay name unique on your LAN (default: `Bathroom-Sync`; set
  `AIRPLAY_NAME` in `.env`).

`network_mode: host` is required, which means **Docker Desktop on macOS
and Windows is not supported as a deployment target.** Edit on any
workstation, deploy on Linux.

## Quick start

The stack must run on a Linux host (it requires `network_mode: host`).
Deploy directly on that host, or develop on a workstation and copy the
repo over via `rsync`/`ssh`.

```bash
# 1. Configure environment
cp .env.example .env
$EDITOR .env       # at minimum, set AIRPLAY_NAME and MQTT_HOST
                   # (or set MQTT_ENABLED=no to skip MQTT entirely)

# 2. Prepare the host volumes (must be writable by UID 1000 — LedFx's user)
mkdir -p volumes/ledfx-pulse volumes/ledfx-config
sudo chown -R 1000:1000 volumes/ledfx-pulse volumes/ledfx-config

# 3. Bring up the stack
docker compose up -d

# 4. Verify
docker compose ps
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8888/   # LedFx web UI
```

### Remote deploy (workstation → Linux host over SSH)

If your workstation isn't the deploy target, replace `<nas>` with the
target host's address:

```bash
rsync -avz --exclude=.git --exclude=volumes ./ <nas>:~/shairport-ledfx/
rsync -avz .env <nas>:~/shairport-ledfx/.env
ssh <nas> 'mkdir -p ~/shairport-ledfx/volumes/ledfx-pulse ~/shairport-ledfx/volumes/ledfx-config && \
           sudo chown -R 1000:1000 ~/shairport-ledfx/volumes/ledfx-pulse ~/shairport-ledfx/volumes/ledfx-config && \
           cd ~/shairport-ledfx && docker compose up -d'
```

After the stack is up, configure LedFx (audio source + WLED device) via
the web UI on port `8888`. See [Configuration → LedFx](#ledfx) below.

## Configuration

### `.env`

`.env.example` has every variable consumed by the stack with inline
documentation. The required ones are:

- `AIRPLAY_NAME` — visible name in AirPlay pickers (default `Bathroom-Sync`).
- `MQTT_HOST` — required only when `MQTT_ENABLED=yes` (the default). Point
  it at your Mosquitto broker.

Image tags (`LEDFX_IMAGE`, `SHAIRPORT_IMAGE`, `RENDER_IMAGE`) are also
in `.env` so you can upgrade by editing one file.

### `shairport-sync/render-config.sh`

The shairport-sync config is rendered from `.env` at each `docker compose up`
by a one-shot init container (`shairport-config-render`). The render
script lives at `shairport-sync/render-config.sh` and emits a libconfig
file to a named volume (`shairport-config`) which the shairport service
reads from.

Edit `.env` to change config — not the rendered file. The defaults are:

- `AIRPLAY_NAME` — visible name in AirPlay pickers, MA, and Apple Home.
- `output_backend = "pulseaudio"` — fixed in the render script. Points at
  the LedFx-managed Unix socket via the shared bind-mount.
- `ignore_volume_control = "yes"` — fixed in the render script. Keeps Pulse
  loopback at unity gain so LedFx FFT sees consistent levels regardless of
  sender volume. Volume control still happens at the speaker side.

### MQTT publisher (Home Assistant integration)

shairport-sync publishes AirPlay session events and track metadata to MQTT,
with built-in HA-autodiscovery so a `media_player` entity appears in HA
automatically. Set the `MQTT_*` vars in `.env` to point at your broker
(typically the HA Mosquitto add-on).

Toggle the publisher with `MQTT_ENABLED` in `.env`. Set to `no` (or
`false`/`0`/`off`) to skip the entire mqtt block in the rendered config —
no broker required, no autodiscovery published. Default is `yes`.

Topics published under `<topic>/...` (where `<topic>` is `MQTT_TOPIC` if set,
otherwise `AIRPLAY_NAME`):

| Topic | When |
|---|---|
| `play_start` | An AirPlay session begins |
| `play_end` | An AirPlay session ends |
| `play_flush` | Sender flushes the audio buffer (e.g., seek/skip) |
| `play_resume` | Sender resumes after pause |
| `title`, `artist`, `album`, `genre`, `format`, `songalbum`, `volume`, `client_ip` | Per-track metadata at session start |

Use these as automation triggers in HA. Example:

```yaml
# configuration.yaml or an automation
automation:
  - alias: "Lights on when bathroom AirPlay starts"
    trigger:
      - platform: mqtt
        topic: "Bathroom-Sync/play_start"
    action:
      - service: light.turn_on
        target: { entity_id: light.bathroom }
```

Replace `Bathroom-Sync` with your `AIRPLAY_NAME` (or your `MQTT_TOPIC`
override).

**Credential storage note.** `MQTT_PASSWORD` from `.env` is rendered into
the libconfig file in plaintext (libconfig has no secret-store mechanism)
inside the `shairport-config` Docker volume, and is also visible in
`docker inspect shairport-config-render`. Both are local-host artifacts
on the deploy machine — not exposed to the network — but if you're on a
shared/multi-tenant Docker host, scope the broker user accordingly. For
HA Mosquitto: create a dedicated user with publish-only ACLs to the
`<AIRPLAY_NAME>/#` and `homeassistant/#` topic spaces rather than
reusing your admin user.

### LedFx

LedFx state lives in a host bind-mount at `./volumes/ledfx-config/`
(must be `chown 1000:1000` so LedFx, running as UID 1000, can write its
log and config). State persists across container restarts and across
`docker compose down/up` cycles. First-time setup is via the web UI at
`http://<host>:8888`:

1. Audio device → select the Pulse default source (also scriptable via
   `PUT /api/audio/devices`).
2. Devices → Add Device → WLED → enter the controller's IP.
3. Virtuals → assign the device → pick an audio-reactive effect.

## Playback paths

The stack supports two independent playback paths:

1. **Music Assistant (or any AirPlay 1/2 sender directly).** Bathroom-Sync
   is just an AirPlay 2 receiver on the LAN. Anything that AirPlays can
   target it, MA can group it with other AirPlay players for synchronized
   playback.
2. **Apple Home AirPlay group.** Pair Bathroom-Sync in Apple Home, group
   it with another AirPlay 2 speaker, and the group appears as a single
   destination in every AirPlay picker on every Apple device on the LAN.

AirPlay groups are Apple-only on the sender side — Android and Google Home
cannot target them.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `shairport` logs `port unavailable` on 319/320 | Another nqptp/PTP service on the host | `sudo ss -lun \| grep -E ":(319\|320)"`, identify, disable |
| `shairport` can't connect to PulseAudio socket | Bind-mount permissions wrong (LedFx runs as UID 1000) | `sudo chown -R 1000:1000 volumes/ledfx-pulse` and restart the stack. LedFx's bundled Pulse is anonymous (`auth-anonymous=1`), so no cookie sharing is needed — only the socket dir's ownership. |
| LedFx audio meters flat | Wrong audio device, sink-input not landing | `docker exec ledfx pactl list short sink-inputs`; reselect device |
| LEDs unresponsive but meters work | WLED DDP receive disabled | WLED → Sync Interfaces → enable DDP receive (port 4048) |
| Bathroom-Sync vanishes from LAN | shairport crashed, mDNS confused | `docker logs shairport`; `docker compose restart shairport` |
| `network_mode: host` errors during `compose up` | Running on Docker Desktop (macOS/Windows) | Deploy on a Linux host |

## Project layout

```
.env.example                          # AIRPLAY_NAME, MQTT_*, image tags
docker-compose.yml                    # ledfx + shairport + render init
shairport-sync/render-config.sh       # renders shairport-sync.conf from .env
volumes/ledfx-pulse/                  # host dir for shared Pulse socket
docs/superpowers/specs/               # design rationale (why this shape)
docs/superpowers/plans/               # step-by-step deployment plan
```

## License

Apache License 2.0. See [`LICENSE`](LICENSE).

This repository contains only orchestration (compose file, configs, docs).
Each component pulled at deploy time remains under its own upstream license:
shairport-sync (MIT), nqptp (GPL-2.0), LedFx (GPL-3.0), Music Assistant
(Apache-2.0), WLED firmware (EUPL-1.2).
