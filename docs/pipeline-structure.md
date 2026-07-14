# Pipeline Structure

## Purpose

This document describes how the Apache Beam pipeline is wired together — the transforms, their connections, and data flow between them. Implementation details of the Stateful DoFn internals are in [feature-schema.md](feature-schema.md) and [rule-engine.md](rule-engine.md).

---

## Pipeline Topology

```
ReadFromPubSub (input topic)
    |
    v
ParseAndValidate (ParDo)
    |--- invalid --> WriteToPubSub (dead letter topic)
    |
    v [valid events]
KeyBySessionId (Map)
    |
    v [(session_id, event) pairs]
SegmentationDoFn (Stateful ParDo, keyed by session_id)
    |--- "segment_change" --> WriteToPubSub (output topic)
    |
    v [no output for non-matching events]

Side Inputs:
    - feature_schemas (periodic refresh from MongoDB)
    - rules (periodic refresh from MongoDB)
```

---

## Pipeline Code (High-Level)

```python
def run_pipeline(options: PipelineOptions) -> None:
    with beam.Pipeline(options=options) as pipeline:
        # Side Inputs: refreshed periodically
        feature_schemas = load_feature_schemas_side_input(pipeline)
        rules = load_rules_side_input(pipeline)

        # Main pipeline
        events = (
            pipeline
            | "ReadFromPubSub" >> beam.io.ReadFromPubSub(
                topic=options.input_topic,
                id_label="event_id",  # Beam-level dedup
            )
            | "ParseAndValidate" >> beam.ParDo(
                ParseAndValidateDoFn()
            ).with_outputs("valid", "invalid")
        )

        # Dead letter
        events.invalid | "DeadLetter" >> beam.io.WriteToPubSub(
            topic=options.dead_letter_topic
        )

        # Segmentation
        segment_events = (
            events.valid
            | "KeyBySession" >> beam.Map(
                lambda e: (e.session_id, e)
            )
            | "Segment" >> beam.ParDo(
                SegmentationDoFn(),
                feature_schemas=beam.pvalue.AsSingleton(feature_schemas),
                rules=beam.pvalue.AsSingleton(rules),
            )
        )

        # Output
        segment_events | "PublishSegments" >> beam.io.WriteToPubSub(
            topic=options.output_topic
        )
```

---

## Transforms

### 1. ReadFromPubSub

```python
beam.io.ReadFromPubSub(
    topic="projects/{project}/topics/{input_topic}",
    id_label="event_id",
)
```

- `id_label="event_id"` enables Beam's built-in dedup (first layer)
- Produces raw bytes from Pub/Sub messages
- Unbounded source — pipeline runs in streaming mode

### 2. ParseAndValidateDoFn

Responsibilities:
- Deserialize JSON
- Validate schema (required fields, types)
- Extract `session_id`, `customer_id`, `event_type`, `client_ts`, `properties`
- Bot filtering (known user-agent patterns)
- Route invalid events to dead letter output

```python
class ParseAndValidateDoFn(beam.DoFn):
    VALID = "valid"
    INVALID = "invalid"

    def process(self, raw_bytes: bytes):
        try:
            event = parse_event(raw_bytes)
            validate_schema(event)

            if is_known_bot(event):
                return  # drop silently

            yield beam.pvalue.TaggedOutput(self.VALID, event)

        except (ValidationError, ParseError) as exc:
            yield beam.pvalue.TaggedOutput(
                self.INVALID,
                DeadLetterRecord(raw=raw_bytes, error=str(exc)),
            )
```

### 3. KeyBySessionId

Simple map that creates keyed pairs for stateful processing:

```python
beam.Map(lambda event: (event.session_id, event))
```

After this step, Beam can colocate all events for the same session on the same worker.

### 4. SegmentationDoFn (Stateful)

The core of the pipeline. Manages per-session state and evaluates rules.

**State specs:**

```python
class SegmentationDoFn(beam.DoFn):
    # Session summary (aggregates, timing, flags, sequences, segments)
    SUMMARY = ReadModifyWriteStateSpec('summary', SessionSummaryCoder())

    # Pinned versions
    SCHEMA_VERSION = ReadModifyWriteStateSpec('schema_v', VarIntCoder())
    RULES_VERSION = ReadModifyWriteStateSpec('rules_v', VarIntCoder())

    # Inactivity timer
    EXPIRY_TIMER = TimerSpec('expiry', TimeDomain.PROCESSING_TIME)
```

**Process method (simplified):**

