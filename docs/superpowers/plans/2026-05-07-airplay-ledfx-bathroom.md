# AirPlay-Driven Audio-Reactive WLED — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy a 2-container Docker stack on the QNAP NAS that exposes an AirPlay 2 receiver named "Bathroom-Sync" whose audio is fed into LedFx for real-time WLED visualization in the bathroom, while the same Music Assistant stream also plays on the Arylic LP10 → bathroom speaker.

**Architecture:** LedFx runs in PulseAudio server mode. shairport-sync (with bundled nqptp) joins as an AirPlay 2 receiver and writes audio into LedFx's Pulse socket via a shared bind-mount. Music Assistant groups the LP10 + Bathroom-Sync as AirPlay 2 sync-group members. Both containers run with `network_mode: host` for mDNS, PTP (319/320), and DDP to WLED.

**Tech Stack:**
- Docker / Docker Compose (QNAP Container Station)
- `mikebrady/shairport-sync:5.0.4` (AirPlay 2 receiver, nqptp bundled)
- `ghcr.io/ledfx/ledfx:v2.1.8` (LedFx, runs Pulse in server mode)
- Existing: Music Assistant 2.8.6 on ODROID HA host, Arylic LP10, WLED in bathroom

**Spec:** `docs/superpowers/specs/2026-05-07-airplay-ledfx-bathroom-design.md`

**Deployment model:** Author this repo on the Mac. Deploy by syncing the repo to the QNAP and running `docker compose up -d` over SSH. Docker Desktop for Mac does **not** support `network_mode: host`, so all `docker compose` commands run **on the QNAP**. The dev loop is: edit on Mac → `rsync` to NAS → run on NAS.

---

## Pre-flight (one-time)

Before Task 1, confirm these are true. They're not "tasks" because they're environmental.

- [ ] **SSH to the QNAP works.** `ssh <nas>` succeeds.
- [ ] **Docker is available on the QNAP.** `ssh <nas> 'docker --version'` returns a version.
- [ ] **Docker Compose v2 is available.** `ssh <nas> 'docker compose version'` returns v2.x. (If only v1 available, replace `docker compose` with `docker-compose` throughout.)
- [ ] **WLED IP is known.** Note it (e.g., `192.168.1.50`). You'll need it in Task 5.
- [ ] **PTP port collision check.** `ssh <nas> 'sudo ss -lun | grep -E ":(319|320) "'` returns nothing. If anything binds those ports (another nqptp, HomeKit hub), resolve before proceeding — shairport-sync AP2 will fail otherwise.
- [ ] **No conflicting AirPlay name.** Confirm "Bathroom-Sync" isn't already on the LAN. From a Mac: `dns-sd -B _airplay._tcp local.` and check the list. (Ctrl-C to stop.)
- [ ] **`.env` loaded in your shell.** Many commands below reference `${NAS_HOST}` and `${WLED_IP}` from `.env`. After Task 1 creates `.env`, run this once per shell: `set -a; source .env; set +a` (or copy/paste the literal values into commands).

---

## File Structure (what we'll create)

```
chromecast-ledfx/
├── .env.example                       # WLED_IP, NAS_HOST, image tags
├── .gitignore                         # ignore .env, ./volumes/, LedFx config
├── docker-compose.yml                 # ledfx + shairport-sync services
├── shairport-sync/
│   └── shairport-sync.conf            # AP2 receiver config
├── volumes/                           # gitignored runtime state
│   └── ledfx-pulse/                   # shared Pulse socket dir (host)
├── docs/superpowers/
│   ├── specs/2026-05-07-airplay-ledfx-bathroom-design.md   (exists)
│   └── plans/2026-05-07-airplay-ledfx-bathroom.md          (this file)
```

LedFx's user config lives in a Docker named volume (`ledfx-config`). We don't bind-mount it — it's persisted across container restarts via the volume.

---

### Task 1: Bootstrap repository structure

**Files:**
- Create: `.gitignore`
- Create: `.env.example`
- Create: `volumes/ledfx-pulse/.gitkeep`

