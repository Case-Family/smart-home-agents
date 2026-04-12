# Satellite Node (sat-1)

## Overview

sat-1 is the first wyoming satellite node, located in the living room. It provides
audio playback via Music Assistant (slimproto/squeezelite) and will provide voice
processing via the Wyoming protocol stack. This document captures the baseline
configuration established in April 2026.

---

## Hardware

| Component | Device |
|---|---|
| Host | Raspberry Pi 5 |
| AI Accelerator | Hailo-8L AI HAT+ |
| Microphone | ReSpeaker USB Mic Array v2.0 |
| DAC | AudioQuest DragonFly Cobalt v1 (USB) |
| Speaker | Audio Pro C20 W |

---

## Audio Playback (squeezelite)

sat-1 streams music from Music Assistant on harpi (192.168.1.227) via the
slimproto protocol, decoded and output through the DragonFly Cobalt DAC.

### Package

squeezelite 2.0.0-1517+git20241227 (arm64, ALSA)

### Configuration

/etc/default/squeezelite:
SL_NAME="Sat-1-DragonFly"
SL_SOUNDCARD="plughw:CARD=v1,DEV=0"
SB_SERVER_IP="192.168.1.227"
SB_EXTRA_ARGS="-a 80:4::1"

The DragonFly appears as ALSA card 0 (`v1`). The ReSpeaker mic array appears as
card 3 (`ArrayUAC10`) and is reserved for voice input.

### Service

squeezelite runs as a systemd service and is enabled at boot:

```bash
sudo systemctl enable squeezelite
sudo systemctl start squeezelite
```

### Music Assistant player name

`Sat-1-DragonFly`

---

## Known Issues / Gotchas

### Stale stream IP in Music Assistant

MA caches the stream server IP in its SQLite WAL files. If harpi has been on a
previous network, MA will advertise the old IP to squeezelite, causing playback
to stall at 0:00 with a connection timeout in the squeezelite logs. The database
files alone are not sufficient to clear this — the WAL and SHM sidecar files
must also be removed:

```bash
docker stop music_assistant
rm /home/hadm/docker/config/ma/library.db-wal
rm /home/hadm/docker/config/ma/auth.db-wal
rm -f /home/hadm/docker/config/ma/*.db-shm
docker start music_assistant
```

Verify the correct IP is advertised at startup:

```Starting streamserver on 192.168.1.227:8097```

---

## Planned

- Containerize squeezelite (align with harpi Docker-based deployment model)
- Wyoming voice pipeline (wyoming-hailo-whisper, wake word detection)
- Self-hosted GitHub Actions runner + CD pipeline

