# Rule Engine Design

## Purpose

The rule engine evaluates segmentation rules against a session summary. It is a pure function: given a summary and a set of rules, it returns the set of matched segment IDs. No side effects, no I/O, no state mutation.

---

## Rule Structure

Rules are stored in the `rules` MongoDB collection. Each rule is a recursive boolean condition tree:

```javascript
{
  _id: ObjectId("..."),
  segment_id: "high_intent_returner",
  name: "High Intent Returning Visitor",
  version: 3,
  active: true,
  min_feature_schema_version: 4,
  conditions: {
    operator: "AND",
    rules: [
      { type: "historical", field: "total_sessions", condition: "gte", value: 2 },
      { type: "aggregation", field: "product_views_count", comparator: "gte", value: 5 },
      { type: "timing", field: "total_duration_sec", comparator: "lte", value: 180 },
      { type: "sequence", name: "browse_to_cart", condition: "completed_within_sec", value: 120 },
      { type: "flag", field: "has_completed_purchase", condition: "false" },
      {
        operator: "OR",
        rules: [
          { type: "aggregation", field: "campaign_page_views_count", comparator: "gte", value: 1 },
          { type: "historical", field: "last_segments", condition: "contains", value: "casual_visitor" }
        ]
      }
    ]
  }
}
```

---

## Evaluation Overview

```
evaluate_rules(summary, rules, schema_version)
    |
    for each rule:
    |   skip if min_feature_schema_version > schema_version
    |   evaluate_condition(rule.conditions, summary)
    |       |
    |       Branch node (AND/OR/NOT)?
    |       |   -> recurse into sub-rules
    |       |
    |       Leaf node?
    |       |   -> evaluate specific condition type
    |       |       -> single dict lookup + comparison
    |
    return set of matched segment_ids
```

---

## Condition Types

### 1. Aggregation

Compares a counter in `summary.aggregates` against a threshold.

```javascript
{ "type": "aggregation", "field": "product_views_count", "comparator": "gte", "value": 10 }
```

**Resolution:** `summary.aggregates.get(field, 0) <comparator> value`