```python
def process(
    self,
    element,
    summary_state=beam.DoFn.StateParam(SUMMARY),
    schema_v_state=beam.DoFn.StateParam(SCHEMA_VERSION),
    rules_v_state=beam.DoFn.StateParam(RULES_VERSION),
    expiry_timer=beam.DoFn.TimerParam(EXPIRY_TIMER),
    feature_schemas=beam.DoFn.SideInputParam(...),
    rules=beam.DoFn.SideInputParam(...),
):
    session_id, event = element
    summary = summary_state.read()

    if summary is None:
        # New or revived session
        summary, schema_version, rules_version = initialize_session(
            session_id, event, feature_schemas, rules
        )
        schema_v_state.write(schema_version)
        rules_v_state.write(rules_version)
    else:
        schema_version = schema_v_state.read()
        rules_version = rules_v_state.read()

    # Get pinned configs
    schema = get_by_version(feature_schemas, schema_version)
    active_rules = get_rules_for_version(rules, rules_version, schema_version)

    # Update summary
    previous_segments = summary.current_segments.copy()
    update_summary(summary, event, schema)

    # Evaluate rules
    new_segments = evaluate_rules(summary, active_rules, schema_version)

    # Detect changes
    added = new_segments - previous_segments
    removed = previous_segments - new_segments

    if added or removed:
        summary.current_segments = new_segments
        yield SegmentChangeEvent(
            session_id=session_id,
            customer_id=summary.customer_id,
            added_segments=added,
            removed_segments=removed,
            timestamp=event.client_ts,
        )

    # Persist updated summary and reset timer
    summary_state.write(summary)
    expiry_timer.set(Timestamp.now() + Duration(seconds=SESSION_TIMEOUT_SEC))
```

**Timer callback (session close):**

```python
@on_timer(EXPIRY_TIMER)
def on_expiry(
    self,
    summary_state=beam.DoFn.StateParam(SUMMARY),
    schema_v_state=beam.DoFn.StateParam(SCHEMA_VERSION),
    rules_v_state=beam.DoFn.StateParam(RULES_VERSION),
):
    summary = summary_state.read()
    if summary is None:
        return

    # Write to MongoDB
    write_session_to_mongodb(summary)
    update_customer_sessions(summary)

    # Clear state
    summary_state.clear()
    schema_v_state.clear()
    rules_v_state.clear()
```

### 5. WriteToPubSub (output)

```python
beam.io.WriteToPubSub(
    topic="projects/{project}/topics/{output_topic}"
)
```

- Beam handles batching automatically
- At-least-once delivery (Pub/Sub semantics)
- Downstream consumers deduplicate by `(session_id, segment_id, timestamp)`

---

## Side Input Loading

### Strategy: Periodically-Triggered Global Window

```python
def load_feature_schemas_side_input(pipeline):
    return (
        pipeline
        | "SchemaTick" >> beam.io.ReadFromPubSub(topic=TICK_TOPIC)
        | "SchemaWindow" >> beam.WindowInto(
            beam.window.GlobalWindows(),
            trigger=beam.trigger.Repeatedly(
                beam.trigger.AfterProcessingTime(REFRESH_INTERVAL_SEC)
            ),
            accumulation_mode=beam.trigger.AccumulationMode.DISCARDING,
        )
        | "LoadSchemas" >> beam.ParDo(LoadFeatureSchemasFromMongoDB())
    )

def load_rules_side_input(pipeline):
    return (
        pipeline
        | "RulesTick" >> beam.io.ReadFromPubSub(topic=TICK_TOPIC)
        | "RulesWindow" >> beam.WindowInto(
            beam.window.GlobalWindows(),
            trigger=beam.trigger.Repeatedly(
                beam.trigger.AfterProcessingTime(REFRESH_INTERVAL_SEC)
            ),
            accumulation_mode=beam.trigger.AccumulationMode.DISCARDING,
        )
        | "LoadRules" >> beam.ParDo(LoadRulesFromMongoDB())
    )
```

**Refresh interval:** 5 minutes (configurable). New sessions pick up new versions; existing sessions continue with their pinned version regardless of refresh.

**Alternative:** Trigger refresh via a dedicated Pub/Sub topic when configs change in MongoDB. Lower latency for rule updates but more infrastructure.

---

## Session Initialization

