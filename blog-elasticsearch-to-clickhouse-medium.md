# Replacing Elasticsearch with ClickHouse for OTEL Logs, Traces & Metrics: A 90% Cost-Reduction Migration

*A practical guide based on shipping this for a crypto-derivatives platform — annual observability bill went from high six figures to ~$50K, with faster queries and AI-powered log search as a bonus.*

---

## Why I'm writing this

If you're paying mid-six figures for Elasticsearch and ~90% of your queries are aggregations (error counts, latency percentiles, service health), you're paying full price for a feature you barely use — full-text search.

This post walks through how a crypto exchange replaced Elasticsearch with ClickHouse for OpenTelemetry logs, traces, and metrics. Same OTEL instrumentation, just a different backend. Result: 5× smaller storage footprint, 2-6× faster queries on benchmarks, and natural-language log queries via an AI agent — at ~10% of the cost.

If you have an existing Elasticsearch + Kibana observability stack and you've been wondering whether ClickHouse is a serious alternative, this is the deep-dive. Includes the schema, the migration plan, the OTEL Collector configuration, the cost numbers, and the gotchas.

> **Note on numbers**: The cost figures below ($400K Elasticsearch → ~$50K ClickHouse) are real annual numbers from this deployment, on log volumes typical of a high-traffic trading platform (low-tens of TB ingested per year, 90-day hot retention, multi-region). Your mileage will vary substantially with log volume, retention, and cluster size. The architectural patterns are universal.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The Problem — Elasticsearch at $400K/Year](#2-the-problem--elasticsearch-at-400kyear)
3. [Why ClickHouse for Observability](#3-why-clickhouse-for-observability)
4. [Platform Architecture](#4-platform-architecture)
5. [OpenTelemetry — The Standard We Keep](#5-opentelemetry--the-standard-we-keep)
5b. [Collection Agent Options — OTEL vs Fluent Bit vs Vector](#5b-collection-agent-options--otel-vs-fluent-bit-vs-vector)
6. [ClickHouse Schema — Logs, Traces, Metrics](#6-clickhouse-schema--logs-traces-metrics)
7. [End-to-End Distributed Tracing — HTTP to ClickHouse](#7-end-to-end-distributed-tracing--http-to-clickhouse)
8. [Full-Text Log Search — Replacing Kibana Discover](#8-full-text-log-search--replacing-kibana-discover)
9. [Visualization — Grafana Replaces Kibana](#9-visualization--grafana-replaces-kibana)
10. [Data Retention & Tiered Storage](#10-data-retention--tiered-storage)
11. [AI Layer — Natural Language Over Logs and Traces](#11-ai-layer--natural-language-over-logs-and-traces)
12. [Migration Plan — Zero Downtime Cutover (Standard OTEL or BindPlane)](#12-migration-plan--zero-downtime-cutover)
13. [Cost Analysis](#13-cost-analysis)
14. [Risk Assessment](#14-risk-assessment)
15. [Success Metrics](#15-success-metrics)
16. [Reference Links](#16-reference-links)

---

## 1. Executive Summary

### Objective

Replace the current Elasticsearch-based observability stack (application logs + OTEL traces + metrics) with ClickHouse — reducing annual infrastructure cost from $400K to ~$35-60K while gaining faster aggregations, better compression, unified storage with business data, and AI-powered log querying.

### The Problem Today

- Paying $400K/year for Elasticsearch managed service
- Elasticsearch is optimized for full-text search — most log queries are aggregations (error counts, latency percentiles, service health) where ClickHouse is **2-6x faster on cold queries, 1.7-2.6x on hot queries** (ClickHouse/TextBench benchmark, OTEL logs at 1B–50B rows)
- Logs, traces, metrics, and business data live in separate systems — no cross-correlation
- Storage costs are high: same OTEL dataset takes **5x more space in Elasticsearch** (49 GB vs 245 GB at 1B rows; 2.4 TB vs 12 TB at 50B rows)
- Kibana is the only query interface — no programmatic access, no AI layer

### The Solution

```
Keep OpenTelemetry (OTEL) as the instrumentation standard — zero application changes.
Change only the destination: swap Elasticsearch exporter → ClickHouse exporter in OTEL Collector.

All logs, traces, and metrics land in ClickHouse.
Grafana reads ClickHouse for dashboards and alerts.
AI agent queries logs/traces in plain English via the same LibreChat + MCP platform.
```

### Expected Outcomes

| Metric | Target |
|--------|--------|
| Annual cost reduction | $340-365K saved (~85-90% reduction) |
| Storage reduction | **5x smaller** total footprint (16x on column files) — real benchmark at 1B–50B OTEL rows |
| Query speed improvement | **2-6x faster** cold queries, **1.7-2.6x** hot queries (ClickHouse/TextBench) |
| Retention period | Same or longer — at lower cost |
| Unification | Logs + traces + metrics + business data in one DB |
| AI queries over logs | Plain English → SQL → instant answer |

---

## 2. The Problem — Elasticsearch at $400K/Year

### Where the Cost Comes From

| Cost Driver | Elasticsearch Behaviour | Annual Cost Share |
|-------------|------------------------|-------------------|
| Storage | Row-oriented index, 2-3x compression, needs SSD | ~$100K |
| Compute | CPU-heavy indexing on every write, inverted index maintenance | ~$150K |
| Licensing | Elastic managed service / Elastic Cloud premium | ~$100K |
| Operations | Shard management, index lifecycle management (ILM), tuning | ~$50K (eng time) |

### The Technical Mismatch

Elasticsearch was built for **full-text search** on documents (web pages, articles). Application logs and OTEL telemetry are **structured time-series data** — they need aggregations, not document search.

| What teams actually query | Elasticsearch efficiency | ClickHouse efficiency |
|---------------------------|------------------------|----------------------|
| "Error count per service last hour" | Slow (aggregation on inverted index) | Fast (columnar scan) |
| "P99 latency for /api/trade endpoint" | Slow (percentile aggregation) | Fast (built-in quantile functions) |
| "Show logs for TraceId = abc123" | Fast (indexed term lookup) | Fast (bloom filter) |
| "Which services degraded after deploy at 14:00?" | Medium | Fast |
| "Free-text: find logs containing OutOfMemoryError" | Fast (native) | Good (tokenbf bloom filter) |

~90% of real observability queries are aggregations. Elasticsearch is paying full price for a capability (full-text search) that covers only ~10% of use cases.

**Where ClickHouse wins most — benchmark by query type (cold, 50B rows):**

| Query Category | Examples | ClickHouse speedup |
|---------------|----------|-------------------|
| Log retrieval (text match + fetch rows) | "Find logs containing OutOfMemoryError" | Narrowest gap — ES competitive |
| Error/match counts | "Count 500 errors in last hour" | Moderate advantage |
| Service-level breakdowns | "Group errors by service" | Large advantage |
| Time-series trend analysis | "Error rate per minute over last 24h" | **Widest gap — 6x+ faster** |

The speedup grows with analytical complexity. Retrieval-only queries (find and show matching rows) is ES's home turf. The moment you add grouping, aggregation, or time-bucketing on top of a text match — the dominant pattern in observability — ClickHouse's vectorized engine pulls away decisively.

### Current Stack Pain Points

```
Application
    │
    ▼ OTEL SDK (instrumented)
OTEL Collector
    │
    ▼ Elasticsearch Exporter
Elasticsearch Cluster
    │
    ▼
Kibana                   ← only UI, no programmatic access
```

- No way to join log data with business data (e.g., "which users were affected by this error?")
- Kibana dashboards require manual setup — no AI layer
- Retention limited by cost — older logs are deleted or archived to cold storage with no query access
- Every new service that emits logs increases Elasticsearch cost linearly

---

## 3. Why ClickHouse for Observability

### Companies Already Doing This

**Cloudflare** — Replaced Elasticsearch with ClickHouse for HTTP logs:
- 36 petabytes of data
- Queries that took 30+ seconds in Elasticsearch run in < 1 second in ClickHouse
- Cost reduced by 80%

**Uber** — Migrated logging to ClickHouse:
- 100+ billion log rows per day
- Storage reduced from petabytes (Elasticsearch) to terabytes (ClickHouse)

**Contentsquare** — OTEL traces and logs in ClickHouse:
- 50 billion events/day
- Replaced both Elasticsearch and a custom Cassandra setup

**Langfuse** (now part of ClickHouse) — LLM observability platform:
- Stores all LLM traces in ClickHouse
- Handles millions of traces per day
- This is the same Langfuse we use in the Agentic AI Platform

### ClickHouse vs Elasticsearch for Observability

| Capability | Elasticsearch | ClickHouse |
|-----------|--------------|------------|
| Storage model | Inverted index (document-oriented) | Columnar (OLAP) |
| Compression ratio | 2-3x | 10-30x |
| Aggregation speed | Seconds (post-indexing) | Milliseconds |
| Write throughput | Medium (index maintenance overhead) | Very high (append-only MergeTree) |
| Full-text search | Excellent (native inverted index) | Good (bloom filter indexes) |
| OTEL native support | Via Logstash/Beats (extra hop) | Native OTEL exporter (direct) |
| Cross-data joins | Not possible | Join logs with business tables |
| AI/LLM integration | Kibana AI (limited) | Full MCP + LLM stack |
| Tiered storage | ILM (complex config) | TTL + S3 (2 lines of SQL) |
| License | Elastic License (paid tiers) | Apache 2.0 (fully open source) |
| Operational complexity | High (shards, replicas, ILM) | Low (MergeTree auto-manages) |

### Key Architectural Advantage

ClickHouse already stores business data (trades, wallets, users, risk) via the Agentic AI Platform. Adding observability data to the **same cluster** means:

```sql
-- This query is IMPOSSIBLE in Elasticsearch:
-- "Which users were affected by the 500 errors on trading-service between 14:00-14:30?"

SELECT DISTINCT l.LogAttributes['user_id'] AS user_id,
       u.kyc_status,
       count() AS error_count
FROM otel_logs l
JOIN mart_users u ON l.LogAttributes['user_id'] = u.username
WHERE l.ServiceName = 'trading-service'
  AND l.SeverityText = 'ERROR'
  AND l.Timestamp BETWEEN '2026-04-17 14:00:00' AND '2026-04-17 14:30:00'
GROUP BY user_id, u.kyc_status
ORDER BY error_count DESC;
```

In ClickHouse: one query, instant answer.
In Elasticsearch + separate business DB: impossible without custom ETL.

### Horizontal Scaling — Parallel Replicas

For high-volume workloads, ClickHouse scales a single query across multiple nodes using **parallel replicas** — the query is split across the replica fleet and results are merged. At 50B rows (same TextBench dataset):

| Nodes | Total query runtime | Speedup |
|-------|-------------------|---------|
| 1 node | 19.1s | baseline |
| 3 nodes | 12.5s | 1.5x |
| 9 nodes | 6.45s | 3x |
| 20 nodes | 3.27s | **5.8x** |

For text-search queries specifically, **index sharding** distributes the inverted index analysis across the replica fleet — delivering a 5.8x speedup on full-text queries at scale. This is relevant for any high-volume environment as log volume grows: add nodes, queries get proportionally faster, no schema or application changes needed.

---

## 4. Platform Architecture

### Target Architecture

```
                    +------------------------------------------+
                    |         CRYPTO EXCHANGE APPS                |
                    |                                          |
                    | Trading Service  Wallet Service          |
                    | Risk Service     User Service            |
                    | API Gateway      Background Workers      |
                    +-------------------+----------------------+
                                        |
                           OTEL SDK (zero app code change)
                           Auto-instruments: HTTP, DB, Kafka, Redis
                                        |
                    +-------------------v----------------------+
                    |           OTEL COLLECTOR                 |
                    |                                          |
                    |  Receivers: OTLP gRPC/HTTP               |
                    |  Processors: batch, resource, filter     |
                    |  Exporters: clickhouseexporter           |
                    +-------------------+----------------------+
                                        |
                    +-------------------v----------------------+
                    |           CLICKHOUSE CLUSTER             |
                    |                                          |
                    |  database: otel                          |
                    |  ┌─────────────┬──────────┬──────────┐  |
                    |  │ otel_logs   │otel_traces│otel_metrics│ |
                    |  │ (Logs)      │(Traces)  │(Metrics) │  |
                    |  └─────────────┴──────────┴──────────┘  |
                    |                                          |
                    |  database: marts (business data)         |
                    |  ┌────────────────────────────────────┐  |
                    |  │ mart_trades  mart_users  mart_wallets│ |
                    |  └────────────────────────────────────┘  |
                    +---+------------------+-------------------+
                        |                  |
           +------------v---+     +--------v-----------+
           |    GRAFANA     |     |   LIBRECHAT + LLM  |
           |                |     |   (AI queries over |
           | Dashboards     |     |   logs in plain    |
           | Trace Viewer   |     |   English)         |
           | Log Explorer   |     |                    |
           | Alerts         |     |                    |
           +----------------+     +--------------------+
```

### What Changes vs Current Stack

| Component | Before | After |
|-----------|--------|-------|
| OTEL SDK (app) | Same | Same — zero changes |
| OTEL Collector | Same | Same — only exporter config changes |
| Storage | Elasticsearch | ClickHouse |
| Log/Trace UI | Kibana | Grafana |
| Alerting | Kibana Alerts | Grafana Alerting |
| AI queries | None | LibreChat + Qwen + MCP |
| Cross-data correlation | Not possible | Native SQL joins |

---

### OTEL Collector Deployment — Two Architectures

There are two ways to deploy the OTEL Collector. Choosing the wrong one causes data loss during ClickHouse restarts or network blips.

#### Agent-Only (not recommended for production)

```
App → OTEL Collector (on same host) → ClickHouse directly
```

Each service runs its own collector. Simple to set up but fragile — if ClickHouse is briefly unreachable, the agent has no buffer and **drops data**. Works for development or low-stakes services.

#### Aggregator (recommended for production)

```
App → OTEL Agent (lightweight, on each host)
          │
          ▼
    OTEL Aggregator (central, 1–2 instances)
    - batches writes
    - retries on ClickHouse failure
    - filters/transforms before storage
          │
          ▼
    ClickHouse
```

The agent on each host is lightweight — just forwards to the aggregator. The aggregator does the heavy work: batching, retry on failure, PII filtering, routing. If ClickHouse goes down for 5 minutes, the aggregator queues data and flushes when it comes back. **No data loss.**

```yaml
# OTEL Agent config (on each service host — minimal, just forwards)
exporters:
  otlp:
    endpoint: otel-aggregator.internal:4317   # sends to aggregator, not ClickHouse

# OTEL Aggregator config (central — does batching, retry, filtering)
processors:
  batch:
    timeout: 10s
    send_batch_size: 50000
  retry_on_failure:
    enabled: true
    initial_interval: 5s
    max_elapsed_time: 300s

exporters:
  clickhouse:
    endpoint: tcp://clickhouse.internal:9000
    database: otel
    compress: lz4
    timeout: 10s
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_elapsed_time: 300s
```

**For a crypto exchange:** Use the Aggregator pattern. Trading and wallet services cannot drop observability data — if an incident occurs during a ClickHouse maintenance window, you need the logs.

---

## 5. OpenTelemetry — The Standard We Keep

### Why OTEL is the Right Foundation

OpenTelemetry is a CNCF standard for instrumentation — vendor-neutral, supported by every major cloud provider and observability tool. We keep OTEL as our instrumentation layer. Only the **backend destination** changes.

```
Application Code → OTEL SDK → OTEL Collector → [ANY BACKEND]
                                                   ↑
                              Swap this: Elasticsearch → ClickHouse
```

### OTEL Data Types We Capture

| Signal | What It Is | Example |
|--------|-----------|---------|
| **Logs** | Structured log events with severity, body, attributes | "ERROR: failed to execute trade, user=john_doe, error=InsufficientBalance" |
| **Traces** | Distributed request flow across services with timing | Full journey of a trade order: API → trading-service → PostgreSQL → Kafka |
| **Metrics** | Numeric measurements over time | HTTP request rate, error rate, latency histogram, DB connection pool size |

### OTEL Collector — Only the Exporter Changes

```yaml
# otel-collector-config.yaml

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 10000
  memory_limiter:
    limit_mib: 512
  resource:
    attributes:
      - key: environment
        value: production
        action: upsert

exporters:
  # ─── BEFORE (Elasticsearch) ───────────────────────────────
  # elasticsearch:
  #   endpoints: [https://elastic-cluster:9200]
  #   logs_index: logs-%{+yyyy.MM.dd}
  #   traces_index: traces-%{+yyyy.MM.dd}

  # ─── AFTER (ClickHouse) ────────────────────────────────────
  clickhouse:
    endpoint: tcp://clickhouse:9000
    database: otel
    logs_table_name:    otel_logs
    traces_table_name:  otel_traces
    metrics_table_name: otel_metrics
    ttl:     90               # days — auto-delete old data
    compress: lz4
    timeout:  5s
    retry_on_failure:
      enabled:           true
      initial_interval:  5s
      max_interval:      30s
      max_elapsed_time:  300s

service:
  pipelines:
    logs:
      receivers:  [otlp]
      processors: [memory_limiter, batch, resource]
      exporters:  [clickhouse]
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, batch, resource]
      exporters:  [clickhouse]
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, batch, resource]
      exporters:  [clickhouse]
```

**That is the only config change needed.** Applications keep emitting OTEL. The collector keeps receiving. Only the destination changes.

### Application Instrumentation — Zero Code Changes

For most languages, attach the OTEL agent at startup:

#### Java (Spring Boot / any JVM)
```bash
# Add to JVM startup — auto-instruments HTTP, JDBC, Kafka, Redis
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=trading-service \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -Dotel.traces.exporter=otlp \
     -Dotel.metrics.exporter=otlp \
     -Dotel.logs.exporter=otlp \
     -jar trading-service.jar
```

#### Python (FastAPI / Django)
```python
# Add 3 lines at startup — auto-instruments HTTP, PostgreSQL, Redis
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.psycopg2 import Psycopg2Instrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

FastAPIInstrumentor.instrument_app(app)
Psycopg2Instrumentor().instrument()
RequestsInstrumentor().instrument()
```

#### Node.js
```javascript
// tracing.js — load before anything else via --require
require('@opentelemetry/auto-instrumentations-node/register');
// Auto-instruments: express, pg, http, redis, mongoose, kafka
```

---

## 5b. Collection Agent Options — OTEL vs Fluent Bit vs Vector

OTEL Collector is not the only way to ship data to ClickHouse. Three agents are production-proven. The compression difference between them is significant enough to affect your storage cost.

### Compression Comparison (real benchmarks — identical log dataset)

| Agent | Compression Ratio | Status | Protocol |
|-------|------------------|--------|----------|
| **Fluent Bit** | **33×** | Mature | HTTP JSONEachRow |
| **Vector** | **21×** | Beta / widely used | HTTP JSON |
| **OTEL Collector** | **14×** | Alpha for ClickHouse exporter | Native ClickHouse TCP |

Fluent Bit achieves 33× compression because it gives you full control over the schema — you define exactly which fields land in which typed columns. OTEL Collector uses a fixed schema (Map types for attributes) which is more flexible but less compressible.

**For a crypto exchange:** OTEL Collector is the right default because we already use OTEL instrumentation end-to-end and the fixed schema covers all signals. If storage cost becomes a concern at high volume, Fluent Bit is the migration path — it requires a custom schema but delivers the best compression.

---

### When to Use Each

| Agent | Use When |
|-------|---------|
| **OTEL Collector** | You already emit OTEL signals (our case). Single config change, zero app changes. Traces + logs + metrics in one pipeline. |
| **Fluent Bit** | K8s environment, log-heavy workload, storage cost is a priority. Best compression. Mature and battle-tested. Does not handle traces. |
| **Vector** | You want a fully custom schema with maximum flexibility. Good middle ground — better compression than OTEL, handles more data types than Fluent Bit. |

---

### Fluent Bit → ClickHouse Config (for reference)

```yaml
# fluent-bit.yaml — ships K8s pod logs directly to ClickHouse
[OUTPUT]
    Name          clickhouse
    Match         *
    Host          clickhouse.internal
    Port          8123
    Database      otel
    Table         fluent_logs
    # Critical: async inserts prevent too-small-batch errors
    async_insert  1
    flush         10s
```

```sql
-- Custom schema for Fluent Bit (33x compression)
CREATE TABLE otel.fluent_logs
(
    timestamp               DateTime64(9)             CODEC(Delta, ZSTD(1)),
    pod_name                LowCardinality(String),
    namespace               LowCardinality(String),
    container_name          LowCardinality(String),
    severity                LowCardinality(String),
    message                 String                    CODEC(ZSTD(1)),
    kubernetes_labels       Map(LowCardinality(String), String) CODEC(ZSTD(1))
)
ENGINE = MergeTree
PARTITION BY toDate(timestamp)
ORDER BY (namespace, pod_name, timestamp)
TTL toDateTime(timestamp) + INTERVAL 90 DAY;
```

> **Important:** Fluent Bit requires `async_insert=1` and flush intervals ≥10 seconds. Without this, each log line triggers a separate HTTP insert and ClickHouse performance degrades significantly from too many small writes.

---

## 6. ClickHouse Schema — Logs, Traces, Metrics

### Logs Table

```sql
CREATE TABLE otel.otel_logs (
    Timestamp           DateTime64(9)              CODEC(Delta, ZSTD(1)),
    TraceId             String                     CODEC(ZSTD(1)),
    SpanId              String                     CODEC(ZSTD(1)),
    TraceFlags          UInt32,
    SeverityText        LowCardinality(String),    -- ERROR, WARN, INFO, DEBUG
    SeverityNumber      Int32,
    ServiceName         LowCardinality(String),
    Body                String                     CODEC(ZSTD(1)),
    ResourceSchemaUrl   String                     CODEC(ZSTD(1)),
    ResourceAttributes  Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    ScopeSchemaUrl      String                     CODEC(ZSTD(1)),
    ScopeName           String                     CODEC(ZSTD(1)),
    ScopeVersion        String                     CODEC(ZSTD(1)),
    ScopeAttributes     Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    LogAttributes       Map(LowCardinality(String), String) CODEC(ZSTD(1)),

    -- Skip indexes for fast filtering (replaces Elasticsearch inverted index for common patterns)
    INDEX idx_trace_id   TraceId     TYPE bloom_filter(0.001) GRANULARITY 1,
    INDEX idx_body       Body        TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 1,
    INDEX idx_service    ServiceName TYPE set(100)             GRANULARITY 4,
    INDEX idx_severity   SeverityText TYPE set(10)             GRANULARITY 4
)
ENGINE = MergeTree
PARTITION BY toDate(Timestamp)
ORDER BY (ServiceName, SeverityText, toUnixTimestamp(Timestamp))
TTL toDateTime(Timestamp) + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;
```

### Traces Table

```sql
CREATE TABLE otel.otel_traces (
    Timestamp           DateTime64(9)              CODEC(Delta, ZSTD(1)),
    TraceId             String                     CODEC(ZSTD(1)),
    SpanId              String                     CODEC(ZSTD(1)),
    ParentSpanId        String                     CODEC(ZSTD(1)),
    TraceState          String                     CODEC(ZSTD(1)),
    SpanName            LowCardinality(String),
    SpanKind            LowCardinality(String),    -- SERVER, CLIENT, PRODUCER, CONSUMER
    ServiceName         LowCardinality(String),
    ResourceAttributes  Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    SpanAttributes      Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    Duration            Int64,                     -- nanoseconds
    StatusCode          LowCardinality(String),    -- STATUS_CODE_OK, STATUS_CODE_ERROR
    StatusMessage       String,
    Events              Nested (
        Timestamp       DateTime64(9),
        Name            LowCardinality(String),
        Attributes      Map(LowCardinality(String), String)
    ),
    Links               Nested (
        TraceId         String,
        SpanId          String,
        TraceState      String,
        Attributes      Map(LowCardinality(String), String)
    ),

    INDEX idx_trace_id   TraceId     TYPE bloom_filter(0.001) GRANULARITY 1,
    INDEX idx_span_name  SpanName    TYPE set(50)             GRANULARITY 4,
    INDEX idx_service    ServiceName TYPE set(50)             GRANULARITY 4
)
ENGINE = MergeTree
PARTITION BY toDate(Timestamp)
ORDER BY (ServiceName, SpanName, toUnixTimestamp(Timestamp))
TTL toDateTime(Timestamp) + INTERVAL 90 DAY;
```

### Metrics Table

```sql
CREATE TABLE otel.otel_metrics (
    ResourceAttributes  Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    ResourceSchemaUrl   String,
    ScopeName           String,
    ScopeVersion        String,
    ScopeAttributes     Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    ScopeDroppedAttrCount UInt32,
    ScopeSchemaUrl      String,
    MetricName          LowCardinality(String),
    MetricDescription   String                 CODEC(ZSTD(1)),
    MetricUnit          String,
    Attributes          Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    StartTimeUnix       DateTime64(9)          CODEC(Delta, ZSTD(1)),
    TimeUnix            DateTime64(9)          CODEC(Delta, ZSTD(1)),
    Value               Float64                CODEC(ZSTD(1)),
    Flags               UInt32,
    Exemplars           Nested (
        FilteredAttributes Map(LowCardinality(String), String),
        TimeUnix           DateTime64(9),
        Value              Float64,
        SpanId             String,
        TraceId            String
    ),
    AggTemp             Int32,
    IsMonotonic         UInt8
)
ENGINE = MergeTree
PARTITION BY toDate(TimeUnix)
ORDER BY (MetricName, ServiceName, toUnixTimestamp(TimeUnix))
TTL toDateTime(TimeUnix) + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;
```

### Why `LowCardinality` and `CODEC` Matter

| Optimization | What It Does | Impact |
|-------------|-------------|--------|
| `LowCardinality(String)` | Dictionary-encodes repeated values (ServiceName, SeverityText) | 3-5x compression + faster filtering |
| `CODEC(Delta, ZSTD(1))` on Timestamp | Delta encodes sequential timestamps, then ZSTD compresses | 5-10x compression on time columns |
| `CODEC(ZSTD(1))` on String cols | General purpose compression | 3-7x |
| `bloom_filter` index on TraceId | Skips data blocks that can't contain the TraceId | Near O(1) lookup |
| `tokenbf_v1` index on Body | Token-based bloom filter for keyword search | Skips irrelevant blocks without full scan |
| `PARTITION BY toDate(Timestamp)` | One partition per day — old days auto-deleted by TTL | Instant TTL, no maintenance |
| `ORDER BY (ServiceName, SeverityText, Timestamp)` | Most queries filter by service + severity + time | Queries read minimal data |

---

## 7. End-to-End Distributed Tracing — HTTP to ClickHouse

### The Full Trace Flow

Every incoming HTTP request gets a `TraceId`. That same ID propagates through every service and DB call. ClickHouse stores all spans. Grafana shows the waterfall.

```
Browser → [traceparent header injected]
              │
              ▼
[1] Nginx               → Span: http.server (method, url, status, duration)
              │           propagates traceparent to upstream
              ▼
[2] API Gateway         → Span: route handling
              │
              ▼
[3] Trading Service     → Span: business logic
              │
        ┌─────┴──────┐
        ▼             ▼
[4] PostgreSQL      [5] ClickHouse    → Span: db.query (SQL text, duration, rows)
(OTEL JDBC auto)    (native OTEL)
        │
        ▼
[6] Kafka Producer      → Span: messaging.publish (topic, partition)

All spans → OTEL Collector → ClickHouse otel_traces table
                                         │
                                         ▼
                                   Grafana Trace UI
                                   (full waterfall)
```

### Layer 1 — Nginx (Trace Entry Point)

```nginx
# nginx.conf
load_module modules/ngx_otel_module.so;

http {
    otel_exporter {
        endpoint otel-collector:4317;
    }
    otel_service_name  "api-gateway";
    otel_trace         on;
    otel_trace_context propagate;    # injects traceparent into upstream request

    server {
        location /api/ {
            otel_trace      on;
            otel_span_name  "$request_method $uri";
            otel_span_attr  http.method  $request_method;
            otel_span_attr  http.url     $uri;
            otel_span_attr  http.status  $status;
            proxy_pass      http://backend;
        }
    }
}
```

### Layer 2 — Application Services (Auto-Instrumented)

OTEL Java agent auto-instruments every JDBC query. For ClickHouse queries, pass the context explicitly:

#### Python — Tracing ClickHouse Queries
```python
from opentelemetry import trace
from opentelemetry.propagate import inject
import clickhouse_connect

tracer = trace.get_tracer("trading-service")

def get_user_risk_profile(username: str):
    with tracer.start_as_current_span("clickhouse.mart_user_risk_profile") as span:
        span.set_attribute("db.system",    "clickhouse")
        span.set_attribute("db.name",      "marts")
        span.set_attribute("db.operation", "SELECT")
        span.set_attribute("db.statement",
            "SELECT * FROM mart_user_risk_profile WHERE username = ?")

        # Inject current trace context into ClickHouse HTTP headers
        headers = {}
        inject(headers)

        client = clickhouse_connect.get_client(
            host="clickhouse",
            http_headers=headers    # ClickHouse reads traceparent from here
        )
        result = client.query(
            "SELECT * FROM mart_user_risk_profile WHERE username = {username:String}",
            parameters={"username": username}
        )
        span.set_attribute("db.rows_returned", len(result.result_rows))
        return result.result_rows
```

#### Go — Tracing ClickHouse Queries
```go
import (
    "github.com/ClickHouse/clickhouse-go/v2"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
)

tracer := otel.Tracer("trading-service")

func getTopTraders(ctx context.Context) ([]Trader, error) {
    ctx, span := tracer.Start(ctx, "clickhouse.mart_trades_futures")
    defer span.End()

    span.SetAttributes(
        attribute.String("db.system",    "clickhouse"),
        attribute.String("db.operation", "SELECT"),
        attribute.String("db.statement", "SELECT username, SUM(quantity*price) FROM mart_trades_futures ..."),
    )

    // clickhouse-go/v2 propagates ctx trace headers automatically
    conn, _ := clickhouse.Open(&clickhouse.Options{
        Addr: []string{"clickhouse:9000"},
    })

    rows, err := conn.QueryContext(ctx, `
        SELECT username, SUM(quantity * price) as volume
        FROM mart_trades_futures
        WHERE transaction_time >= today()
        GROUP BY username
        ORDER BY volume DESC
        LIMIT 10
    `)
    return parseRows(rows), err
}
```

### Layer 3 — ClickHouse Native Tracing

ClickHouse reads the `traceparent` header from every query and emits its own internal spans to `system.opentelemetry_span_log`:

```sql
-- ClickHouse internal spans for a specific trace
SELECT
    trace_id,
    span_id,
    parent_span_id,
    operation_name,
    (finish_time_us - start_time_us) / 1000  AS duration_ms,
    attribute['clickhouse.query']             AS sql_query
FROM system.opentelemetry_span_log
WHERE trace_id = '4bf92f3577b34da6a3ce929d0e0e4736'
ORDER BY start_time_us;
```

Export these to `otel_traces` via a materialized view so they appear in Grafana alongside app spans:

```sql
-- Auto-export ClickHouse internal spans to otel_traces
CREATE MATERIALIZED VIEW otel.clickhouse_spans_mv TO otel.otel_traces AS
SELECT
    fromUnixTimestamp64Micro(start_time_us)  AS Timestamp,
    lower(hex(trace_id))                     AS TraceId,
    lower(hex(span_id))                      AS SpanId,
    lower(hex(parent_span_id))               AS ParentSpanId,
    ''                                       AS TraceState,
    operation_name                           AS SpanName,
    'SPAN_KIND_SERVER'                       AS SpanKind,
    'clickhouse'                             AS ServiceName,
    map()                                    AS ResourceAttributes,
    attribute                                AS SpanAttributes,
    (finish_time_us - start_time_us) * 1000  AS Duration,   -- convert to nanoseconds
    'STATUS_CODE_OK'                         AS StatusCode,
    ''                                       AS StatusMessage,
    []                                       AS `Events.Timestamp`,
    []                                       AS `Events.Name`,
    []                                       AS `Events.Attributes`,
    []                                       AS `Links.TraceId`,
    []                                       AS `Links.SpanId`,
    []                                       AS `Links.TraceState`,
    []                                       AS `Links.Attributes`
FROM system.opentelemetry_span_log;
```

### What a Full Trace Looks Like in ClickHouse

```sql
-- Full waterfall for a single trade request
SELECT
    SpanName,
    ServiceName,
    Duration / 1e6                          AS duration_ms,
    SpanAttributes['http.method']           AS http_method,
    SpanAttributes['http.route']            AS route,
    SpanAttributes['db.statement']          AS sql,
    SpanAttributes['db.rows_affected']      AS rows,
    StatusCode,
    Timestamp
FROM otel.otel_traces
WHERE TraceId = '4bf92f3577b34da6a3ce929d0e0e4736'
ORDER BY Timestamp ASC;
```

**Result — full waterfall in one query:**

```
SpanName                          Service              duration_ms
──────────────────────────────    ──────────────────── ───────────
http.server POST /api/trade       api-gateway   245.3 ms
  └─ trade.execute                trading-service      241.1 ms
       ├─ db.query (risk check)   trading-service       18.4 ms  ← SELECT mart_user_risk_profile
       ├─ db.query (insert trade) trading-service        4.2 ms  ← INSERT INTO mart_trades_futures
       ├─ SELECT (internal CH)    clickhouse             3.1 ms  ← ClickHouse internal span
       └─ kafka.produce           trading-service        2.1 ms
```

---

## 8. Full-Text Log Search — Replacing Kibana Discover

### Updated Position — ClickHouse Now Competitive on Full-Text Search

Full-text search used to be the primary reason to keep Elasticsearch. ClickHouse has significantly closed this gap with a **new `text` index type** that works natively on object storage (S3) with the same performance as local disk — removing the last major technical advantage Elasticsearch held.

> Reference: [ClickHouse Full-Text Search on Object Storage](https://clickhouse.com/blog/clickhouse-full-text-search-object-storage)

### New `text` Index — How to Add It to otel_logs

```sql
-- Add the new text index to the Body column
ALTER TABLE otel.otel_logs
    ADD INDEX body_text_idx(Body)
    TYPE text(tokenizer = 'splitByNonAlpha', preprocessor = lower(Body))
    GRANULARITY 1;

-- Materialize the index on existing data
ALTER TABLE otel.otel_logs MATERIALIZE INDEX body_text_idx;
```

**What the `text` index supports:**

| Function | Example | Use Case |
|----------|---------|----------|
| `hasToken` | `hasToken(Body, 'OutOfMemoryError')` | Exact token match — fastest |
| `hasAllTokens` | `hasAllTokens(Body, ['trade', 'failed'])` | All tokens must appear |
| `hasAnyTokens` | `hasAnyTokens(Body, ['ERROR', 'FATAL'])` | Any token match |
| `LIKE` | `Body LIKE '%OutOfMemoryError%'` | Wildcard match |
| `startsWith` | `startsWith(Body, 'WARN')` | Prefix match |
| `match` (regex) | `match(Body, 'user_[0-9]+')` | Regex search |

**Performance:** 7.4x speedup vs full table scan on text search (ClickHouse benchmark, 10M rows with array tags).

**Why it works on S3:** The index uses sequential dictionary reads with front-coding compression — no random I/O, which is the key constraint on object storage. 94.5% of tokens appear in ≤6 rows, so embedded posting lists handle the vast majority of lookups without reading large posting lists.

> **Real-world proof point — gitTrends:** ClickHouse's own reference demo ([github.com/ClickHouse/gitTrends](https://github.com/ClickHouse/gitTrends)) searches **10 billion+ GitHub events** using `hasToken()` on a `body` text index — the exact same pattern used for log search here. The app lets users compare FTS index vs bloom-filter skip index vs full table scan in real time, with live row-scan counters streamed from ClickHouse. Sub-second queries at 10B rows validate the production viability of `hasToken()` for high-volume text search workloads.

### How ClickHouse Handles Log Search

#### Keyword Search — New `text` index (preferred)
```sql
-- Uses new text index — 7.4x faster than full scan
SELECT Timestamp, ServiceName, SeverityText, Body
FROM otel.otel_logs
WHERE hasToken(Body, 'OutOfMemoryError')
  AND Timestamp >= now() - INTERVAL 1 HOUR
ORDER BY Timestamp DESC
LIMIT 100;

-- LIKE also uses the text index automatically
SELECT Timestamp, ServiceName, SeverityText, Body
FROM otel.otel_logs
WHERE Body LIKE '%OutOfMemoryError%'
  AND Timestamp >= now() - INTERVAL 1 HOUR
ORDER BY Timestamp DESC
LIMIT 100;
```

#### Exact TraceId Lookup (bloom filter)
```sql
-- Near O(1) — bloom_filter index makes this very fast
SELECT *
FROM otel.otel_logs
WHERE TraceId = 'abc123def456'
ORDER BY Timestamp;
```

#### Structured Attribute Search
```sql
-- Filter on OTEL log attributes — common in structured logging
SELECT ServiceName, count() AS error_count
FROM otel.otel_logs
WHERE LogAttributes['http.status_code'] = '500'
  AND Timestamp >= today()
GROUP BY ServiceName
ORDER BY error_count DESC;
```

#### Multi-Condition Log Search (most common Kibana query)
```sql
-- "All ERROR logs from trading-service in last 2 hours that mention user_id"
SELECT
    Timestamp,
    SeverityText,
    Body,
    LogAttributes['user_id']    AS user_id,
    LogAttributes['error_code'] AS error_code,
    TraceId
FROM otel.otel_logs
WHERE ServiceName  = 'trading-service'
  AND SeverityText = 'ERROR'
  AND Timestamp    >= now() - INTERVAL 2 HOUR
  AND hasToken(Body, 'user_id')     -- uses text index
ORDER BY Timestamp DESC
LIMIT 200;
```

#### Case-Insensitive Search (preprocessor)
```sql
-- The text index preprocessor lowercases at index time — search is case-insensitive
SELECT Timestamp, ServiceName, Body
FROM otel.otel_logs
WHERE hasToken(lower(Body), 'outofmemoryerror')   -- matches OOM, oom, Oom, etc.
  AND Timestamp >= now() - INTERVAL 1 HOUR;
```

### Full-Text Search Comparison — Updated

| Query Type | Elasticsearch | ClickHouse | Notes |
|-----------|--------------|------------|-------|
| Exact token match (`hasToken`) | ~10ms | **~15-30ms** | Near-comparable with text index |
| Keyword search (`LIKE`) | ~10ms | **~20-50ms** | text index — 7.4x faster than scan |
| Structured attribute filter | ~50ms | ~5-20ms | CH wins (columnar) |
| Aggregation (error count by service) | ~200ms-2s | ~10-50ms | CH wins significantly |
| Time-range + service filter | ~100ms | ~10-30ms | CH wins (partition pruning) |
| TraceId lookup | ~10ms | ~20-50ms | Comparable (bloom filter) |
| Free-text fuzzy search | Excellent | Good (regex via `match()`) | ES still leads for fuzzy |
| Search on S3/object storage | Degraded (SSD required) | **Full speed on S3** | CH advantage — no SSD needed |

**Bottom line:** ~95% of observability queries are structured (service + time + severity + attribute) — ClickHouse wins on all of those. For the remaining ~5% requiring keyword search in log bodies, the new `text` index brings ClickHouse to near-Elasticsearch performance. The only remaining ES advantage is fuzzy/phrase search, which is rarely needed for structured application logs.

**Coming soon in ClickHouse:** phrase search (position-aware token matching) and JSON column indexing — which will close the remaining gap further.

---

## 9. Visualization — Grafana Replaces Kibana

### Setup

```bash
# Install ClickHouse datasource plugin
grafana-cli plugins install grafana-clickhouse-datasource

# Or in docker-compose:
environment:
  - GF_INSTALL_PLUGINS=grafana-clickhouse-datasource
```

### Datasource Configuration

```yaml
# grafana/provisioning/datasources/clickhouse.yaml
apiVersion: 1
datasources:
  - name: ClickHouse-OTEL
    type: grafana-clickhouse-datasource
    uid: clickhouse-otel
    jsonData:
      host: clickhouse
      port: 9000
      database: otel
      username: grafana_readonly
    secureJsonData:
      password: ${GRAFANA_CH_PASSWORD}
```

### Key Dashboards to Build

#### 1. Service Health Overview
```sql
-- Error rate per service — last 1 hour, 1-minute buckets
SELECT
    toStartOfMinute(Timestamp)   AS time,
    ServiceName,
    countIf(SeverityText = 'ERROR') AS errors,
    count()                          AS total,
    errors / total * 100             AS error_rate_pct
FROM otel.otel_logs
WHERE Timestamp >= now() - INTERVAL 1 HOUR
GROUP BY time, ServiceName
ORDER BY time ASC;
```

#### 2. Latency Distribution (P50 / P95 / P99)
```sql
-- HTTP endpoint latency percentiles — last 30 minutes
SELECT
    SpanAttributes['http.route']          AS endpoint,
    quantile(0.50)(Duration) / 1e6        AS p50_ms,
    quantile(0.95)(Duration) / 1e6        AS p95_ms,
    quantile(0.99)(Duration) / 1e6        AS p99_ms,
    count()                               AS request_count
FROM otel.otel_traces
WHERE SpanKind    = 'SPAN_KIND_SERVER'
  AND Timestamp  >= now() - INTERVAL 30 MINUTE
GROUP BY endpoint
ORDER BY p99_ms DESC;
```

#### 3. Slowest Database Queries
```sql
-- Slowest ClickHouse queries in last 1 hour
SELECT
    SpanAttributes['db.statement']     AS sql,
    count()                             AS calls,
    avg(Duration) / 1e6                 AS avg_ms,
    max(Duration) / 1e6                 AS max_ms,
    quantile(0.99)(Duration) / 1e6      AS p99_ms
FROM otel.otel_traces
WHERE SpanAttributes['db.system'] IN ('clickhouse', 'postgresql')
  AND Timestamp >= now() - INTERVAL 1 HOUR
GROUP BY sql
ORDER BY p99_ms DESC
LIMIT 20;
```

#### 4. Log Explorer (Kibana Discover equivalent)
```sql
-- Live log tail with filtering — wire to Grafana Logs panel
SELECT
    Timestamp,
    ServiceName,
    SeverityText,
    Body,
    TraceId,
    LogAttributes
FROM otel.otel_logs
WHERE Timestamp >= $__timeFrom()    -- Grafana time variable
  AND Timestamp <= $__timeTo()
  AND ServiceName = '$service'      -- Grafana template variable
  AND SeverityText IN ($severity)
ORDER BY Timestamp DESC
LIMIT 1000;
```

#### 5. Distributed Trace Waterfall

Configure Grafana Explore → Traces with ClickHouse datasource:

| Config Field | Value |
|-------------|-------|
| Table | otel.otel_traces |
| TraceID column | TraceId |
| SpanID column | SpanId |
| Parent SpanID | ParentSpanId |
| Start time | Timestamp |
| Duration | Duration (nanoseconds) |
| Service name | ServiceName |
| Operation name | SpanName |

Click any log line with a TraceId → jump directly to full trace waterfall.

### Correlating Logs + Traces + Business Data in Grafana

```sql
-- Grafana panel: "Affected users for errors in selected time range"
-- This is impossible in Kibana — requires joining log data with business data

SELECT
    l.LogAttributes['user_id']    AS user_id,
    u.kyc_status,
    count()                        AS error_count,
    max(l.Timestamp)               AS last_error
FROM otel.otel_logs l
JOIN marts.mart_users u ON l.LogAttributes['user_id'] = u.username
WHERE l.ServiceName  = '$service'
  AND l.SeverityText = 'ERROR'
  AND l.Timestamp    BETWEEN $__timeFrom() AND $__timeTo()
GROUP BY user_id, u.kyc_status
ORDER BY error_count DESC
LIMIT 50;
```

### Grafana Plugin — Upcoming Features

> These are confirmed items being prototyped by the ClickHouse Grafana plugin team — not yet shipped. Reference: [Our Vision for the ClickHouse Grafana Plugin](https://clickhouse.com/blog/grafana-plugin-vision)

**1. Deployment & K8s Annotation Presets**

Grafana annotations are vertical markers on time-series panels that flag notable events. The plugin will generate these automatically from OTel data — no manual query writing:

| Preset | Source | What it surfaces |
|--------|--------|-----------------|
| Deployment detection | `ResourceAttributes['service.version']` change | Service version change / rollback markers on dashboards |
| K8s lifecycle events | OTel resource attributes | Pod restarts, OOM kills, autoscaling events |

For a crypto exchange: deploy a new trading-service version → the timestamp appears as a marker on all latency/error dashboards automatically. Immediately answers "did this error spike start at deployment?"

**2. JWT Per-User Query Identity**

Currently, all Grafana users share a single ClickHouse datasource credential. The roadmap item forwards each Grafana user's JWT identity to ClickHouse, enabling:
- **Row-level access control** based on actual user identity — compliance team sees only compliance-relevant tables, risk team sees risk tables
- **Per-user audit trail** in ClickHouse query log — every dashboard query is attributed to a named person, not a shared service account
- **Per-user cost tracking** — token/compute cost per team member

This closes the gap between Kibana's per-user access model and the current Grafana/ClickHouse setup.

**3. Visual Metrics Builder (OTel Map Column Support)**

OTel metrics (CPU, memory, network I/O) currently require writing SQL aggregation queries by hand. The upcoming metrics builder provides:
- Select metric name from a dropdown (populated from the `otel_metrics` table)
- Choose aggregation (sum, avg, max, p99)
- Add group-by dimensions

The key improvement: OTel uses Map-type columns (`ResourceAttributes`, `LogAttributes`) containing key-value pairs. The builder will expose a **key picker** so users can filter on `ResourceAttributes['k8s.namespace.name']` or `ResourceAttributes['host.name']` without writing bracket notation. Makes infrastructure metrics explorable without SQL.

**4. Out-of-the-Box OTel + K8s Dashboards**

The plugin will ship importable JSON dashboards covering:
- Log volume by severity with per-service breakdown
- Trace duration distribution and service dependency map
- Per-service RED metrics (request rate, error rate, duration)
- Top spans visibility
- Kubernetes observability (namespaces, pods, nodes)

Goal: **from data ingestion to usable dashboards in minutes**, not hours of manual panel building.

**5. Compact Search-First Mode**

A new query mode resembling the ClickStack UI — a search bar with filter pills, no SQL required for common tasks:
- Click `+` / `-` on any field value in a log detail panel to instantly add include/exclude filters
- Select text within a log body to add a "line contains" full-text filter (backed by `hasToken()`)
- Facet autocomplete for column names, operators, and values
- SQL preview pane shows the generated query live — "Edit as SQL" button opens the editor pre-populated

Aimed at operators and on-call engineers who need to investigate quickly without knowing ClickHouse SQL.

---

## 10. Data Retention & Tiered Storage

### The Cost Driver — Retention × Volume

Elasticsearch cost is roughly: `daily_volume_GB × retention_days × cost_per_GB_per_day`

ClickHouse changes the equation at two levels:

1. **Compression:** 1TB of raw logs → ~200GB in ClickHouse (**5x smaller** end-to-end; 16x on column files — ClickHouse/TextBench)
2. **Tiered storage:** Hot recent data on SSD, cold older data on cheap S3

### TTL — Simple One-Line Retention

```sql
-- Delete logs older than 90 days — runs automatically in background
ALTER TABLE otel.otel_logs
    MODIFY TTL toDateTime(Timestamp) + INTERVAL 90 DAY;

-- Different retention per severity
ALTER TABLE otel.otel_logs
    MODIFY TTL
        toDateTime(Timestamp) + INTERVAL 7 DAY   -- default: 7 days
        OVERRIDE WHERE SeverityText = 'ERROR',    -- errors: 90 days
        toDateTime(Timestamp) + INTERVAL 90 DAY  WHERE SeverityText = 'ERROR',
        toDateTime(Timestamp) + INTERVAL 365 DAY WHERE SeverityText = 'CRITICAL';
```

### Tiered Storage — Hot/Cold/Archive

```sql
-- Storage policy: SSD (0-7 days) → S3 (7-90 days) → delete (90+ days)
-- Define in storage_configuration in config.xml:

-- hot:  /var/lib/clickhouse/data  (local SSD, fast)
-- cold: s3://clickhouse-cold-tier/ (S3, cheap)

ALTER TABLE otel.otel_logs
    MODIFY TTL
        toDateTime(Timestamp) + INTERVAL 7 DAY TO VOLUME 'cold',    -- move to S3
        toDateTime(Timestamp) + INTERVAL 90 DAY DELETE;             -- delete
```

### Storage Comparison

Real benchmark numbers from ClickHouse/TextBench (identical hardware: AWS m6i.8xlarge, OTEL log data):

| Dataset | Elasticsearch | ClickHouse | Ratio |
|---------|--------------|------------|-------|
| 1B rows | 245 GB | 49 GB | **5x smaller** |
| 10B rows | ~1.2 TB | ~245 GB | **5x smaller** |
| 50B rows | 12 TB | 2.4 TB | **5x smaller** |
| Column file compression | — | **16x** | — |

### Why Elasticsearch Uses 5x More Storage — Component Breakdown

The 5x gap is not just compression. It comes from four on-disk structures that Elasticsearch maintains but ClickHouse either handles more efficiently or doesn't need at all (at 50B rows):

| Storage Component | Elasticsearch | ClickHouse | Why the difference |
|-------------------|--------------|------------|-------------------|
| Columnar storage (doc_values / column files) | 5.02 TiB | 1.92 TiB | CH chains codecs per column (Delta+ZSTD, GCD) vs ES generic compression |
| Inverted index | 3.37 TiB | 515 GiB | CH inverted index is designed for analytics granules, not Lucene doc IDs |
| Stored fields (`_source`) | **3.00 TiB** | **0** | ES stores original JSON to reconstruct documents; CH reconstructs from columns directly |
| Points + norms (BKD trees, relevance scoring) | **617 GiB** | **0** | ES maintains numeric range indexes and per-doc relevance weights; CH uses sparse primary index (320 MiB) |
| **Total** | **12.01 TiB** | **2.43 TiB** | **5x smaller** |

**The two biggest drivers of Elasticsearch's overhead:**
- `_source` (3 TiB) — a near-complete second copy of all log data stored as compressed JSON so Elasticsearch can reconstruct the original document on retrieval. ClickHouse has no equivalent because it reconstructs rows directly from individual column files.
- Points + norms (617 GiB) — BKD-tree numeric range indexes and per-document relevance scoring metadata. Useful for web search ranking; irrelevant for observability queries. ClickHouse's entire sparse primary index for 50B rows is 320 MiB.

**Ingestion speed (50B rows):**

| System | Time |
|--------|------|
| ClickHouse | **Under 4 hours** |
| Elasticsearch | **~5 days** (after pipeline tuning) |

**Estimated storage at this scale:**

| Scenario | Elasticsearch | ClickHouse |
|----------|--------------|------------|
| 90-day retention | ~9TB on SSD | ~1.8TB on SSD (or much less on S3) |
| 1-year retention | ~36TB | ~7TB on S3 (cheap) |
| Storage cost/year | ~$100K+ | ~$2-5K |

---

## 11. AI Layer — Natural Language Over Logs and Traces

### The Unique Advantage

Since ClickHouse already hosts the Agentic AI Platform (LibreChat + Qwen + MCP), observability data in the same cluster is **automatically queryable via plain English**. No additional setup.

### Example AI Queries Over Logs

```
User: "Which services had the most errors in the last hour?"
→ SQL: SELECT ServiceName, count() FROM otel_logs WHERE SeverityText='ERROR'
        AND Timestamp >= now()-INTERVAL 1 HOUR GROUP BY ServiceName ORDER BY count() DESC

User: "Show me all traces where ClickHouse queries took more than 500ms today"
→ SQL: SELECT TraceId, ServiceName, Duration/1e6 AS ms, SpanAttributes['db.statement'] AS sql
        FROM otel_traces WHERE SpanAttributes['db.system']='clickhouse'
        AND Duration > 500000000 AND Timestamp >= today() ORDER BY Duration DESC

User: "Which users were affected by the trading-service errors between 2pm and 3pm today?"
→ SQL: SELECT l.LogAttributes['user_id'], u.kyc_status, count() as errors
        FROM otel_logs l JOIN mart_users u ON l.LogAttributes['user_id'] = u.username
        WHERE l.ServiceName='trading-service' AND l.SeverityText='ERROR'
        AND l.Timestamp BETWEEN today()+toIntervalHour(14) AND today()+toIntervalHour(15)
        GROUP BY 1,2 ORDER BY errors DESC
```

The last query is **impossible in Kibana** — it crosses observability data (logs) with business data (users). In ClickHouse, it's one SQL query the AI generates automatically.

### Business Glossary Additions for Observability

Add to `business_glossary.yaml`:

```yaml
# Observability terms
- "error logs" = otel_logs WHERE SeverityText = 'ERROR'
- "slow query" = otel_traces WHERE SpanAttributes['db.system'] IN ('clickhouse','postgresql') AND Duration > 500000000
- "HTTP 5xx" = otel_logs WHERE LogAttributes['http.status_code'] LIKE '5%'
- "trade service" = ServiceName = 'trading-service'
- "trace" = otel_traces WHERE TraceId = '<id>'
- "latency" = Duration / 1e6 (milliseconds)
- "p99" = quantile(0.99)(Duration) / 1e6
- "error rate" = countIf(SeverityText='ERROR') / count() * 100
```

---

## 12. Migration Plan — Zero Downtime Cutover

### Option A — Standard OTEL Collector (Recommended for Simple Setups)

If you run a small number of centralized OTEL Collectors (one per environment), the standard approach is sufficient — edit the collector config YAML to add the ClickHouse exporter alongside the existing Elasticsearch exporter for dual-write. No extra tooling needed.

### Option B — BindPlane (Recommended for Large Collector Fleets)

If you run OTEL Collectors on many individual servers/services, **BindPlane** is worth considering. It is a centralized management platform for OTEL Collector fleets — instead of editing YAML configs on each server manually, you manage all collector configurations from one dashboard.

```
Without BindPlane:
  Edit otel-collector.yaml on server 1
  Edit otel-collector.yaml on server 2
  Edit otel-collector.yaml on server N
  Restart each collector
  ...

With BindPlane:
  Add ClickHouse destination once in BindPlane UI
  Roll out to entire fleet in one click
```

**What BindPlane adds for this migration:**

| Capability | Value |
|-----------|-------|
| Central config management | One change pushes to all collectors instantly |
| Dual-write in one click | Route to Elasticsearch AND ClickHouse simultaneously without touching individual collectors |
| Service-by-service cutover | Route `trading-service` logs to ClickHouse first, validate, then add more services gradually |
| Severity-based routing | Route ERROR logs to Elasticsearch (keep during validation), INFO/DEBUG to ClickHouse only |
| Safe fleet rollout | Progressive rollout with automatic rollback on failure |
| 130+ sources and destinations | Supports standard OTEL ClickHouse exporter as destination |

**BindPlane for self-managed ClickHouse:**

BindPlane's native "ClickStack destination" connects to ClickHouse Cloud managed product. For self-managed ClickHouse, use the standard OTEL `clickhouseexporter` as a generic OTLP destination in BindPlane — same outcome, slightly more manual config. Example BindPlane destination config:

```yaml
# BindPlane destination config for self-managed ClickHouse
destination:
  type: otlp
  name: clickhouse-self-managed
  config:
    endpoint: tcp://clickhouse.internal:9000
    headers:
      - key: x-clickhouse-database
        value: otel
    tls:
      insecure: false
```

**Reference:** https://clickhouse.com/blog/bindplane-faster-otel-migrations-to-clickstack

---

### Phase 1 — Deploy & Validate (Week 1-2)

| Task | Details |
|------|---------|
| Deploy ClickHouse schema | Create otel_logs, otel_traces, otel_metrics tables |
| Add ClickHouse exporter to OTEL Collector | Dual-write: send to both Elasticsearch AND ClickHouse (via YAML edit or BindPlane) |
| Set up Grafana | Install ClickHouse datasource, build core dashboards |
| Validate data parity | Compare row counts, spot-check log content |
| Test trace waterfall | Pick 5-10 real TraceIds, verify waterfall in Grafana matches Kibana |

### Phase 2 — Parallel Run (Week 3-4)

| Task | Details |
|------|---------|
| Run both stacks simultaneously | Elasticsearch + ClickHouse receive same data |
| Migrate dashboards | Rebuild all Kibana dashboards in Grafana |
| Migrate alerts | Recreate all Kibana alerts in Grafana Alerting |
| Train teams | Grafana walkthrough for each team |
| Build AI log queries | Add observability terms to business glossary |
| Gradual service cutover (optional) | Use BindPlane to route one service at a time to ClickHouse-only, validate, expand |

### Phase 3 — Cutover (Week 5)

| Task | Details |
|------|---------|
| Confirm all dashboards working in Grafana | Sign-off from each team |
| Remove Elasticsearch exporter from OTEL Collector | Single line config change (or one click in BindPlane) |
| Verify ClickHouse-only flow | 24-hour monitoring window |
| Cancel Elasticsearch subscription | After 48-hour clean run |

### Phase 4 — Optimise (Week 6+)

| Task | Details |
|------|---------|
| Tune TTL and tiered storage | Configure S3 cold tier based on actual usage patterns |
| Enable AI log queries in LibreChat | Add observability glossary, test with teams |
| Set up cross-data dashboards | Logs + business data correlation panels in Grafana |
| Performance tuning | Review slow queries via system.query_log, tune ORDER BY keys if needed |

### Risk Mitigation During Migration

- **Dual-write period:** Both systems receive data simultaneously for 4 weeks — no data loss risk
- **Rollback:** Removing ClickHouse exporter (or reverting BindPlane config) restores Elasticsearch-only in seconds
- **No application changes:** OTEL SDK configuration is unchanged throughout
- **Gradual cutover:** Cut over one service at a time using BindPlane routing rules if preferred

---

## 13. Cost Analysis

### Current Elasticsearch Cost Breakdown

| Component | Annual Cost |
|-----------|-------------|
| Elasticsearch managed service (compute) | ~$200K |
| Storage (SSD, replicated) | ~$100K |
| Licensing (Elastic managed / premium) | ~$50K |
| Engineering time (ops, tuning, ILM) | ~$50K |
| **Total** | **~$400K** |

### ClickHouse Cost (Self-Hosted)

| Component | Annual Cost |
|-----------|-------------|
| ClickHouse nodes (2x m7g.4xlarge, Graviton3) | ~$18K |
| Storage SSD (hot, last 7 days) | ~$3K |
| S3 (cold, 7-90 days) | ~$2K |
| ClickHouse Keeper (3x t3.small for consensus) | ~$2K |
| Engineering time (minimal ops) | ~$10K |
| **Total** | **~$35K** |

### ClickHouse Cost (ClickHouse Cloud — Recommended Start)

| Component | Annual Cost |
|-----------|-------------|
| ClickHouse Cloud (auto-scaling) | ~$36-60K |
| S3 tiered storage | Included |
| Engineering time | ~$5K (fully managed) |
| **Total** | **~$41-65K** |

### Savings Summary

| Scenario | Annual Cost | Saving vs Elasticsearch |
|----------|-------------|------------------------|
| Current (Elasticsearch) | $400K | — |
| ClickHouse Cloud | $41-65K | **$335-359K saved (84-90%)** |
| ClickHouse Self-Hosted | $35K | **$365K saved (91%)** |

### Additional Value (Not Counted in Savings)

- AI queries over logs — eliminates ad-hoc log digging by engineering (~20 hrs/month)
- Cross-data correlation — compliance/risk can correlate errors with affected users instantly
- Longer retention — at ClickHouse costs, retain 1 year vs 90 days for same budget
- Unified cluster — observability + business data in one system, one operations team

---

## 14. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Full-text search gaps | Low | Low | 95% of queries are structured; bloom filters cover keyword search |
| Data loss during migration | Very Low | High | 4-week dual-write window eliminates risk |
| Grafana learning curve | Medium | Low | Grafana is widely used; team familiarity is high |
| ClickHouse cluster instability | Low | High | 2-replica HA, Keeper for consensus, daily S3 backups |
| OTEL Collector overload | Low | Medium | Batch processor + memory limiter configured; scale collector horizontally |
| Schema changes in new service | Low | Low | MergeTree handles new columns gracefully; OTEL schema is stable |
| Cold data access latency (S3) | Medium | Low | S3 queries are slower but acceptable for historical lookups |

---

## 15. Success Metrics

### 30-Day Targets (Post Cutover)

| Metric | Target |
|--------|--------|
| All dashboards migrated to Grafana | 100% |
| Elasticsearch subscription cancelled | Done |
| Log query latency (aggregation) | < 500ms for last 24h queries |
| Trace lookup latency | < 2 seconds for TraceId lookup |
| Data compression vs Elasticsearch | > 5x smaller footprint |
| Alert parity | All Kibana alerts recreated in Grafana |

### 90-Day Targets

| Metric | Target |
|--------|--------|
| Annual cost reduction | > $300K vs Elasticsearch baseline |
| AI log queries active | Teams using LibreChat for log analysis |
| Cross-data queries | At least 5 dashboards joining logs + business data |
| Retention extended | From 90 days to 180+ days (same cost) |
| Engineering time saved | 20+ hrs/month (no more ad-hoc log queries) |

---

## 16. Reference Links

### ClickHouse Observability

| Resource | URL |
|----------|-----|
| ClickHouse Observability docs | https://clickhouse.com/docs/use-cases/observability |
| Observability solution guide | https://clickhouse.com/docs/use-cases/observability/overview |
| ClickHouse as Elasticsearch alternative | https://clickhouse.com/blog/elasticsearch-to-clickhouse-for-logs |
| Building an Observability solution with ClickHouse | https://clickhouse.com/blog/storing-log-data-in-clickhouse-fluent-bit-vector-open-telemetry |
| ClickHouse for logs blog | https://clickhouse.com/blog/using-clickhouse-for-log-analytics |

### OpenTelemetry + ClickHouse

| Resource | URL |
|----------|-----|
| OTEL Collector ClickHouse exporter | https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/clickhouseexporter |
| OTEL Collector contrib repo | https://github.com/open-telemetry/opentelemetry-collector-contrib |
| OTEL ClickHouse schema reference | https://clickhouse.com/docs/use-cases/observability/schema-design |

### Grafana + ClickHouse

| Resource | URL |
|----------|-----|
| Grafana ClickHouse datasource | https://grafana.com/grafana/plugins/grafana-clickhouse-datasource |
| Grafana ClickHouse plugin docs | https://grafana.com/docs/grafana/latest/datasources/clickhouse |

### Benchmarks & Case Studies

| Resource | URL |
|----------|-----|
| Cloudflare: ClickHouse for HTTP logs | https://blog.cloudflare.com/log-analytics-using-clickhouse |
| ClickHouse vs Elasticsearch log analytics benchmark | https://clickhouse.com/blog/elasticsearch-log-analytics-clickhouse |
| Benchmark source code (reproducible) | https://github.com/ClickHouse/TextBench |
| BindPlane — OTEL fleet management for migrations | https://clickhouse.com/blog/bindplane-faster-otel-migrations-to-clickstack |
| Langfuse + ClickHouse (LLM observability) | https://clickhouse.com/blog/langfuse-llm-analytics |

### Appendix A — Technology Stack Summary

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Instrumentation | OTEL SDK (unchanged) | Auto-instrument apps — zero code changes |
| Collection | OTEL Collector + clickhouseexporter | Receives and ships logs/traces/metrics to ClickHouse |
| Storage | ClickHouse otel database | Logs, traces, metrics — compressed columnar storage |
| Visualization | Grafana + ClickHouse datasource | Dashboards, trace waterfall, log explorer, alerting |
| AI queries | LibreChat + Qwen + MCP (existing) | Plain English queries over logs and traces |
| Cold storage | S3 (tiered via ClickHouse TTL) | Cheap long-term retention for historical data |
| HA | ClickHouse 2-replica cluster | Same HA setup as the existing ClickHouse cluster |

### Appendix B — Quick Reference: Kibana → Grafana Equivalents

| Kibana Feature | Grafana Equivalent |
|---------------|--------------------|
| Discover (log search) | Explore → Logs panel with ClickHouse query |
| Dashboard | Dashboard (same concept) |
| Visualize | Panel with ClickHouse SQL query |
| APM (traces) | Explore → Traces panel with ClickHouse datasource |
| Alerts | Grafana Alerting (same or better) |
| Index Lifecycle Management | ClickHouse TTL (simpler — one SQL line) |
| KQL (Kibana Query Language) | SQL (standard, more powerful) |
| Lens (drag-drop charts) | Grafana panel builder |

---

## Closing thoughts

If you're considering this migration, the decisions that matter most:

1. **Don't skip the OTEL Aggregator pattern.** Agent-only loses data on ClickHouse blips. Run a couple of central aggregators with retry-on-failure — that's the production-grade choice.
2. **Use BindPlane if you have a large collector fleet.** Worth it for fleet-wide config rollout. For a handful of central collectors, standard YAML is fine.
3. **Get the schema right the first time.** `ORDER BY (ServiceName, SeverityText, Timestamp)` and the right CODECs are the difference between a 3× and 30× compression ratio. The schema in Section 6 has been validated at production scale.
4. **Run dual-write for 4 weeks**, not 1. The gradual cutover is cheap insurance and lets you validate every dashboard/alert before cutting Elasticsearch off.
5. **The AI layer pays for itself.** Plain-English log queries via LibreChat + an LLM means no more pinging engineering when the compliance team needs a one-off analysis. Once ClickHouse has the data, the AI integration is one config change.

The full schema, OTEL Collector configs, Grafana queries, migration plan, and 16 production-tested recovery runbooks live in the [companion repo on GitHub](https://github.com/rakeshtherani/clickhouse-ai-dba). The repo also includes the AI DBA MCP server (152 tools for ClickHouse operations) — if you're operating at scale, that's worth a look.

If you've migrated off Elasticsearch (or are mid-migration), I'd love to compare notes. Reach out via [LinkedIn](https://www.linkedin.com/) or comment below.

If this is useful to your team, the deeper architectural piece — *Building an Agentic AI Data Platform on ClickHouse* — is coming next.