- [ ] **Step 1: Write `.gitignore`**

```
.env
volumes/
!volumes/ledfx-pulse/.gitkeep
```

- [ ] **Step 2: Write `.env.example`**

```
# AirPlay-LedFx environment configuration.
# Copy to .env and fill in. Do NOT commit .env.

# WLED controller's LAN IP (the bathroom WLED device).
WLED_IP=192.168.1.50

# QNAP NAS SSH host (used only by the dev rsync loop on your workstation).
NAS_HOST=nas.local

# Image tags. Override here to upgrade without editing compose.yml.
LEDFX_IMAGE=ghcr.io/ledfx/ledfx:v2.1.8
SHAIRPORT_IMAGE=mikebrady/shairport-sync:5.0.4
```

- [ ] **Step 3: Create the host volume dir with placeholder**

```bash
mkdir -p volumes/ledfx-pulse
touch volumes/ledfx-pulse/.gitkeep
```

- [ ] **Step 4: Commit**

```bash
git add .gitignore .env.example volumes/ledfx-pulse/.gitkeep
git commit -m "Bootstrap repo: gitignore, env template, volume placeholder"
```

---

### Task 2: LedFx service alone (verify Pulse server works)

**Goal:** Get LedFx running with PulseAudio in server mode and confirm the socket is exposed on the host bind-mount before adding shairport-sync.

**Files:**
- Create: `docker-compose.yml`

- [ ] **Step 1: Write `docker-compose.yml` with only the LedFx service**

```yaml
services:
  ledfx:
    image: ${LEDFX_IMAGE}
    container_name: ledfx
    restart: unless-stopped
    network_mode: host
    volumes:
      - ledfx-config:/home/ledfx/ledfx-config:rw
      - ./volumes/ledfx-pulse:/home/ledfx/.config/pulse:rw
    healthcheck:
      test: ["CMD-SHELL", "test -S /home/ledfx/.config/pulse/pulseaudio.socket || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 6
      start_period: 30s
    logging:
      options:
        max-size: "10m"
        max-file: "5"

volumes:
  ledfx-config:
```

Notes:
- `network_mode: host` is required so DDP packets to WLED, the LedFx web UI port, and mDNS all work without bridge-network NAT.
- The healthcheck waits for the Pulse socket file to appear, which is the precondition shairport-sync needs.
- LedFx's container uses UID/GID 1000:1000; the host bind-mount dir must be writable by that UID.

- [ ] **Step 2: Sync to QNAP and prepare host volume permissions**

```bash
rsync -avz --exclude=.git --exclude=volumes ./ ${NAS_HOST}:~/chromecast-ledfx/
ssh ${NAS_HOST} 'mkdir -p ~/chromecast-ledfx/volumes/ledfx-pulse && sudo chown -R 1000:1000 ~/chromecast-ledfx/volumes/ledfx-pulse'
```

Expected: no errors. If `sudo` requires a password, enter it. If `chown` fails because the user can't sudo, run `chmod 0777 ~/chromecast-ledfx/volumes/ledfx-pulse` as a fallback (less clean but works).

- [ ] **Step 3: Copy `.env` onto the NAS (one-time)**

```bash
cp .env.example .env
# Edit .env locally with the real WLED_IP and NAS_HOST
rsync -avz .env ${NAS_HOST}:~/chromecast-ledfx/.env
```

- [ ] **Step 4: Verify the container fails to start without the bind-mount working (fail-first check)**

Before running compose normally, intentionally make the bind path unwritable to confirm the healthcheck catches a broken setup:

```bash
ssh ${NAS_HOST} 'cd ~/chromecast-ledfx && chmod 000 volumes/ledfx-pulse && docker compose up -d ledfx && sleep 30 && docker inspect --format "{{.State.Health.Status}}" ledfx'
```

Expected: `unhealthy` or `starting`. Then restore:

