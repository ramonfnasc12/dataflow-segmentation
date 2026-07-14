# Context

## Glossary

| Term | Definition |
|------|-----------|
| **Session** | A bounded sequence of tracking events from a single visitor, identified by `session_id`. Demarcated by a 30-minute inactivity timeout (processing-time timer in Beam). Can be revived from MongoDB if the same session-id reappears after state is cleared. |
| **Session Summary** | A compact derived state object per session containing aggregates, timing, sequence progress, event flags, historical context, and current segment assignments. Never contains raw events. Bounded by the feature schema, not event count (~1 KB per session). |
| **Feature Schema** | A versioned configuration document (in MongoDB) that defines what the pipeline tracks per session: which counters, flags, sequences, and timing features to maintain. Decoupled from segmentation rules — serves both rules and future ML. |
| **Segment** | A classification assigned to a session based on visitor behavior. Non-exclusive — a session can match multiple segments simultaneously. Continuously re-evaluated on every event; can be assigned or revoked mid-session. |
| **Rule** | A structured document in MongoDB defining conditions for segment assignment. A recursive boolean tree (AND/OR/NOT) of leaf conditions: aggregation, timing, flag, sequence, negation, historical. Pinned to a version per session. Declares `min_feature_schema_version` for compatibility. |
| **Sequence** | An ordered pattern of event types (with optional property filters) defined in the feature schema. Tracked via timestamps per step using earliest-match semantics. Order validation is on client timestamps, not arrival order — handles out-of-order events without buffering. |
| **Session Close** | When the inactivity timer fires (30 min no events): pipeline writes full summary to MongoDB (sessions + customer_sessions) and clears all Beam state for that key. Only time MongoDB is written during a session's lifecycle. |
| **Session Revival** | The process of loading a previously persisted session summary from MongoDB when a new event arrives for a session-id with no active Beam state. Uses pinned schema and rules versions from the stored document. |
| **Customer Sessions** | A MongoDB collection mapping customer-id to current session-id and last segment assignments from their most recent closed session. Used for historical lookups on session start. |
| **Segment Change Event** | An output element yielded by the Stateful DoFn when `current_segments` changes. Published to an output Pub/Sub topic by a downstream WriteToPubSub step. Contains `segment_assigned` or `segment_revoked` type. |
| **Side Input** | Beam mechanism for providing slowly-changing reference data to all workers. Used for feature schemas and rules. Refreshed periodically (every 5 min). |
| **Dead Letter Topic** | A Pub/Sub topic receiving events that fail parsing/validation. Prevents pipeline crashes from malformed input. |

## Key Architecture Decisions

1. **No raw events in state** — only the session summary (effect of events). Bounds memory by rule/schema complexity, not event volume.
2. **No mid-session MongoDB writes** — state lives in Beam only during the session. MongoDB is written once on session close.
3. **Direct Pub/Sub publish from Dataflow** — no CDC/outbox pattern. Segment changes flow directly to output topic as a separate pipeline step (not inside the DoFn).
4. **Feature schema decoupled from rules** — enables ML consumption of the same summary without depending on rule definitions.
5. **Version pinning per session** — schema and rules versions are fixed at session start. No mid-session config changes.
6. **Earliest-match sequence tracking** — handles out-of-order event arrival via client timestamps without buffering or reordering.
