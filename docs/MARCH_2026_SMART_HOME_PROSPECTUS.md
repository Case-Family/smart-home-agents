# March 2026 Smart Home Prospectus
## smart-home-agents

### Vision
A fully local, privacy-first smart home AI ecosystem branded "Jarvis" — no walled gardens, no cloud dependencies for core functions. Voice interaction, presence awareness, behavioral intelligence, and adaptive automation running on owned infrastructure.

---

### Hardware Foundation

**HA Server (harpi)**
Raspberry Pi 5 8GB, Hailo-10H AI HAT+ 2 (40 TOPS, 8GB RAM), CanaKit Turbine extended case with active cooling, USB SSD. Handles LLM inference locally via Llama 3.2 3B, PostgreSQL + TimescaleDB, and all backend services.

**Theater Satellite (pilot)**
Raspberry Pi 5 8GB, Hailo-8L AI HAT+ (26 TOPS), GeeekPi metal case with active cooler, USB SSD. ReSpeaker USB Mic Array v2.0 top-mounted on Audio Pro C20 W speaker. Audioquest Dragonfly DAC → Audio Pro C20 W aux in. Handles wake word detection, Whisper STT, and Piper TTS locally via Hailo acceleration.

**Network**
TP-Link Deco XE75 Pro replacement pending structured wiring cable survey. Target: UniFi Cloud Gateway Ultra + PoE switch + U6 Mesh APs. Real VLAN segmentation — trusted, IoT, and guest networks. Existing structured wiring throughout house likely intact pending survey with KAIWEETS cable tester.

---

### Software Stack

**Satellite**
- linux-voice-assistant (ESPHome protocol) — satellite ↔ HA communication
- wyoming-hailo-whisper — Hailo-accelerated Whisper STT via Wyoming protocol bridge
- Piper TTS — voice response synthesis
- Docker throughout, GitHub Actions self-hosted runner for CD

