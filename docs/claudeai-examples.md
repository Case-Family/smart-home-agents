# Example Scenarios - Intelligence System

## Scenario 1: Simple Single-User Gaming (Tier 1 - Local Rules)

**Signals:**
```json
{
  "chris_iphone": {"state": "home", "wifi": "connected"},
  "gaming_pc_office": {
    "steam_user": "chris",
    "active": true,
    "game": "wow.exe",
    "idle_seconds": 30
  },
  "nest_office": {"motion": "detected_2min_ago"}
}
```

**Expected Behavior:**
- **Tier 1 (Local Rules)**: All signals agree → Chris in office gaming
- **Method**: local_rules
- **Confidence**: 0.95
- **Cost**: $0
- **Response Time**: <100ms

**Prediction:**
```json
{
  "chris": {
    "location": "office",
    "activity": "gaming_wow",
    "confidence": 0.95
  }
}
```

**Validation** (30 seconds later):
- Echo office: Chris voice detected
- **Accuracy**: 1.0 (perfect)
- **Learning**: Reinforce simple gaming pattern

---

## Scenario 2: Recognized Pattern (Tier 2 - Pattern Match)

**Signals:**
```json
{
  "chris_iphone": "home",
  "oline_iphone": "home",
  "xbox_living_room": {"streaming": true},
  "nest_kitchen": {"motion": true, "timestamp": "now"},
  "echo_kitchen": {"last_activity": "15sec_ago"},
  "time": "20:30",
  "day": "Friday"
}
```

**Pattern Match:** Similar to 8 previous Friday evenings
- Pattern: "Friday_evening_movie_with_snack_run"
- Historical accuracy: 7/8 correct