```bash
ssh ${NAS_HOST} 'cd ~/chromecast-ledfx && docker compose down && chmod 0755 volumes/ledfx-pulse && sudo chown -R 1000:1000 volumes/ledfx-pulse'
```

- [ ] **Step 5: Bring up LedFx for real**

```bash
ssh ${NAS_HOST} 'cd ~/chromecast-ledfx && docker compose up -d ledfx'
```

Wait ~30s, then check health:

```bash
ssh ${NAS_HOST} 'docker inspect --format "{{.State.Health.Status}}" ledfx'
```

Expected: `healthy`.

- [ ] **Step 6: Confirm the Pulse socket exists on the host**

```bash
ssh ${NAS_HOST} 'ls -la ~/chromecast-ledfx/volumes/ledfx-pulse/'
```

Expected: a file named `pulseaudio.socket` (a unix socket — `srwx...`) and a `cookie` file. If only one or neither, the LedFx image isn't running Pulse in server mode — check container logs with `docker logs ledfx` and verify the image tag.

- [ ] **Step 7: Confirm LedFx web UI is reachable**

From your workstation:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://${NAS_HOST}:8888/
```

Expected: `200`.

- [ ] **Step 8: Commit**

```bash
git add docker-compose.yml
git commit -m "Add LedFx service with PulseAudio server mode"
```

---

### Task 3: shairport-sync config + service (AirPlay 2 receiver)

**Goal:** Bring up the AirPlay 2 receiver, confirm it's discoverable, and confirm audio AirPlayed to it reaches LedFx's Pulse.

**Files:**
- Create: `shairport-sync/shairport-sync.conf`
- Modify: `docker-compose.yml` (add `shairport` service)

- [ ] **Step 1: Write `shairport-sync/shairport-sync.conf`**

```
// shairport-sync 5.x configuration. Reference: man shairport-sync.conf

general = {
    name = "Bathroom-Sync";
    output_backend = "pulseaudio";

    // Hold playback at unity gain so LedFx FFT sees consistent levels
    // regardless of what the AirPlay sender sets its volume to. Volume
    // control still happens at the LP10 / amplifier on the speaker side.
    ignore_volume_control = "yes";

    // AP2 mode is the default with the non-classic image; explicit for clarity.
    interpolation = "auto";
};

pulseaudio = {
    server = "unix:/tmp/pulseaudio.socket";
    sink = "";  // Use the Pulse default sink created by LedFx's pulseaudio.
};

diagnostics = {
    log_verbosity = 1;  // 0 silent, 1 normal, 2 verbose, 3 very verbose
};
```

- [ ] **Step 2: Add `shairport` service to `docker-compose.yml`**

Append to the `services:` block (above the trailing `volumes:` block):

```yaml
  shairport:
    image: ${SHAIRPORT_IMAGE}
    container_name: shairport
    restart: unless-stopped
    network_mode: host
    cap_add:
      - SYS_NICE          # nqptp scheduling priority
    depends_on:
      ledfx:
        condition: service_healthy
    environment:
      PULSE_SERVER: "unix:/tmp/pulseaudio.socket"
      PULSE_COOKIE: "/tmp/cookie"
    volumes:
      - ./shairport-sync/shairport-sync.conf:/etc/shairport-sync.conf:ro
      - ./volumes/ledfx-pulse:/tmp:rw
    logging:
      options:
        max-size: "10m"
        max-file: "5"
