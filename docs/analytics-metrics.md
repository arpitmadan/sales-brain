# Call Analytics Metrics

All metrics are derived from Zoom's VTT transcript, which includes per-utterance speaker labels and millisecond-accurate timestamps. No third-party speech analysis service is required.

## Metrics

### Talk Ratio
**Definition:** Percentage of the call duration during which the rep was speaking.

**Formula:**
```
talk_ratio = sum(rep_utterance_duration) / total_call_duration * 100
```

**Target range:** 40–60% (rep talks less than the prospect in a good discovery)

**Flag logic:**
- Above range (>60%): rep is dominating — not enough listening
- Within range: good balance
- Below range (<40%): rep may not be driving the conversation

---

### Longest Monologue
**Definition:** The longest uninterrupted stretch of rep speech without the prospect speaking.

**Formula:** Max duration of consecutive rep utterances where no prospect utterance falls within the gap.

**Target range:** <2 min

**Flag logic:**
- Above range: rep is pitching without checking in — prospect may be checked out

---

### Longest Customer Story
**Definition:** The longest uninterrupted stretch of prospect speech.

**Formula:** Same as monologue, but for prospect utterances.

**Target range:** >1 min (prospects should be telling their story)

**Flag logic:**
- Below range: rep may not be asking open-ended questions or is cutting the prospect off

---

### Interactivity
**Definition:** How conversational the call is — measured as speaker switches per hour.

**Formula:**
```
interactivity = speaker_switch_count / (call_duration_minutes / 60)
```

**Target range:** 8–14 switches/hour for a demo or disco

**Flag logic:**
- Low interactivity: one-sided conversation
- High interactivity: could indicate interruptions or a very back-and-forth negotiation

---

### Patience
**Definition:** How long the rep waits after the prospect finishes speaking before responding. Measured in seconds.

**Formula:** Average gap between end of a prospect utterance and start of the next rep utterance, across all transitions.

**Target range:** >1.0 sec

**Flag logic:**
- Below range: rep is interrupting or jumping in too fast — prospect doesn't feel heard
- Very high: may indicate disengagement or awkward silences

---

## Configurable Benchmarks

Each metric's "within range" thresholds are stored in the database and can be updated per call type:

| Call Type | Talk Ratio | Monologue | Customer Story | Interactivity | Patience |
|-----------|-----------|-----------|---------------|--------------|---------|
| Discovery | 40–55% | <90 sec | >60 sec | 10–16 | >1.2 sec |
| Demo | 50–65% | <120 sec | >30 sec | 8–14 | >0.8 sec |
| Renewal | 45–60% | <90 sec | >45 sec | 8–12 | >1.0 sec |

These are starting defaults — adjust based on what you observe from your top performers.

---

## Implementation Notes

Zoom VTT format per utterance:
```
WEBVTT

00:00:05.123 --> 00:00:09.456
[Speaker_1]: Hey, thanks for joining today...
```

The parser:
1. Reads the VTT file from the Zoom API response
2. Maps `Speaker_1`, `Speaker_2`, etc. to participants via the meeting participants endpoint
3. Identifies the rep as the Zoom host (internal user)
4. Builds an array of `{ speaker, start_ms, end_ms, text }` objects
5. Runs each metric computation against that array