**Expected Behavior:**
- **Tier 1**: Ambiguous (Xbox doesn't identify users)
- **Tier 2 (Pattern Match)**: Found matching pattern
- **Method**: pattern_match
- **Confidence**: 0.87
- **Cost**: $0
- **Response Time**: <500ms

**Prediction:**
```json
{
  "chris": {
    "location": "living_room",
    "activity": "streaming_video",
    "confidence": 0.85
  },
  "oline": {
    "location": "kitchen",
    "activity": "preparing_snacks",
    "confidence": 0.85,
    "details": {"likely_returning_to": "living_room"}
  }
}
```

**Validation** (90 seconds later):
- Oline iPhone: Moved to living_room WiFi zone
- Nest living_room: Two people detected (increased motion)
- **Accuracy**: 0.95 (predicted kitchen→living_room movement)
- **Learning**: Reinforce pattern, confidence 0.87→0.89

---

## Scenario 3: Complex Multi-User (Tier 3 - AI Reasoning)

**Signals:**
```json
{
  "chris_iphone": "home",
  "oline_iphone": "home",
  "brion_iphone": "home",
  "gaming_pc_office": {
    "steam_user": "chris",
    "process": "wow.exe",
    "duration_seconds": 7200,
    "active": true
  },
  "gaming_pc_living_room": {
    "steam_user": "brion",
    "process": "wow.exe",
    "duration_seconds": 7200,
    "active": true
  },
  "nest_kitchen": {"motion": true},
  "echo_kitchen": {"voice": "oline", "utterance": "Set timer 10 minutes"},
  "time": "19:30",
  "day": "Tuesday"
}
```

**Expected Behavior:**
- **Tier 1**: Ambiguous (multiple gaming, unclear coordination)
- **Tier 2**: No matching pattern (first time seeing this configuration)
- **Tier 3 (AI Reasoning)**: Call Claude API

**AI Prompt Context:**
```
Analyze these signals:
- Chris: Steam/WoW on office PC for 2 hours
- Brion: Steam/WoW on living room PC for 2 hours  
- Oline: Voice in kitchen 1 minute ago, motion detected
- Time: Evening, weekday

The coordinated 2-hour WoW sessions suggest Chris and Brion might be in same group/raid.
Oline's kitchen activity during their gaming suggests she might be preparing something.

Infer locations and activities.
```

**AI Response:**
```json
{
  "chris": {
    "location": "office",
    "activity": {
      "activity_name": "coordinated_wow_raid",
      "details": {"raid_group": ["chris", "brion", "possibly_oline"]},
      "interruptibility": "low"
    },
    "confidence": 0.90
  },
  "brion": {
    "location": "living_room",
    "activity": {
      "activity_name": "coordinated_wow_raid",
      "details": {"raid_group": ["chris", "brion", "possibly_oline"]},
      "interruptibility": "low"
    },
    "confidence": 0.90
  },
  "oline": {
    "location": "kitchen",
    "activity": {
      "activity_name": "preparing_raid_snacks",
      "details": {
        "inferred_from": "kitchen_activity_during_coordinated_gaming",
        "likely_destination": "master_bedroom"
      },
      "interruptibility": "medium"
    },
    "confidence": 0.75
  }
}
```

**Cost**: $0.03 (1200 input tokens, 400 output tokens)
**Response Time**: 3.2 seconds

**Validation** (2 minutes later):
- Echo office: Chris voice "Need more DPS"
- Nest living_room: Sustained motion
- Oline iPhone: Moved to master_bedroom
- Oline Mac (if monitored): WoW process started

**Accuracy**: 0.90 (all locations correct, activity inference accurate)

**Learning**:
- Create new pattern: "coordinated_family_wow_raid"
- Store with high confidence (0.90)
- Note: Novel activity "preparing_raid_snacks" validated
- Add to known activities list
- **Next time**: Use pattern match, save $0.03

---

## Scenario 4: Guest Profile Error (Tier 2 with Correlation)

**Signals:**
```json
{
  "tesla_model_y": {
    "active_profile": "Guest",
    "location": {"distance_from_home": 150},
    "moving": true
  },
  "chris_iphone": {"wifi_disconnected": "90sec_ago"},
  "oline_iphone": {"wifi": "connected", "location": "home"},
  "brion_iphone": {"wifi": "connected", "location": "home"}
}
```

**Expected Behavior:**
- **Tier 1**: Guest profile - can't directly identify
- **Tier 2**: Correlate with iPhone departures
  - Chris iPhone left 90 seconds ago
  - Model Y left shortly after
  - Correlation: Chris likely driving (profile error)

**Prediction:**
```json
{
  "chris": {
    "location": "away",
    "activity": "driving_tesla",
    "confidence": 0.85,
    "note": "Profile shows Guest but iPhone correlation suggests Chris"
  }
}
```

**Cost**: $0 (pattern-based correlation)
**Additional Action**: Notify Chris about profile issue

**Validation** (later when returns):
- Tesla arrives with Chris profile (he fixed it)
- Or Chris's iPhone confirms he was gone
- **Accuracy**: 1.0
- **Learning**: Reinforce "guest_profile_with_single_iphone_departure" pattern

---

## Scenario 5: Budget Exhausted (Graceful Degradation)

**Monthly Status:**
- Budget: $25.00
- Spent: $25.15
- Budget exceeded: Yes

**New Complex Signal Arrives** (requires AI):
```json
{
  "multiple_ambiguous_signals": {...}
}
```

**Expected Behavior:**
- Check budget: Exhausted
- Skip Tier 3 (AI)
- Use Tier 2 (best effort pattern match)
- OR use Tier 1 (conservative local rules)
- Log: "Budget exhausted, using fallback inference"

**Prediction:**
```json
{
  "chris": {
    "location": "office",
    "confidence": 0.65,
    "note": "Lower confidence - budget prevented AI reasoning"
  }
}
```

**Cost**: $0
**Dashboard Alert**: "AI budget exhausted, using local intelligence only"

**Validation**: Still happens
**Learning**: Still happens (but from lower-confidence predictions)

---

## Scenario 6: New Activity Discovery

**AI generates novel activity:**
```json
{
  "chris": {
    "location": "office",
    "activity": {
      "activity_name": "late_night_debugging_session",
      "details": {
        "inferred_from": "office_pc_active + 2am + no_gaming + rapid_typing",
        "likely_duration_minutes": 90,
        "stress_indicators": ["late_hour", "no_breaks"]
