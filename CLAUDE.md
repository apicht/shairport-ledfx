# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A 3-container Docker Compose stack (2 long-running + 1 init) that exposes an AirPlay 2 receiver whose audio is fed to LedFx for real-time WLED audio-reactive lighting. shairport-sync also publishes AirPlay session events over MQTT for Home Assistant automations.

There is no application code — every artifact is config, docs, or a small render script. The "build" is `docker compose up`. Target host is a Linux box (developed against QNAP Container Station). Apache-2.0.

The directory is currently named `chromecast-ledfx` for historical reasons; the project will be renamed to `shairport-ledfx` before publishing. Don't rename it yourself — wait for explicit user direction.

## Common commands

These run on a Linux host. **Docker Desktop on macOS/Windows is not a valid deploy target** — `network_mode: host` doesn't behave correctly there. You can validate compose syntax on macOS, but actual `up` must happen on Linux.

```bash
# Validate the compose file resolves correctly with .env values
cp .env.example .env && docker compose config --quiet && rm .env

# Test the shairport-sync.conf render in isolation (no compose needed)
docker run --rm -v "$(pwd)/shairport-sync:/scripts:ro" \
  -e AIRPLAY_NAME=Test -e MQTT_ENABLED=no \
  bash:5.2-alpine3.22 bash /scripts/render-config.sh /dev/stdout

# Bring up the full stack (on the Linux deploy host)
docker compose up -d

# Read what each container is doing
docker logs ledfx
docker logs shairport
docker logs shairport-config-render   # exits 0 after rendering; logs show the rendered path

# Inspect runtime audio state inside LedFx
docker exec ledfx pactl info
docker exec ledfx pactl list short sink-inputs
```

There is no test suite, no linter, no build step.

## Architecture (things that require reading multiple files)

**The PulseAudio handoff between LedFx and shairport-sync.** LedFx runs PulseAudio in server mode inside its container and exposes the unix socket via bind-mount of `./volumes/ledfx-pulse:/home/ledfx/.config/pulse`. The shairport-sync container mounts the *same host dir* at `/tmp` and connects via `PULSE_SERVER=unix:/tmp/pulseaudio.socket`. **LedFx's Pulse is configured `auth-anonymous=1`** in the upstream image's `start.sh`, so no cookie sharing is required — ownership of the bind-mount dir (must be UID 1000) is what matters. Earlier docs incorrectly claimed cookies were needed; don't reintroduce that misconception.

**The config render flow.** `shairport-sync.conf` does **not** exist as a static file in the repo. It's rendered at each `docker compose up` by a one-shot init container (`shairport-config-render`, image `bash:5.2-alpine3.22`) that runs `shairport-sync/render-config.sh`. The script reads env vars from compose, escapes them for libconfig, conditionally emits the `mqtt = {…}` block based on `MQTT_ENABLED`, and writes the result to a named volume (`shairport-config`). The `shairport` service `depends_on: shairport-config-render` with `condition: service_completed_successfully` and reads the rendered file via `command: ["-c", "/cfg/shairport-sync.conf"]`. If you grep for "shairport-sync.conf" expecting a checked-in file, you won't find one.

**Why `network_mode: host` everywhere and what it costs.** Required by shairport-sync (AirPlay 2 needs PTP on UDP/319-320 + mDNS) and by LedFx (DDP unicast to WLED + WLED auto-discovery). This means the stack cannot be simply re-deployed inside Kubernetes or anywhere else without bridge → host translation. It also means port collisions on the host are real — only one nqptp/PTP-using process per host. Documented in the spec under "Conflict checks."

**Env var flow has two levels.** Compose's automatic `.env` discovery handles `${VAR}` substitution **inside the YAML** (image tags, the `${VAR:?required}` checks under `environment:`). Each substituted value then becomes a literal in the `environment:` map, which Docker injects into the container's process. We deliberately do **not** use `env_file:` because (a) it bypasses compose's required-var checks and (b) it would leak compose-only vars (image tags, etc.) into containers indiscriminately. Don't switch without understanding why.

**shairport-sync 5.x with bundled nqptp.** The `mikebrady/shairport-sync:latest` (and `:5.0.4`) image bundles `nqptp` internally and is built with `--with-mqtt-client`. Don't add a separate `nqptp` sidecar — earlier design iterations had one, and it was removed when we verified the upstream image bundles it. Also: never use the `-classic` tag (AirPlay 1 only).

## Canonical docs

- `README.md` — user-facing.
- `docs/superpowers/specs/2026-05-07-airplay-ledfx-bathroom-design.md` — architecture, rationale, revision history. Read this before making non-trivial changes; the **revision history at the bottom** explains why the design evolved (e.g., why the render container exists).

The implementation plan that used to live at `docs/superpowers/plans/` was deleted as part of audit follow-up — README + spec are the canonical pair. Don't recreate the plan unless you've discussed it with the user.

## Gotchas worth remembering

- The `volumes/ledfx-pulse/` host dir must be `chown 1000:1000` before first `compose up` — LedFx writes the Pulse socket there as UID 1000. The `.gitkeep` placeholder in that dir gets clobbered by LedFx's `start.sh` on first run; this is expected and the `.gitignore` accommodates it.
- `MQTT_PASSWORD` is rendered as plaintext into the libconfig file inside the `shairport-config` Docker volume, and is also visible in `docker inspect shairport-config-render`. libconfig has no secret-store mechanism, so this is structural — document, don't fix.
- `.env.example` only contains vars the stack actually reads. Don't add deploy-helper vars (like SSH hostnames) — those belong in shell or README examples.
- The `shairport-config` named volume is regenerated on every `up`. Don't put any user-managed state in it.
