# Data Structures - Intelligence System

## Core Prediction Structure

### Prediction (Generic)
```python
{
  "prediction_id": "pred_abc123",
  "prediction_type": "location",  # or "identity", "activity", future types
  "timestamp": "2024-01-15T14:30:00Z",
  
  "predictions": {
    "chris": "office",      # entity_id: predicted_value
    "oline": "kitchen"
  },
  
  "method": "pattern_match",  # local_rules, pattern_match, ai_reasoning
  "confidence": 0.85,
  "reasoning": "Historical pattern: kitchen Echo + motion = Oline getting snack",
  
  "signals_used": {
    "chris_iphone": {"state": "home", "room": "unknown"},
    "gaming_pc_office": {"steam_user": "chris", "active": true, "game": "wow"},
    "echo_kitchen": {"voice_user": "oline", "timestamp": "2024-01-15T14:29:45Z"},
    "nest_kitchen": {"motion": true, "timestamp": "2024-01-15T14:29:50Z"}
  },
  
  "context": {
    "time_of_day": "afternoon",
    "day_type": "weekday",
    "other_users_home": ["chris", "oline", "brion"]
  },
  
  "cost": 0.0,  # $0 for local/pattern, $0.02-0.05 for AI
  "validated": false,
  "validation_window_seconds": 60
}
```

### ValidationResult
```python
{
  "prediction_id": "pred_abc123",
  "prediction_type": "location",
  "validated_at": "2024-01-15T14:30:45Z",
  
  "accuracy_score": 1.0,  # 0.0 (wrong) to 1.0 (perfect)
  "was_correct": true,    # accuracy >= 0.8
  
  "actual_values": {
    "chris": "office",    # Confirmed by Echo voice
    "oline": "kitchen"    # Confirmed by Nest motion + Echo
  },
  
  "error_type": null,  # wrong_room_same_floor, wrong_floor, etc.
  "error_details": {}
}
```

### LearnedPattern
```python
{
  "pattern_id": "pattern_xyz789",
  
  "fingerprint": {
    "present_users": ["chris", "oline"],
    "active_devices": ["gaming_pc_office", "echo_kitchen", "nest_kitchen"],
    "recent_movements": ["chris_stationary_2h", "oline_kitchen_brief"],
    "time_of_day": "evening",
    "day_type": "weekday"
  },
  
  "outcome": {
    "chris": {"location": "office", "activity": "gaming_wow"},
    "oline": {"location": "kitchen", "activity": "preparing_snacks"}
  },
  
  "confidence_score": 0.92,  # Updated based on validations
  "usage_count": 15,         # Times pattern was used
  "success_count": 14,       # Times it was correct
  
  "created_at": "2024-01-10T20:15:00Z",
  "last_success": "2024-01-15T19:30:00Z",
  "last_failure": null,
  "status": "active"  # active, unreliable, deprecated
}
```

## Prediction Type Schemas

### PredictionSchema
```python
{
  "prediction_type": "activity",
  "value_type": "structured",  # discrete, continuous, structured
  
  # For discrete types
  "known_values": [
    "gaming_steam", "gaming_wow", "streaming_video",
    "working", "sleeping", "cooking"
  ],
  "allow_new_values": true,  # AI can generate new activities
  
  # For structured types
  "schema_definition": {
    "type": "object",
    "properties": {
      "activity_name": {"type": "string"},
      "confidence": {"type": "number", "min": 0, "max": 1},
      "details": {"type": "object"},
      "duration_estimate": {"type": "number"},
      "interruptibility": {
        "type": "string",
        "enum": ["high", "medium", "low", "none"]
      }
    }
  },
  
  "description": "Current user activity with context",
  "examples": [
    "gaming_wow",
    '{"activity_name": "family_movie_night", "interruptibility": "low"}'
  ]
}
```

### Activity Value Examples

**Simple (string):**
```python
"gaming_wow"
```

**Structured (AI-generated):**
```python
{
  "activity_name": "late_night_coding_session",
  "details": {
    "inferred_from": "office_pc_active + late_hour + no_gaming_process",
    "likely_duration_minutes": 120
  },
  "interruptibility": "medium",
  "confidence": 0.75
}
```

## Signal Structures

### Tesla Signal
```python
{
  "device_id": "tesla_model_y",
  "active_profile": "Chris",  # or "Guest" if profile error
  "location": {
    "distance_from_home": 150,  # meters
    "latitude": 37.7749,
    "longitude": -122.4194
  },
  "battery_level": 80,
  "charging": false,
  "timestamp": "2024-01-15T14:30:00Z"
}
```

### Gaming Device Signal
```python
{
  "device_id": "gaming_pc_office",
  "steam_user": "chris",
  "active": true,
  "game_process": "wow.exe",
  "game_title": "World of Warcraft",
  "idle_seconds": 15,  # Last keyboard/mouse input
  "session_duration": 7200,  # 2 hours
  "timestamp": "2024-01-15T14:30:00Z"
}
```

### Echo Voice Signal
```python
{
  "device_id": "echo_kitchen",
  "type": "voice",
  "user_id": "oline",
  "room": "kitchen",
  "utterance": "Set timer 10 minutes",
  "timestamp": "2024-01-15T14:29:45Z"
}
```

### Nest Sensor Signal
```python
{
  "device_id": "nest_protect_kitchen",
  "sensor_type": "pathlight",  # motion, occupancy, button
  "triggered": true,
  "room": "kitchen",
  "timestamp": "2024-01-15T14:29:50Z"
}
```

## Database Schemas

### predictions table
```sql
CREATE TABLE predictions (
    prediction_id TEXT PRIMARY KEY,
    prediction_type TEXT NOT NULL,
    timestamp TEXT NOT NULL,
    method TEXT NOT NULL,
    confidence REAL NOT NULL,
    predictions_json TEXT NOT NULL,
    signals_json TEXT,
    context_json TEXT,
    reasoning TEXT,
    cost REAL DEFAULT 0.0,
    validated INTEGER DEFAULT 0,
    validated_at TEXT,
    accuracy_score REAL,
    actual_values_json TEXT,
    was_correct INTEGER
);
```

### learned_patterns table
```sql
CREATE TABLE learned_patterns (
    pattern_id TEXT PRIMARY KEY,
    fingerprint TEXT NOT NULL,
    outcome TEXT NOT NULL,
    time_of_day TEXT,
    day_type TEXT,
    confidence_score REAL DEFAULT 1.0,
    usage_count INTEGER DEFAULT 1,
    success_count INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_success TIMESTAMP,
    last_failure TIMESTAMP,
    status TEXT DEFAULT 'active'
);
```

### budget_tracking (Redis keys)
```
ai_cost:2024-01 → 15.47  (monthly spend)
ai_request_log → [list of request details]
accuracy_stats:local_rules → {total: 150, correct: 142, avg: 0.95}
accuracy_stats:pattern_match → {total: 80, correct: 71, avg: 0.89}
accuracy_stats:ai_reasoning → {total: 25, correct: 24, avg: 0.96}
```