```

`./volumes/ledfx-pulse` is mounted at `/tmp` inside the shairport container so that `unix:/tmp/pulseaudio.socket` and `/tmp/cookie` resolve to the same files LedFx creates in `~/.config/pulse/`.

- [ ] **Step 3: Sync and bring up the full stack**

```bash
rsync -avz --exclude=.git --exclude=volumes --exclude=.env ./ ${NAS_HOST}:~/chromecast-ledfx/
ssh ${NAS_HOST} 'cd ~/chromecast-ledfx && docker compose up -d'
```

Compose will start `ledfx` first, wait for its healthcheck to pass, then start `shairport`. If `shairport` fails to start, the most likely cause is that the Pulse socket isn't where `shairport-sync.conf` expects it — check Step 5 below.

- [ ] **Step 4: Verify both containers healthy**

```bash
ssh ${NAS_HOST} 'docker compose -f ~/chromecast-ledfx/docker-compose.yml ps'
```

Expected: both `ledfx` (healthy) and `shairport` (Up).

- [ ] **Step 5: Verify shairport-sync logs show clean startup with AP2 + nqptp**

```bash
ssh ${NAS_HOST} 'docker logs shairport 2>&1 | tail -40'
```

Expected: lines like `Startup in AirPlay 2 mode`, `Successful clock control via /dev/shm/nqptp`, no `port unavailable`, no `unable to connect to PulseAudio`. If you see PulseAudio auth errors, the cookie isn't being shared correctly — verify `/tmp/cookie` exists in the container: `docker exec shairport ls -la /tmp/`.

- [ ] **Step 6: Confirm Bathroom-Sync is discoverable on the LAN**

From a Mac on the same network:

```bash
dns-sd -B _airplay._tcp local.
```

Expected: a row containing `Bathroom-Sync`. Ctrl-C to stop.

- [ ] **Step 7: AirPlay test from a Mac/iPhone**

Manually: open AirPlay in Control Center on a Mac/iPhone, select **Bathroom-Sync**, play any audio for ~10 seconds. (You won't hear anything — there's no physical output yet — that's correct. We're just confirming the receiver accepts a stream.)

While playing, verify Pulse is receiving the stream:

```bash
ssh ${NAS_HOST} 'docker exec ledfx pactl list short sink-inputs'
```

Expected: at least one sink-input row referencing shairport-sync. If empty, the audio isn't reaching Pulse — recheck the bind-mount and `PULSE_SERVER` setting.

- [ ] **Step 8: Commit**

```bash
git add shairport-sync/shairport-sync.conf docker-compose.yml
git commit -m "Add shairport-sync AirPlay 2 receiver feeding LedFx Pulse"
```

---

### Task 4: Configure LedFx audio input + WLED device, verify end-to-end reactivity

**Goal:** Tell LedFx to listen on the Pulse default source and to drive the WLED controller; confirm LEDs react to audio AirPlayed to Bathroom-Sync.

This task is **interactive in the LedFx web UI** but the verification commands are scriptable.

**Files:** none in repo (LedFx state lives in the named volume).

- [ ] **Step 1: List LedFx audio devices via API**

```bash
curl -s http://${NAS_HOST}:8888/api/audio/devices | python3 -m json.tool
```

Expected: a JSON object with `devices` keyed by index. Identify the Pulse default source (likely shows up as something like "Monitor of <sink>" or "pulse"). Note its index (e.g., `1`).

- [ ] **Step 2: Set the active audio device**

Replace `<idx>` with the index from Step 1:

```bash
curl -s -X PUT http://${NAS_HOST}:8888/api/audio/devices \
  -H "Content-Type: application/json" \
  -d '{"audio_device": <idx>}'
```

Expected: `{"status": "success"}`.

- [ ] **Step 3: Confirm LedFx sees audio levels**

Open `http://${NAS_HOST}:8888/` in a browser. AirPlay audio from a phone/Mac to **Bathroom-Sync**. The dashboard's audio meter (or the "Audio" debug panel) should show level activity. If flatlined, recheck Task 3 Step 7.

- [ ] **Step 4: Add WLED device in LedFx**

In the web UI: **Devices → Add Device → WLED**. Enter the WLED IP from `.env` (`WLED_IP`). LedFx auto-detects pixel count. Save.

Equivalent API call (if you prefer scripted):

```bash
curl -s -X POST http://${NAS_HOST}:8888/api/devices \
  -H "Content-Type: application/json" \
  -d '{
    "type": "wled",
    "config": {
      "name": "Bathroom",
      "ip_address": "'${WLED_IP}'",
      "sync_mode": "DDP",
      "timeout": 1
    }
  }'
```