```python
def initialize_session(
    session_id: str,
    event: TrackingEvent,
    feature_schemas: FeatureSchemaSet,
    rules: RuleSet,
) -> tuple[SessionSummary, int, int]:
    # Try revival from MongoDB
    existing = read_session_from_mongodb(session_id)

    if existing is not None:
        # Revival: use pinned versions from stored session
        schema_version = existing.feature_schema_version
        rules_version = existing.rules_version
        summary = existing.to_summary()
        summary.status = "active"
        return summary, schema_version, rules_version

    # New session: pin to latest versions
    schema = get_latest_active(feature_schemas)
    schema_version = schema.version
    rules_version = get_latest_compatible_rules_version(rules, schema_version)

    # Load historical context
    historical = load_historical_context(event.customer_id)

    summary = SessionSummary(
        session_id=session_id,
        customer_id=event.customer_id,
        status="active",
        started_at=event.client_ts,
        event_count=0,
        aggregates={},
        timing=TimingState(),
        flags={},
        sequences={},
        current_segments=set(),
        historical=historical,
    )

    return summary, schema_version, rules_version
```

---

## MongoDB Interactions

All MongoDB I/O happens in two places only:

| When | Operation | Collection | Frequency |
|------|-----------|-----------|-----------|
| Session initialization (new/revival) | Read | `sessions`, `customer_sessions` | On first event for cold key (~10-100/s) |
| Session close (timer fires) | Write | `sessions`, `customer_sessions` | On session timeout (~5-30/s) |
| Side Input refresh | Read | `feature_schemas`, `rules` | Every 5 minutes (all workers) |

No MongoDB I/O happens during normal event processing (between initialization and close).

---

## Output Event Schema

Published to the output Pub/Sub topic:

```json
{
  "type": "segment_assigned",
  "session_id": "session_id_abc123",
  "customer_id": "42",
  "segment_id": "heavy_browser",
  "timestamp": "2026-07-08T10:15:30.123Z",
  "pipeline_ts": "2026-07-08T10:15:30.456Z"
}
```

```json
{
  "type": "segment_revoked",
  "session_id": "session_id_abc123",
  "customer_id": "42",
  "segment_id": "quick_browser",
  "timestamp": "2026-07-08T10:15:30.123Z",
  "pipeline_ts": "2026-07-08T10:15:30.456Z"
}
```

Fields:
- `type`: `segment_assigned` or `segment_revoked`
- `session_id`: the session where the change was detected
- `customer_id`: may be null for anonymous visitors
- `segment_id`: which segment was assigned or revoked
- `timestamp`: client timestamp of the event that triggered the change
- `pipeline_ts`: processing time when the pipeline emitted this event

---

## Error Handling

| Failure | Behavior |
|---------|----------|
| Parse/validate failure | Event routed to dead letter topic |
| MongoDB read failure (revival) | Start fresh session (log warning) |
| MongoDB write failure (session close) | Retry with exponential backoff; state remains in Beam |
| Rule evaluation error (bad rule) | Skip that rule, log error, continue with remaining rules |
| Pub/Sub publish failure | Beam retries the bundle (at-least-once) |

The pipeline never crashes from bad input or transient external failures.

---

## MongoDB Connection Management

> Reference: [Apache Beam Master Architecture Blueprint](files/apache_beam_master_architecture_blueprint.pdf), Sections 6.1–6.3

### Python Runtime: Process-Level Isolation

This pipeline runs on Python (Apache Beam Python SDK on Dataflow). Python's GIL forces Beam to use **multiple OS processes** (typically 1 per vCPU) on each worker VM. Processes cannot share memory, so each process maintains its own connection pool.

```
Maximum MongoDB Connections = Cluster Nodes × vCPU Cores Per VM × Pool Size Per Process
```

**Example:** 4 worker nodes × 8 vCPUs × 2 connections per pool = **64 simultaneous MongoDB connections.**

This is critical for sizing your MongoDB Atlas cluster — connection limits must account for this multiplication factor.

### Connection Lifecycle in DoFn

External connections must follow Beam's DoFn lifecycle methods:

```python
class SegmentationDoFn(beam.DoFn):

    def setup(self):
        """Called once per DoFn instance (once per OS process).
        Establish the MongoDB connection client here."""
        self._mongo_client = create_mongo_client(
            connection_string=self._connection_string,
            max_pool_size=2,
            server_selection_timeout_ms=5000,
        )
        self._db = self._mongo_client[self._database_name]

    def teardown(self):
        """Called when this DoFn instance is being destroyed.
        Close the client — but ONLY this instance's client."""
        if self._mongo_client:
            self._mongo_client.close()
```

### The `apache_beam.utils.shared.Shared` Pattern

For Python, use `Shared` to ensure a single connection client per process (avoiding duplicate clients across multiple DoFn instances in the same process):

