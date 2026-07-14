# Feature Schema Design

## Purpose

The feature schema defines **what the pipeline tracks** per session. It is the input contract that both rule-based evaluation and future ML models consume. It is decoupled from segmentation rules — the schema describes what to observe, rules describe what to conclude.

---

## Separation of Concerns

```
Feature schema defines -> what to track in summary
                               |
                      [session summary]
                         |           |
                    Rule engine    ML model
                         |           |
                    Segments      Segments
```

| Concern | Drives | Owned by |
|---------|--------|----------|
| Feature schema | What the DoFn tracks (counters, sequences, flags, timing) | Data/ML team |
| Segmentation rules | How to assign segments from tracked features | Business/UX team |

---

## Schema Structure

Stored in the `feature_schemas` MongoDB collection:

```javascript
{
  _id: ObjectId("..."),
  version: 4,
  active: true,
  created_at: ISODate("2026-07-10T..."),

  aggregates: [...],   // Event counters
  flags: [...],        // Boolean markers
  sequences: [...],    // Ordered event patterns
  timing: [...]        // Duration and time-between features
}
```

---

## Feature Types

### Aggregates (event counters)

Count occurrences of specific event types, optionally filtered by properties.

```javascript
aggregates: [
  { event_type: "product_page_view", field: "product_views_count" },
  { event_type: "search", field: "searches_count" },
  {
    event_type: "search",
    field: "non_empty_searches_count",
    filters: [{ property: "query_length", operator: "gt", value: 0 }]
  }
]
```

**Update logic:** On each event, iterate aggregate definitions. If event_type matches and filters pass, increment `summary.aggregates[field]` by 1.

**Memory cost:** One integer per definition. Typically 5-15 aggregates = 60-120 bytes.

### Flags (boolean markers)

Set to `true` once a specific event type occurs. Monotonic — once true, never reverts.

```javascript
flags: [
  { event_type: "add_to_cart", field: "has_added_to_cart" },
  { event_type: "purchase", field: "has_completed_purchase" },
  {
    event_type: "product_page_view",
    field: "has_viewed_premium",
    filters: [{ property: "tier", operator: "eq", value: "premium" }]
  }
]
```

**Update logic:** On match, set `summary.flags[field] = True`. Skip if already true.

**Memory cost:** One boolean per definition. Typically 3-10 flags = 10-40 bytes.

### Sequences (ordered event patterns)

Track whether a specific sequence of events has occurred in client-timestamp order. This is the feature type that preserves **event ordering** without storing raw events.

```javascript
sequences: [
  {
    name: "campaign_influenced",
    steps: [
      { event_type: "product_page_view" },
      { event_type: "search" },
      { event_type: "campaign_page_view" }
    ]
  },
  {
    name: "comparison_shopper",
    steps: [
      { event_type: "product_page_view", filters: [{ property: "category", operator: "eq", value: "electronics" }] },
      { event_type: "product_page_view", filters: [{ property: "category", operator: "eq", value: "electronics" }] },
      { event_type: "search", filters: [{ property: "query", operator: "contains", value: "compare" }] }
    ],
    distinct_required: [0, 1],
    allow_repeat: false
  }
]
```

**State shape per sequence:**

```python
{
  "step_timestamps": [datetime | None, ...],  # one per step
  "step_values": [dict | None, ...],          # captured properties for distinct checks
  "completed": bool,
  "match_count": int
}
```

**Update logic:**
1. For each sequence definition, check if event matches any step
2. If match: record client_ts using **earliest-match semantics** (only update if earlier than existing or slot is empty)
3. After update: check if all slots filled AND timestamps strictly increasing
4. If complete: set `completed = True`, increment `match_count`

**Why this handles out-of-order:** The check is on timestamps, not arrival order. A late-arriving event that fills an earlier slot self-corrects the sequence.

**Memory cost:** One timestamp + one optional dict per step per sequence. Typically 3-5 sequences x 3-4 steps = ~200-500 bytes.

### Known Sequence Limitations

The earliest-match semantics are optimal for the stated requirements ("events A, B, C in exactly this order") but define a clear expressiveness boundary:

| Scenario | Supported | Why |
|----------|-----------|-----|
| "product_view → search → campaign_page" in order | Yes | Core use case — timestamps checked strictly increasing |
| "at least 2 distinct product views then add_to_cart" | Yes | `distinct_required` field handles this |
| "**most recent** product view was in category X, then search" | No | Earliest-match always wins; latest timestamp is discarded |
| Sequences with optional/skippable steps | No | All steps must fill for completion |
| "A happened, then B, then A did NOT happen again after B" | No | Would require negation-within-sequence; current model only tracks presence |
| "A happened N times, then B" | No | Steps are positional slots, not repetition counters |

