
# Smart Home Intelligence System - Architecture Overview

## System Purpose
Multi-agent smart home automation system running on Raspberry Pi 5 with AI-powered intelligence layer for presence detection, location tracking, and activity inference.

## Key Constraints
- Monthly AI budget: $20-30 (Anthropic Claude API)
- Local processing preferred (Pi 5, 8GB RAM)
- Must work offline (graceful degradation)
- Self-improving through validation and learning
- Extensible to new prediction types

## Physical Environment
**House Layout:**
- Downstairs: kitchen, living_room, entry, garage, backyard
- Upstairs: office, master_bedroom, kid_bedroom (Brion), raina_bedroom, upstairs_hallway

**Residents:**
- Chris (permanent) - iPhone, Apple Watch, Tesla Model Y profile
- Oline (permanent) - iPhone, Apple Watch (occasional), Tesla Model S profile
- Brion (permanent) - iPhone, Tesla "lolmobile" profile
- Raina (occasional - away at college) - iPhone, Tesla Model Y profile (shared)

**Devices:**
- 4 Nest Protect (hardwired): kitchen, upstairs_hallway, master_bedroom, kid_bedroom
- 3 Amazon Echo: kitchen, master_bedroom, kid_bedroom
- 3 Tesla vehicles (all profiles shared across family)
- Xbox One X (living_room): Streaming only, always Chris profile
- Gaming PC (living_room): Windows kiosk, Steam Big Picture, multi-user
- Gaming PC (office): Primarily Chris, Steam
- Oline's Mac (master_bedroom): WoW gaming
- Roborock vacuum (downstairs): Has mapped floor plan
- Various other devices (litter-robot, smart fridge, etc.)

## Core Architecture

### Agent Layer
1. **Identity Agent**: Detects presence (home/away) via iPhone WiFi, Tesla profiles
2. **Location Agent**: Infers room-level location from device interactions, Nest sensors
3. **Gaming Activity Agent**: Tracks gaming device usage for location/activity detection
4. **Intelligence Agent**: Multi-signal fusion with three-tier approach
5. **Notification Agent**: Routes notifications to appropriate channel

### Service Layer
1. **Authorization Service**: RBAC, privacy controls, guest management
2. **Logging Service**: Centralized logging
3. **Config Service**: Configuration management
4. **Monitor Service**: Health monitoring

### Infrastructure Layer
1. **Home Assistant**: Device integration hub
2. **MQTT Broker** (Mosquitto): Inter-agent communication
3. **Redis**: Real-time state cache
4. **SQLite**: Persistent storage

## Intelligence Layer - Three Tiers

**Tier 1: Local Rules Engine (Free, <100ms)**
- Handles 70-80% of cases
- Deterministic logic for clear signals
- Example: Single user, single device active = high confidence location

**Tier 2: Pattern Matcher (Free, <500ms)**
- Handles 15-20% using learned patterns
- Historical pattern recognition
- Self-improves through validation

**Tier 3: Claude AI Reasoning (Paid, 2-4s)**
- Handles 5-10% complex/ambiguous cases
- Cost: ~$0.02-0.05 per inference
- Only when budget allows

## Key Design Patterns

### Signal Confidence Weighting
- Tesla profile + movement: 99% (phone is car key, profile for comfort)
- Gaming device active: 90-95% (must be physically present)
- iPhone WiFi: 80% (good but has lag)
- Echo voice: 90% (direct interaction)
- Nest pathlight: 75% (movement detected)
- Nest motion: 65% (someone there, but who?)

### Prediction Validation Loop
1. System makes prediction (location, activity, etc.)
2. Record prediction with 60-second validation window
3. Wait for confirming signals (voice, device login, etc.)
4. Calculate accuracy score
5. Learn: Reinforce correct patterns, weaken incorrect ones

### Budget Management
- Track API calls and token usage in real-time
- Alert at 80% of monthly budget
- Fall back to local intelligence when budget exhausted
- Cache similar scenarios to avoid repeat API calls

## Extensibility Requirements

### Prediction Types
**Currently Supported:**
- Identity: Who is present (chris, oline, brion, raina, guests)
- Location: Which room (kitchen, office, etc.)
- Activity: What they're doing (gaming_wow, working, streaming_video, etc.)

**Key Requirement:** Activity predictions can include AI-generated novel activities
- Known activities: gaming_steam, gaming_wow, working, sleeping, etc.
- AI can generate: "late_night_coding_session", "family_movie_night", etc.
- System must handle and learn from these new activities

**Future Expansion:** System must support adding new prediction types (mood, intention, health_status, etc.) without architectural changes

### Pattern Learning
- Patterns stored with fingerprint (signal configuration) + outcome
- Confidence scores: start at 1.0, adjusted based on validation
- Patterns can be reinforced (more accurate) or weakened (less accurate)
- Unreliable patterns (confidence < 0.6) marked as inactive

## Data Flow Example

**Scenario: Complex evening activity**
```
Signals:
- Chris iPhone: home
- Oline iPhone: home
- Gaming PC (office): Chris Steam account, WoW.exe, 2 hours
- Gaming PC (living_room): Brion Steam account, WoW.exe, 2 hours
- Echo (kitchen): Oline voice, "Set timer 10 minutes"
- Nest (kitchen): Motion detected

Tier 1 (Local Rules): Ambiguous - multiple people, multiple locations
Tier 2 (Pattern Match): No matching pattern found
Tier 3 (AI Reasoning):
  → Query Claude with signals and context
  → Response: Chris (office, gaming_wow), Brion (living_room, gaming_wow),
              Oline (kitchen, preparing_snacks_for_raid)
  → Cost: $0.03
  → Confidence: 88%

Validation (2 minutes later):
- Echo (office): Chris voice detected
- Nest (living_room): Sustained motion
- Oline iPhone: Moved to living_room
→ Accuracy: 90% (all locations correct, activity inference close)
→ Learn: Create pattern "coordinated_family_wow_raid"
→ Next time: Use pattern match (save $0.03)
```

## Success Metrics
1. **Accuracy**: >90% prediction accuracy across all types
2. **Cost efficiency**: <$30/month AI budget
3. **Local coverage**: >85% cases handled without AI
4. **Learning rate**: Pattern accuracy improves over time
5. **Latency**: <5 seconds for complex inferences