**Supported comparators:** `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `in`, `not_in`

### 2. Timing

Compares a timing feature against a threshold.

```javascript
{ "type": "timing", "field": "total_duration_sec", "comparator": "gte", "value": 300 }
```

**Resolution:** Looks up `field` in `summary.timing` (either top-level or in `durations` dict).

**Available fields:**
- `total_duration_sec` — seconds since session start
- `time_since_last_event_sec` — gap since previous event
- Any `time_on_event_type` field defined in the feature schema (e.g., `time_on_product_pages_sec`)

### 3. Flag

Checks a boolean marker.

```javascript
{ "type": "flag", "field": "has_added_to_cart", "condition": "true" }
```

**Resolution:** `summary.flags.get(field, False) == (condition == "true")`

**Conditions:** `true`, `false`

### 4. Sequence

Checks the progress or completion of an ordered event pattern.

```javascript
{ "type": "sequence", "name": "campaign_influenced", "condition": "completed" }
```

**Conditions:**

| Condition | Meaning |
|-----------|---------|
| `completed` | All steps filled AND timestamps strictly increasing |
| `not_completed` | Sequence has not yet completed |
| `match_count_gte` | Sequence completed at least N times (with `value: N`) |
| `steps_filled_gte` | At least N steps have timestamps (partial progress, with `value: N`) |
| `completed_within_sec` | Completed AND (max_ts - min_ts) <= N seconds (with `value: N`) |

### 5. Negation

Asserts that something did NOT happen, optionally gated by a precondition.

```javascript
{
  "type": "negation",
  "event_type": "search",
  "after": { "type": "aggregation", "field": "product_views_count", "comparator": "gte", "value": 1 },
  "condition": "never_occurred"
}
```

**Resolution:**
1. Evaluate `after` clause (if present) — if false, negation is false (precondition not met)
2. Look up the count for `event_type` in aggregates
3. Apply condition (`never_occurred` → count == 0, `occurred_less_than` → count < value)

**Requires:** The negated event type must have a corresponding aggregate in the feature schema.

### 6. Historical

Checks cross-session context loaded from `customer_sessions` at session start.

```javascript
{ "type": "historical", "field": "last_segments", "condition": "contains", "value": "casual_visitor" }
```

**Available fields:**
- `last_segments` — list of segments from the visitor's previous session
- `total_sessions` — how many sessions this visitor has had
- `days_since_last_session` — computed at session start

**Conditions:** `contains`, `not_contains`, `eq`, `neq`, `gt`, `gte`, `lt`, `lte`

---

## Boolean Combinators

### AND

All sub-conditions must be true. Short-circuits on first false.

```javascript
{ "operator": "AND", "rules": [...] }
```

### OR

At least one sub-condition must be true. Short-circuits on first true.

```javascript
{ "operator": "OR", "rules": [...] }
```

### NOT

Inverts a single condition. Wraps exactly one sub-condition.

```javascript
{ "operator": "NOT", "rules": [{ "type": "flag", "field": "has_completed_purchase", "condition": "true" }] }
```

These nest arbitrarily: `AND(OR(A, B), NOT(C), D)` is valid.

---

## Segment Semantics

### Non-exclusive

A session can match multiple segments simultaneously. Rules are independent — matching "heavy_browser" doesn't prevent matching "high_intent_returner".

### Continuous re-evaluation

Rules are evaluated on **every event**. Segments can be assigned AND revoked mid-session:

- Event 50: `total_duration_sec = 50` → matches "quick_browser" (duration < 60s)
- Event 80: `total_duration_sec = 120` → no longer matches "quick_browser" → **revoked**

The caller diffs previous segments against new evaluation results to detect changes.

### No priority / mutual exclusion at this layer

If business logic requires "a visitor can only be in one segment," that's enforced downstream (Customer 360 API), not in the pipeline. The pipeline reports all matches.

---

## Performance Optimizations

### Short-circuit evaluation

Python's `all()` / `any()` stop on first failure/success. Place cheap, likely-to-fail conditions first in AND groups.

### Load-time condition sorting

When rules are loaded from MongoDB (via Side Input), sort AND-group leaves by cost:

| Priority | Condition type | Cost |
|----------|---------------|------|
| 0 | flag | Single boolean lookup |
| 1 | aggregation | Single integer lookup + compare |
| 2 | timing | Single float lookup + compare |
| 3 | sequence | Multiple timestamp checks |
| 4 | negation | Evaluates a sub-condition |
| 5 | historical | Struct field access |

This is applied once at load time, not per evaluation.

### Skip unchanged rules

If a rule's condition types don't overlap with what changed in the summary update, it can be skipped. For example, if only a timing field changed, rules that use only aggregation + flag conditions can't have changed their result.

This is an optional optimization — the baseline (evaluate all) is already fast enough at ~120 comparisons per event.

---

## Rule Versioning

### Version pinning per session

- When a session starts, it pins to the current set of active rules
- The pinned `rules_version` is stored in Beam state and in the MongoDB session document
- All evaluations for that session use the same rules for its entire lifetime

### Why not update rules mid-session?

A rule change mid-session could:
- Cause a segment to be assigned AND revoked in the same session (confusing for downstream)
- Invalidate sequence progress (if the sequence definition changed)
- Create non-deterministic behavior on replay (Beam retries would produce different results)

Pinning eliminates these problems. Since sessions are short-lived (30 min max), new rules take effect within 30 minutes for all visitors.

### Deploying new rules

1. Insert new rule document (or new version of existing rule) in MongoDB
2. Set `active: true` on new version, `active: false` on old
3. The Side Input refreshes (next periodic trigger)
4. New sessions pick up the new rules; existing sessions continue with pinned version

No pipeline redeployment needed.

---

## Example: Full Rule Evaluation Trace

**Rule:** "High Intent Returner"
```
AND(
  historical.total_sessions >= 2,
  aggregation.product_views_count >= 5,
  timing.total_duration_sec <= 180,
  sequence.browse_to_cart completed_within_sec 120,
  flag.has_completed_purchase == false
)
```

**Summary state:**
```python
{
  "aggregates": {"product_views_count": 7, "searches_count": 2},
  "timing": {"total_duration_sec": 95},
  "flags": {"has_added_to_cart": True, "has_completed_purchase": False},
  "sequences": {"browse_to_cart": {"completed": True, "step_timestamps": [t=10, t=45, t=80]}},
  "historical": {"total_sessions": 3, "last_segments": ["casual_visitor"]}
}
```

**Evaluation (sorted by cost):**

| Step | Condition | Lookup | Result |
|------|-----------|--------|--------|
| 1 | flag: has_completed_purchase == false | `flags["has_completed_purchase"]` = False | True |
| 2 | aggregation: product_views >= 5 | `aggregates["product_views_count"]` = 7 | True |
| 3 | timing: duration <= 180 | `timing["total_duration_sec"]` = 95 | True |
| 4 | sequence: browse_to_cart completed_within_sec 120 | completed=True, span=(80-10)=70s <= 120 | True |
| 5 | historical: total_sessions >= 2 | `historical["total_sessions"]` = 3 | True |

**Result:** All true → segment "high_intent_returner" matched.