```python
import apache_beam as beam
from apache_beam.utils.shared import Shared

class SegmentationDoFn(beam.DoFn):

    def __init__(self, connection_string: str, database: str):
        self._connection_string = connection_string
        self._database_name = database
        self._shared_client = Shared()

    def setup(self):
        def create_client():
            from pymongo import MongoClient
            return MongoClient(
                self._connection_string,
                maxPoolSize=2,
                serverSelectionTimeoutMS=5000,
                connectTimeoutMS=5000,
                socketTimeoutMS=10000,
            )

        self._mongo_client = self._shared_client.acquire(create_client)
        self._db = self._mongo_client[self._database_name]
```

`Shared.acquire()` guarantees: one client instance per process, reused across all DoFn instances within that process. The creation function is called only once; subsequent `acquire()` calls return the cached reference.

### Connection Isolation Between Pipeline Stages

The Stateful DoFn (session revival/close) and the Side Input loaders (feature schemas, rules) **will not share the same connection pool**. This is by design:

1. **Code encapsulation:** Each DoFn manages its own `@setup` lifecycle independently
2. **Pipeline shuffling:** Stateful DoFns and downstream sinks may run on different workers after a shuffle boundary — memory-level sharing is physically impossible across network hops

**Implication:** Size your MongoDB Atlas connection limit accounting for connections from:
- Stateful DoFn processes (the majority)
- Side Input loader processes (fewer, periodic bursts)

### Connection Sizing Formula

```
Required Atlas Connection Limit ≥
    (Worker Nodes × vCPUs × Pool Size)     [Stateful DoFn]
  + (Worker Nodes × vCPUs × 1)             [Side Input loaders, worst case]
  + Buffer (20%)

Example (baseline):
    4 nodes × 8 vCPUs × 2 pool = 64
  + 4 nodes × 8 vCPUs × 1     = 32
  + 20% buffer                  = 19
  = ~115 connections minimum

Example (peak / autoscaled):
    8 nodes × 8 vCPUs × 2 pool = 128
  + 8 nodes × 8 vCPUs × 1     = 64
  + 20% buffer                  = 38
  = ~230 connections minimum
```

MongoDB Atlas M30+ clusters support 1500+ connections, so this is well within limits. For M10/M20 tiers (500 connections), monitor usage during autoscaling events.

### The Teardown Pool Destruction Trap

> ⚠️ **Never close a shared connection pool inside `@teardown`.**

Because Beam destroys individual DoFn instances dynamically as traffic fluctuates, a single teardown call would destroy the shared client for all other active DoFn instances in the same process. The `Shared` pattern avoids this by reference-counting — the actual client is only destroyed when the last reference is released (process exit).

If you're NOT using `Shared`, isolate teardown to instance-level clients only:

```python
def teardown(self):
    # Safe: closes only this instance's client
    if self._mongo_client and not self._is_shared:
        self._mongo_client.close()
```

### Java vs Python: Why This Matters

This pipeline uses Python, but for reference:

| Aspect | Java Runtime | Python Runtime (this pipeline) |
|--------|-------------|-------------------------------|
| Concurrency model | Multi-threaded (single JVM) | Multi-processed (1 per vCPU) |
| Connection sharing | Static class variable shared across all threads | `Shared()` cache per process |
| Connection formula | Nodes × Pool Size | Nodes × vCPUs × Pool Size |
| Pool library | HikariCP (static singleton) | PyMongo internal pool via `Shared()` |
| Connection count (4 nodes, 8 vCPU, pool=2) | 4 × 2 = **8** | 4 × 8 × 2 = **64** |

**The Python runtime uses 8x more connections for the same cluster size.** This is the most important sizing consideration when configuring MongoDB Atlas connection limits.

---

## Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `SESSION_TIMEOUT_SEC` | 1800 (30 min) | Inactivity timeout before session close |
| `REFRESH_INTERVAL_SEC` | 300 (5 min) | Side Input refresh period |
| `MAX_REASONABLE_GAP_SEC` | 1800 | Max gap to count toward time-on-page |
| `MONGODB_READ_TIMEOUT_MS` | 5000 | Timeout for session revival read |
| `MONGODB_WRITE_RETRIES` | 3 | Retry attempts for session close write |
| `MONGODB_MAX_POOL_SIZE` | 2 | Connections per process per pool |
| `MONGODB_CONNECT_TIMEOUT_MS` | 5000 | Timeout for establishing new connections |
| `MONGODB_SOCKET_TIMEOUT_MS` | 10000 | Timeout for socket operations |
