# Real-Time Customer Segmentation Pipeline

A real-time customer segmentation pipeline built on **GCP Dataflow** (Apache Beam, Python) with **MongoDB Atlas** as the durable storage and configuration layer.

The pipeline processes 4k–8k tracking events/second from an e-commerce platform and assigns visitors to behavioral segments within 2 seconds end-to-end.

## How It Works

1. Tracking events arrive via **Pub/Sub**
2. A stateful Dataflow pipeline maintains a compact **session summary** per visitor (~1 KB)
3. On every event, a **rule engine** evaluates segment conditions against the summary
4. Segment changes are published to an output **Pub/Sub topic** in real-time
5. On session close (30 min inactivity), the summary is persisted to **MongoDB Atlas**

```
Pub/Sub (input) → Parse/Validate → Stateful DoFn → Pub/Sub (segment changes)
                                        ↕
                                  MongoDB Atlas
                              (sessions, rules, schemas)
```

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| No raw events in state | Memory bounded by schema complexity (~1 KB), not event volume |
| No mid-session MongoDB writes | Reduces write pressure from ~4K/s to ~5-30/s |
| Feature schema decoupled from rules | Enables ML consumption without rule dependency |
| Version pinning per session | Consistent evaluation, deterministic on Beam retries |
| Earliest-match sequence tracking | Handles out-of-order events without buffering |
| Direct Pub/Sub publish | Beam bundle atomicity eliminates dual-write risk |

## Architecture

See [docs/architecture-overview.md](docs/architecture-overview.md) for the full architecture with diagrams.

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture Overview](docs/architecture-overview.md) | End-to-end architecture, diagrams, design decisions, scalability profile |
| [Pipeline Structure](docs/pipeline-structure.md) | Beam transforms, DoFn lifecycle, side inputs, MongoDB connection management |
| [Feature Schema](docs/feature-schema.md) | What the pipeline tracks per session: aggregates, flags, sequences, timing |
| [Rule Engine](docs/rule-engine.md) | How segmentation rules are structured and evaluated |
| [Context](CONTEXT.md) | Glossary and key architecture decisions summary |

## Tech Stack

- **Processing:** GCP Dataflow (Apache Beam, Python SDK)
- **Messaging:** GCP Pub/Sub (input and output)
- **Storage:** MongoDB Atlas (sessions, customer history, configuration)
- **Rules & Schemas:** Stored as MongoDB documents, loaded as Beam Side Inputs

## Scalability

| Metric | Baseline | Peak |
|--------|----------|------|
| Events/second | 4,000 | 8,000 |
| Concurrent sessions | ~100,000 | ~200,000 |
| State per session | ~1 KB | ~1 KB |
| Total Beam state | ~100-200 MB | ~200-400 MB |
| End-to-end latency | ~170ms | ~360ms |

## Future: ML Integration

The session summary is designed as a feature vector from day one. The transition path:

1. **Current** — Rule-based evaluation
2. **Shadow mode** — ML runs in parallel, results logged to BigQuery for offline comparison
3. **A/B test** — Split by session_id hash
4. **Full ML** — Model replaces rules via Beam's `RunInference` transform