**Future extension path:** If the business needs "most recent match" semantics, add a `match_strategy` field to sequence definitions:

```javascript
{
  name: "recent_product_then_search",
  match_strategy: "latest",  // default: "earliest"
  steps: [
    { event_type: "product_page_view" },
    { event_type: "search" }
  ]
}
```

This stores the latest timestamp per step instead of earliest. It's an additive schema change — no architectural rework required. Existing sequences keep `"earliest"` behavior by default.

---

### Timing (duration features)

Track time-based features that require knowledge of the previous event.

```javascript
timing: [
  { type: "total_duration" },
  { type: "time_since_last_event" },
  { type: "time_on_event_type", event_type: "product_page_view", field: "time_on_product_pages_sec" }
]
```

**Types:**

| Type | Measures | State needed |
|------|----------|-------------|
| `total_duration` | Seconds since session start | `started_at` (already tracked) |
| `time_since_last_event` | Gap between current and previous event | `last_event_timestamp` |
| `time_on_event_type` | Cumulative time spent on a specific page type | `last_event_type` + `last_event_timestamp` |

**Update logic for `time_on_event_type`:** If the *previous* event was of the target type, the time gap between previous and current event is accumulated into the duration field. This measures "time spent on product pages" by assuming the user was viewing the product page until the next event arrived.

**Memory cost:** A few floats + previous event metadata. ~50-100 bytes.

---

## ML-Specific Feature Extensions (Future)

When ML is introduced, the schema can add features that rules don't consume:

```javascript
// Added in feature_schema v5+ for ML
{
  event_type_buffer: { size: 10 },  // last N event types (circular buffer)
  transitions: true                  // bigram counts: event_type_A -> event_type_B
}
```

**Event type buffer:** Fixed-size array of recent event types. No payloads, just type strings.

```python
# State: ["search", "product_view", "product_view", "campaign_page", ...]
# Memory: 10 strings x ~20 bytes = ~200 bytes
```

**Transition counts:** Count pairs of consecutive event types.

```python
# State: {"search->product_view": 3, "product_view->add_to_cart": 1, ...}
# Bounded by event_types^2 (typically < 100 pairs)
# Memory: ~100 pairs x ~40 bytes = ~4 KB (worst case)
```

These are bounded and don't require storing raw events — just the previous event type and a dictionary of pair counts.

---

## Versioning

### Rules

- Feature schemas are versioned with a monotonically increasing integer
- Only one version is `active: true` at any time
- Old versions remain in the collection for sessions that are pinned to them

### Pinning

- When a new session starts, it pins to the current active schema version
- When a session is revived from MongoDB, it uses the version stored in its document
- A session **never** changes schema version mid-flight

### Compatibility with Rules

Rules declare `min_feature_schema_version`:

```javascript
// This rule requires feature_schema >= 3
// (because searches_count was added in v3)
{
  segment_id: "search_heavy",
  min_feature_schema_version: 3,
  conditions: { type: "aggregation", field: "searches_count", comparator: "gte", value: 5 }
}
```

The rule engine skips rules whose `min_feature_schema_version` exceeds the session's pinned schema version.

---

## Schema Evolution

### Additive changes (safe)

Adding new aggregates, flags, sequences, or timing features:
1. Create new schema version (v4 -> v5)
2. Existing sessions continue on v4 — they don't track the new feature
3. New sessions start on v5 — they track everything
4. Gap is at most 30 minutes (max session length)

### Breaking changes (coordinated)

Removing or renaming a feature:
1. Update or deactivate rules that reference the old field
2. Wait for all active sessions on the old schema to close (~30 min)
3. Create new schema version without the field
4. Mark old version inactive

---

## Total Summary Memory Budget

| Feature type | Typical count | Memory per session |
|--------------|--------------|-------------------|
| Aggregates | 10-15 fields | ~120 bytes |
| Flags | 5-10 fields | ~40 bytes |
| Sequences | 3-5 sequences x 3-4 steps | ~500 bytes |
| Timing | 3-5 fields + metadata | ~100 bytes |
| Historical context | Fixed struct | ~100 bytes |
| Session metadata | Fixed fields | ~100 bytes |
| **Total** | | **~960 bytes (~1 KB)** |

At 200k concurrent sessions: **~200 MB total Beam state.** Well within Dataflow's capacity.