- [ ] **Step 5: Add a virtual + an audio-reactive effect**

In the UI: **Virtuals → Add Virtual → assign Bathroom device → Effects → pick an audio-reactive effect** (e.g., "Bands", "Energy", "Scroll"). Set effect to active.

- [ ] **Step 6: End-to-end reactive test**

AirPlay a song with clear dynamics (drums, beats) directly to Bathroom-Sync from a phone. Confirm:
- Audio is **silent** at the speaker (LP10 not yet involved).
- LEDs in the bathroom react to the audio in real time.
- No visible packet loss or freezing in the LED output.

If LEDs are unresponsive but audio meters move, the WLED side is misconfigured — check WLED's sync settings (DDP must be enabled in WLED's Sync Interfaces page, port 4048).

- [ ] **Step 7: Commit (no repo changes; just record progress)**

There are no repo changes in this task — LedFx state is in a Docker volume, not git. Skip the commit step. (Optional: tag the achievement with `git tag -a milestone/ledfx-reactive -m "..."`.)

---

### Task 5: Music Assistant integration (LP10 + Bathroom-Sync sync group)

**Goal:** Make MA address both the LP10 and Bathroom-Sync as a synchronized AirPlay group so MA-driven playback hits the speaker AND the LEDs.

This is performed entirely in the **Music Assistant UI** (web frontend on the HA server) — no repo changes.

- [ ] **Step 1: Verify LP10 is in MA**

In MA → Settings → Providers → AirPlay. Confirm the LP10 is listed under discovered players. If not, click **Add Player** or wait ~60 s for mDNS rediscovery. Note its MA player ID.

- [ ] **Step 2: Verify Bathroom-Sync is in MA**

Same screen. **Bathroom-Sync** should appear automatically since it's advertising on the same LAN as the HA host. If MA prompts for an AirPlay 2 PIN/verifier, paste the verifier code from the shairport-sync container logs:

```bash
ssh ${NAS_HOST} 'docker logs shairport 2>&1 | grep -i "verification\|setup code\|pin"'
```

