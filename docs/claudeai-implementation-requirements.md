# Implementation Requirements - Intelligence System

## Technical Stack
- **Language**: Python 3.11+
- **Platform**: Raspberry Pi 5 (8GB RAM, 128GB SD)
- **Message Bus**: MQTT (Mosquitto)
- **Cache**: Redis
- **Database**: SQLite
- **AI API**: Anthropic Claude (claude-sonnet-4-20250514)

## Critical Requirements

### 1. Budget Management
**MUST implement:**
- Real-time token tracking (input + output tokens)
- Monthly budget enforcement ($20-30)
- Alert at 80% budget usage
- Graceful degradation when budget exhausted
- Cost per request logging

**Token Costs (Anthropic Claude Sonnet):**
- Input: $3 per million tokens
- Output: $15 per million tokens

**Expected costs:**
- Simple inference: ~$0.02 (500 input, 200 output tokens)
- Complex inference: ~$0.05 (1500 input, 500 output tokens)

### 2. Validation Window
- All predictions recorded with 60-second validation window
- Active predictions stored in Redis with TTL
- Confirming signals trigger validation automatically
- Validation calculates accuracy score (0.0 to 1.0)

### 3. Pattern Learning
**Pattern creation:**
- Extract fingerprint from signals (abstracted, reusable)
- Store with initial confidence 1.0
- Track usage_count and success_count

**Pattern updates:**
- Reinforce: success_count++, confidence = success/usage
- Weaken: confidence = success/(usage+1)
- Mark unreliable if confidence < 0.6

**Pattern matching:**
- Find patterns with similarity > 0.85
- Compare fingerprints (users, devices, movements, time, day)
- Use most similar pattern if confidence >= 0.80

### 4. Extensibility
**New prediction types MUST:**
- Register with PredictionSchema
- Define value_type (discrete, continuous, structured)
- Specify if allow_new_values (for AI-generated values)
- Provide validator (or use default)
- Work with existing validation/learning infrastructure

**No code changes required** to add new types beyond registration

### 5. AI Prompt Design
**Context to provide Claude:**
- Current signals (all available sensor data)
- Present users and last known locations
- Recent activity history (past 30 minutes)
- Time of day and day of week context
- Any relevant patterns or historical context

**Request from Claude:**
- Structured JSON response
- Predictions for each user
- Confidence scores
- Reasoning for complex inferences
- Alternative interpretations if ambiguous

**Example prompt structure:**
```
You are analyzing smart home sensor data to infer user locations and activities.

Current Signals:
- Chris's iPhone: home WiFi connected
- Gaming PC (office): Chris's Steam account logged in, WoW process active for 2 hours
- Gaming PC (living_room): Brion's Steam account, WoW process active for 2 hours  
- Echo (kitchen): Oline voice detected 2 minutes ago saying "Set timer 10 minutes"
- Nest (kitchen): Motion detected 2 minutes ago
- Nest (living_room): Sustained motion

Context:
- Time: 7:30 PM, weekday
- All three users confirmed home (Tesla arrivals, iPhone WiFi)
- This is unusual - typically not all gaming simultaneously

Based on these signals, infer:
1. Current location of each user (room level)
2. Current activity of each user
3. Any coordinated activities

Respond in JSON format:
{
  "chris": {
    "location": "office",
    "activity": {...},
    "confidence": 0.95
  },
  ...
}
```

### 6. Performance Requirements
- Tier 1 (Local Rules): < 100ms response time
- Tier 2 (Pattern Match): < 500ms response time  
- Tier 3 (AI Reasoning): < 5 seconds acceptable
- Overall system: Handle 100+ signal updates per minute

### 7. Reliability Requirements
- Must work offline (Tier 1 & 2 only)
- Graceful degradation when services unavailable
- Retry logic for transient failures
- Logging of all decisions and errors

### 8. Testing Requirements
**Unit tests:**
- Each tier independently
- Validators for each prediction type
- Pattern learning logic
- Budget management

**Integration tests:**
- Signal fusion across tiers
- Validation loop
- Pattern learning from validations
- Budget enforcement

**Acceptance tests:**
- Real-world scenarios from usage examples
- Accuracy measurement
- Cost tracking verification
- Learning effectiveness

## Implementation Phases

### Phase 1: Core Framework (Week 1)
- Prediction framework (generic structures)
- Prediction registry with initial types
- Database schema and storage
- Basic validators

### Phase 2: Inference Tiers (Week 2)
- Local rules engine
- Pattern matcher with learning
- Claude API integration
- Budget manager

### Phase 3: Validation & Learning (Week 3)
- Prediction validation system
- Pattern learning from validations
- Accuracy tracking
- Confidence updates

### Phase 4: Integration (Week 4)
- Connect to existing agents
- Signal ingestion
- MQTT integration
- Dashboard for monitoring

### Phase 5: Testing & Tuning (Week 5)
- Acceptance test suite
- Real-world validation
- Pattern library seeding
- Budget optimization

## Key Algorithms

### Signal Similarity Calculation
```python
def calculate_similarity(fingerprint1, fingerprint2):
    """
    Compare two pattern fingerprints
    Returns 0.0 (completely different) to 1.0 (identical)
    """
    scores = []
    
    # Compare present users (Jaccard similarity)
    users1, users2 = set(fp1['present_users']), set(fp2['present_users'])
    if users1 and users2:
        scores.append(len(users1 & users2) / len(users1 | users2))
    
    # Compare active devices
    devices1, devices2 = set(fp1['active_devices']), set(fp2['active_devices'])
    if devices1 and devices2:
        scores.append(len(devices1 & devices2) / len(devices1 | devices2))
    
    # Exact matches for time/day
    if fp1['time_of_day'] == fp2['time_of_day']:
        scores.append(1.0)
    if fp1['day_type'] == fp2['day_type']:
        scores.append(1.0)
    
    return sum(scores) / len(scores) if scores else 0.0
```

### Accuracy Score Calculation
```python
def calculate_accuracy(predicted, actual, prediction_type):
    """
    Type-specific accuracy calculation
    """
    if prediction_type == "location":
        if predicted == actual:
            return 1.0  # Exact match
        elif rooms_adjacent(predicted, actual):
            return 0.7  # Close but not exact
        elif same_floor(predicted, actual):
            return 0.3  # Same floor
        else:
            return 0.0  # Wrong
    
    elif prediction_type == "activity":
        if predicted == actual:
            return 1.0
        elif same_category(predicted, actual):
            return 0.7  # e.g., both gaming
        elif related(predicted, actual):
            return 0.5
        else:
            return 0.0
    
    # Default: exact match only
    return 1.0 if predicted == actual else 0.0
```

## Edge Cases to Handle

1. **Multiple people, ambiguous signals**: Use AI reasoning
2. **Guest profile on Tesla**: Correlate with iPhone departures
3. **Gaming PC logged in but idle**: Confidence decay over time
4. **Xbox streaming (unknown who)**: Cross-reference with other signals
5. **Budget exhausted**: Fall back to pattern match or local rules
6. **New activity from AI**: Validate, add to known activities if accurate
7. **Pattern becomes unreliable**: Mark inactive, stop using
8. **Conflicting signals**: Weight by confidence, use most recent