**Server**
- Home Assistant, Music Assistant
- nginx reverse proxy (DuckDNS + Let's Encrypt) — chris-alexa.duckdns.org, chris-music.duckdns.org
- PostgreSQL + TimescaleDB — single database instance
- All containers managed via Docker Compose, deployed via GitHub Actions pipeline

---

### Service Architecture

| Service | Responsibility |
|---------|----------------|
| Identity | Voiceprint enrollment, user lifecycle, guest registration initiated by authorized resident |
| Location | Presence detection, trajectory inference (Nest Protect, Tesla, iPhone WiFi correlation) |
| Activity | Behavioral patterns, context awareness, baseline modeling |
| Forecasting | Prediction recording, accuracy tracking, confidence scoring, LLM routing guidance |
| Supervisor | Adaptive control loop, anomaly detection, Phase 1: notify admin + HA app prompt |
| Orchestrator | Intent routing, task coordination across services |
| Action | Notification routing, device control, HA integration — Phase 1 focused on notification |

---

### Key Design Decisions

**Local first**
Hailo-8L handles STT/TTS on satellite. Hailo-10H handles LLM inference on server (Llama 3.2 3B at launch). RTX 5080 workstation handles deep reasoning tasks (debate coach, long context). API calls reserved for frontier model capability only. Target API budget ~$20-30/month.

**ESPHome protocol over Wyoming for satellite**
wyoming-satellite deprecated in favor of linux-voice-assistant (OHF-Voice) implementing ESPHome protocol via aioesphomeapi. wyoming-hailo-whisper bridges Hailo STT into HA voice pipeline via Wyoming protocol — the two protocols serve different layers and are complementary.

**PostgreSQL + TimescaleDB**
Single database instance. Standard tables for identity, voiceprints, user preferences. Hypertables for prediction/accuracy time series data. HA recorder migrated from SQLite to PostgreSQL. Schema migrations via Flyway planned for Phase 1.5.

**LLDAP evaluated and rejected**
Schema too limited for voiceprint storage — no binary/octetString attribute type. Modification operations not supported via standard LDAP. Identity and voiceprint data stored in PostgreSQL. LLDAP may be revisited for pure identity/group management if needed.

**Voiceprint identification**
SpeechBrain CPU-based speaker identification — Hailo reserved for latency-critical STT/TTS. Natural conversation enrollment initiated by authorized resident post wake word. Post wake word identification only in Phase 1. Uncertain match escalates to HA app notification for resident resolution. All voice data stays local; embeddings may be offloaded to server for enrollment processing.

**LLM routing via Forecasting service**
Forecasting service records predictions made by all agents with confidence estimates and resolves against actual outcomes. Accuracy history per agent per query domain informs routing decisions — local Hailo-10H vs RTX 5080 workstation vs external API. Routing intelligence grows from real accuracy data rather than static rules.

**Notification routing via Action service**
Notification = f(Identity, Location, Activity, Priority). Channel selection driven by who, where, what state, and priority. Chris asleep → suppress routine, allow emergency via bedroom speaker. Chris in office → HA app for routine, phone text for urgent. At satellite → respond via that channel's TTS. Action service Phase 1 focused on notification; device control (Roborock, lights, etc.) added as handlers.

**Network segmentation**
Real VLAN isolation prerequisite for voiceprint/presence/behavioral data security. TP-Link Deco IoT "network" is same subnet — not real segmentation. UniFi upgrade planned post cable survey. IoT VLAN isolates Nest Protects, Roborock, Litter-Robot, smart devices from trusted Pi infrastructure.

**CD pipeline**
GitHub Actions with self-hosted runner on harpi. Push to main triggers docker compose pull && docker compose up -d --remove-orphans. Idempotent by nature. docker compose config validation gate before deploy. Secrets via .env on Pi, never in repo. .env.example documents required variables.

**Secrets management**
SOPS + age under consideration. DuckDNS token and LetsEncrypt certs externalized from repo. Age private key in personal password manager, GitHub Secret for pipeline use. Primary threat model: IoT devices with poor security on home network — addressed by VLAN segmentation.

---

### Phasing

**Phase 0 — Foundation**
- GitHub repo initialized, compose file and nginx config captured
- Secrets externalized to .env
- Self-hosted GitHub Actions runner on harpi
- CD pipeline operational — push to main deploys
- PostgreSQL migration from SQLite
- Restack from repo validated

**Phase 1 — Theater Satellite Pilot**
- Pi 5 satellite assembled, Hailo-8L validated (hailortcli scan)
- ReSpeaker USB audio capture verified
- Audioquest Dragonfly audio out verified
- linux-voice-assistant + wyoming-hailo-whisper containers
- Wake word detection operational
- STT/TTS through to HA voice pipeline
- Basic HA integration — music, lights, simple commands

**Phase 1.5 — Database Migrations**
- Flyway migration tooling added to pipeline
- Schema versioning established before service development begins

**Phase 2 — Identity and Notification**
- Identity service: voiceprint enrollment, guest registration flow
- SpeechBrain speaker identification
- Natural conversation enrollment UX
- Action service: notification routing per Identity/Location/Activity context
- Guest lifecycle management

**Phase 3 — Intelligence Layer**
- Forecasting service: prediction recording, accuracy tracking, TimescaleDB hypertables
- Supervisor: anomaly detection, admin notification, HA app escalation
- LLM routing guidance from Forecasting confidence scores
- Sentiment analysis

**Phase 4 — Scale**
- Additional satellites (kitchen, office, bedroom)
- Repeatable satellite deployment via CD pipeline
- Full service mesh operational
- UniFi network segmentation with IoT VLAN isolation

---

### Integrations

| Integration | Notes |
|-------------|-------|
| Tesla vehicles | Presence inference via iPhone WiFi correlation — API does not expose active driver profile |
| Amazon Echo devices | Alexa Media Player integration — repurposed as Action service audio endpoints in secondary rooms |
| Nest Protect sensors | Motion/presence events → Location service trajectory inference |
| Roborock | Action service target |
| Litter-Robot | Action service target |
| Music Assistant | Context-aware playback, Chris/Oline preference profiles |
| Speech Coach | Brion debate prep — playback analysis, composition, brainstorming via RTX 5080 workstation |
| Sony Aibo ERS-7 | Future mobile sensor platform — earmarked, not yet active |

---

### Infrastructure Notes

- **harpi** user: `hadm`, working directory: `/home/hadm/docker/`
- **Shared media volume**: `/home/hadm/docker/shared_media` — mounted by both HA and MA
- **nginx config**: single `default.conf`, two virtual hosts (Alexa bridge :5000, MA UI/stream :8095/:8097)
- **DuckDNS subdomains**: chris-alexa, chris-music
- **Alexa integration**: stalled but preserved — nginx/DuckDNS/LetsEncrypt infrastructure valuable independently
- **Pi 5 server**: cabled to router in NOC closet (walk-in closet)
- **NOC closet**: Netgear Nighthawk cable modem, ChannelPlus structured wiring panels, coax distribution, shelf with Deco primary node and harpi

---

*Prospectus current as of March 2026. Hardware ordered, pipeline work next.*