If shairport-sync 5.x's default config doesn't print a setup code, edit `shairport-sync/shairport-sync.conf` to add (rare; most networks don't need it):

```
general = {
    // ...
    // Uncomment if your setup requires HomeKit pairing for MA to control the receiver:
    // homekit_pairing_pin = "3939";
};
```

- [ ] **Step 3: Create the sync group in MA**

In MA → Players → select **Bathroom-Sync** → Player options → **Sync with…** → select **Arylic LP10**. Save. The group's name in MA will be something like "Bathroom-Sync + Arylic LP10."

- [ ] **Step 4: Test playback to the group**

From MA's "Now Playing" view, queue a track and direct it to the sync group. Confirm:
- Audible: bathroom speaker plays the track.
- Visible: bathroom LEDs react in sync (within ~100 ms).
- No "stuttering" on either endpoint.

If the LP10 plays but LEDs don't react: MA may be streaming only to the LP10 and not to Bathroom-Sync. Re-create the group — MA's grouping UX has been known to silently de-sync after restarts; toggling the sync off and back on usually fixes it.

- [ ] **Step 5: No commit (MA state lives outside this repo)**

---

### Task 6: (Optional) Apple Home AirPlay 2 group

**Goal:** Enable direct AirPlay from any Apple device to a HomeKit-defined group containing both endpoints.

Skip this task if you only want MA-driven playback.

- [ ] **Step 1: Add Bathroom-Sync to Apple Home**

On an iPhone with the Home app: **Add Accessory** → tap "More options" if it's not auto-detected → choose **Bathroom-Sync**. If a HomeKit pairing code is required, get it from the shairport-sync container logs:

```bash
ssh ${NAS_HOST} 'docker logs shairport 2>&1 | grep -i "homekit\|pin\|setup"'
```

If no code prints (5.x publishes it on first pairing attempt), check the AP2 docs: https://github.com/mikebrady/shairport-sync/blob/master/AIRPLAY2.md

- [ ] **Step 2: Confirm LP10 is in Apple Home**

The Arylic LP10 appears in Home automatically as an AirPlay 2 device. If not, follow Arylic's app instructions to enable HomeKit on the LP10.

- [ ] **Step 3: Group them**

Long-press Bathroom-Sync in the Home app → settings → **Speakers & TVs → Group with…** → add LP10. The group will appear as a single AirPlay target (e.g., "Bathroom").

- [ ] **Step 4: Test direct AirPlay**

From any Apple device, AirPlay to the **Bathroom** group. Confirm both speaker and LEDs react in sync. AirPlay 2 native multi-room sync should keep them within ~10–50 ms.

---

### Task 7: NAS reboot resilience + acceptance

**Goal:** Confirm the stack survives a reboot and recovers without manual intervention.

- [ ] **Step 1: Reboot the NAS**

```bash
ssh ${NAS_HOST} 'sudo reboot'
```

Wait for the NAS to come back online (typically 1–3 minutes for QNAP).

- [ ] **Step 2: Verify the stack came back up automatically**

```bash
ssh ${NAS_HOST} 'docker compose -f ~/chromecast-ledfx/docker-compose.yml ps'
```

Expected: both services running. If `ledfx` is unhealthy after 60s, check Container Station's "Auto-start" setting for the project.

- [ ] **Step 3: Verify Bathroom-Sync rediscovered by MA**

In MA → Players, Bathroom-Sync should be online within ~60 s of reboot. If it stays offline, MA may need its airplay provider reloaded (Settings → Providers → AirPlay → Reload).

- [ ] **Step 4: Repeat the end-to-end reactive test from Task 4 Step 6**

Direct AirPlay → confirm LEDs react.

- [ ] **Step 5: Repeat the MA group test from Task 5 Step 4**

MA group playback → confirm speaker + LEDs both work.

- [ ] **Step 6: Final commit (any cleanup edits)**

If you tweaked any config during testing, commit them now:

```bash
git status
git add -A
git diff --cached
git commit -m "Tune config after acceptance testing"   # only if changes exist
```

If no changes, skip.

---

## Acceptance criteria (whole plan)

The implementation is complete when **all** of these hold:

1. `docker compose ps` on the NAS shows both `ledfx` (healthy) and `shairport` (running).
2. LedFx web UI at `http://<nas>:8888` responds 200 and shows audio levels when AirPlay is active.
3. Bathroom-Sync appears in any AirPlay picker on the LAN.
4. Direct AirPlay from a Mac/iPhone to Bathroom-Sync drives audio-reactive LEDs in the bathroom (audio is silent — speaker not in this path).
5. Music Assistant has both LP10 and Bathroom-Sync as players, can group them, and group playback drives both speaker and LEDs in sync.
6. (Optional, if Task 6 done) Apple Home AirPlay group drives both endpoints.
7. NAS reboot recovers the stack without manual intervention.

---

## Troubleshooting reference

| Symptom | Likely cause | Fix |
|---|---|---|
| `shairport` logs `port unavailable` | Another nqptp/PTP service binding 319/320 | Identify with `sudo ss -lun \| grep :319`; disable conflicting service |
| `shairport` logs PulseAudio auth error | Cookie not shared between containers | Verify `./volumes/ledfx-pulse/cookie` exists and is readable by UID 1000 |
| LedFx audio meters flat | Wrong audio device selected, or sink-input not landing | `pactl list short sink-inputs` inside ledfx container; reselect device via API |
| LEDs unresponsive but meters work | WLED DDP not enabled, or wrong IP | WLED Sync Interfaces page → enable DDP receive (port 4048) |
| MA group plays only on LP10 | Group de-sync after restart | Toggle MA sync off and on |
| Bathroom-Sync vanishes from LAN | Shairport crashed, mDNS confused | `docker logs shairport`; restart; verify Avahi on host isn't blocking |
| `network_mode: host` errors on Mac dev | Docker Desktop limitation | Test on the QNAP, not on the Mac |

---
