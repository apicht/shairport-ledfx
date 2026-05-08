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
- An AirPlay name unique on your LAN (default: `Bathroom-Sync`; change in
  `shairport-sync/shairport-sync.conf`).

`network_mode: host` is required, which means **Docker Desktop on macOS
and Windows is not supported as a deployment target.** Edit on any
workstation, deploy on Linux.

## Quick start

```bash
# 1. Configure environment
cp .env.example .env
$EDITOR .env                           # set WLED_IP, NAS_HOST
set -a; source .env; set +a

# 2. (If deploying remotely) sync to host and prepare the bind-mount
rsync -avz --exclude=.git --exclude=volumes ./ ${NAS_HOST}:~/shairport-ledfx/
ssh ${NAS_HOST} 'mkdir -p ~/shairport-ledfx/volumes/ledfx-pulse && \
                 sudo chown -R 1000:1000 ~/shairport-ledfx/volumes/ledfx-pulse'
rsync -avz .env ${NAS_HOST}:~/shairport-ledfx/.env

# 3. Bring up the stack
ssh ${NAS_HOST} 'cd ~/shairport-ledfx && docker compose up -d'

# 4. Verify
ssh ${NAS_HOST} 'docker compose -f ~/shairport-ledfx/docker-compose.yml ps'
curl -s -o /dev/null -w "%{http_code}\n" http://${NAS_HOST}:8888/   # LedFx web UI
```

If deploying directly on the host (no rsync), step 2 becomes:
```bash
mkdir -p volumes/ledfx-pulse
sudo chown -R 1000:1000 volumes/ledfx-pulse
docker compose up -d
```

For the full step-by-step (including pre-flight conflict checks, AirPlay
discovery test, LedFx audio device selection, WLED setup, and Music
Assistant / Apple Home grouping), see
[`docs/superpowers/plans/2026-05-07-airplay-ledfx-bathroom.md`](docs/superpowers/plans/2026-05-07-airplay-ledfx-bathroom.md).

## Configuration

### `.env`

```
WLED_IP=192.168.1.50
NAS_HOST=nas.local
LEDFX_IMAGE=ghcr.io/ledfx/ledfx:v2.1.8
SHAIRPORT_IMAGE=mikebrady/shairport-sync:5.0.4
```

### `shairport-sync/shairport-sync.conf`

The defaults are tuned for this stack:

- `name = "Bathroom-Sync"` — change to anything unique on your LAN.
- `ignore_volume_control = "yes"` — keeps Pulse loopback at unity gain so
  LedFx FFT sees consistent levels regardless of sender volume. Volume
  control still happens at the speaker side.
- `output_backend = "pulseaudio"` and a Unix-socket pointer at
  `/tmp/pulseaudio.socket` (which is the LedFx-managed socket via the
  shared bind-mount).

### LedFx

LedFx state lives in a Docker named volume (`ledfx-config`), persisted
across container restarts. First-time setup is via the web UI at
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
| `shairport` logs PulseAudio auth error | Cookie not shared between containers | Verify `volumes/ledfx-pulse/cookie` exists and is readable by UID 1000 |
| LedFx audio meters flat | Wrong audio device, sink-input not landing | `docker exec ledfx pactl list short sink-inputs`; reselect device |
| LEDs unresponsive but meters work | WLED DDP receive disabled | WLED → Sync Interfaces → enable DDP receive (port 4048) |
| Bathroom-Sync vanishes from LAN | shairport crashed, mDNS confused | `docker logs shairport`; `docker compose restart shairport` |
| `network_mode: host` errors during `compose up` | Running on Docker Desktop (macOS/Windows) | Deploy on a Linux host |

## Project layout

```
.env.example                          # WLED_IP / NAS_HOST / image tags
docker-compose.yml                    # ledfx + shairport on host networking
shairport-sync/shairport-sync.conf    # AP2 receiver config
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
