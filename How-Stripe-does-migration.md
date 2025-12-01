# Stripe's Document Database & Zero-Downtime Data Migrations

A deep dive into how Stripe maintains 99.999% uptime while migrating petabytes of data.

Based on: [Stripe's Engineering Blog Post](https://stripe.com/blog/how-stripes-document-databases-supported-99.999-uptime-with-zero-downtime-data-migrations)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [The Data Movement Platform](#the-data-movement-platform)
  - [Step 1: Registration](#step-1-chunk-migration-registration)
  - [Step 2: Bulk Import](#step-2-bulk-data-import)
  - [Step 3: Async Replication](#step-3-async-replication)
  - [Step 4: Correctness Check](#step-4-correctness-check)
  - [Step 5: Traffic Switch (Versioned Gating)](#step-5-traffic-switch-versioned-gating)
  - [Step 6: Cleanup](#step-6-chunk-migration-deregistration)
- [Key Concepts Explained](#key-concepts-explained)
  - [CDC Pipeline](#cdc-change-data-capture-pipeline)
  - [Kafka](#what-is-kafka)
  - [Versioned Gating Deep Dive](#versioned-gating-deep-dive)
- [Timeline Example](#complete-timeline-example)
- [Key Takeaways](#key-takeaways)

---

## Overview

Stripe processes **$1 trillion in payments yearly** and maintains **99.999% uptime**. They can't afford downtime for database maintenance, scaling, or migrations.

**The Challenge:** How do you move data between database servers without any interruption?

**The Solution:** DocDB — a custom Database-as-a-Service built on MongoDB Community.

### Stats

| Metric | Value |
|--------|-------|
| Queries per second | 5M+ |
| Database shards | 2,000+ |
| Collections | 5,000+ |
| Query shapes | 10,000+ |
| Data volume | Petabytes |

---

## Architecture

### High-Level Overview

```
┌─────────────────┐
│  Product Apps   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────────┐
│  Proxy Servers  │◄───►│ Chunk Metadata Svc  │
│     (Go)        │     │ (routing info)      │
└────────┬────────┘     └─────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│              Database Shards                │
├─────────┬─────────┬─────────┬───────────────┤
│ Shard 1 │ Shard 2 │ Shard 3 │    ....       │
└─────────┴─────────┴─────────┴───────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│           CDC Pipeline                      │
│    (oplog → Kafka → S3)                     │
└─────────────────────────────────────────────┘
```

### How Routing Works

1. App sends query to proxy server
2. Proxy asks chunk metadata service: "Where does this data live?"
3. Metadata service responds: "Shard 47"
4. Proxy routes query to Shard 47
5. Response flows back to app

### Sharding Model

```
┌─────────────────────────────────────────────────────┐
│                 Logical Database                    │
│              (what product teams see)               │
└─────────────────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │ Chunk A │      │ Chunk B │      │ Chunk C │
   └────┬────┘      └────┬────┘      └────┬────┘
        │                │                │
        ▼                ▼                ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │ Shard 1 │      │ Shard 2 │      │ Shard 3 │
   │(primary │      │(primary │      │(primary │
   │+ replicas)     │+ replicas)     │+ replicas)
   └─────────┘      └─────────┘      └─────────┘
```

---

## The Data Movement Platform

"Data Movement" covers any scenario where you're moving chunks of data between shards:

- **Splitting** — one shard is too big/hot, split across multiple shards
- **Merging** — consolidate underutilized shards
- **Upgrading** — move data from MongoDB v4 to v6 shard
- **Rebalancing** — even out data distribution

### The 6-Step Process

```
┌─────────────────────────────────────────────────────┐
│                 DATA MOVEMENT                       │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. Register intent                                 │
│           │                                         │
│           ▼                                         │
│  2. Bulk Import ──────► Copy historical snapshot    │
│           │              (time T, takes hours)      │
│           ▼                                         │
│  3. Async Replication ─► Replay writes after T      │
│           │              (via CDC: oplog→Kafka→S3)  │
│           ▼                                         │
│  4. Correctness ───────► Compare checksums          │
│           │                                         │
│           ▼                                         │
│  5. Traffic Switch ────► Version bump + route update│
│           │              (< 2 seconds of rejects)   │
│           ▼                                         │
│  6. Cleanup ───────────► Drop data from source      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Step 1: Chunk Migration Registration

Register the intent to migrate in the chunk metadata service, then build indexes on target shards.

```
Coordinator: "I'm going to move Chunk X from Shard 1 → Shard 5"
                    │
                    ▼
         ┌─────────────────────┐
         │ Chunk Metadata Svc  │
         │                     │
         │ chunk_x:            │
         │   source: shard_1   │
         │   target: shard_5   │
         │   status: MIGRATING │
         └─────────────────────┘
                    │
                    ▼
         Build indexes on Shard 5
```

---

### Step 2: Bulk Data Import

Take a snapshot at time T and copy all documents to target.

```
Time T: Snapshot taken
         │
         ▼
    ┌─────────────┐
    │ Read from   │  ← From secondary replica or S3
    │ snapshot    │    (not primary — avoid impacting queries)
    └─────────────┘
         │
         ▼
    ┌─────────────┐
    │ Sort data   │  ← Sort by index attributes
    │ in memory   │    (10x throughput improvement!)
    └─────────────┘
         │
         ▼
    ┌─────────────┐
    │ Batch write │  ← Write in sorted batches
    │ to target   │
    └─────────────┘
         │
         ▼
    Hours later: Bulk import complete
```

#### Why Sorting Helps

MongoDB uses B-tree data structure. Inserting in sorted order means:
- Sequential disk writes (not random)
- Better cache utilization
- 10x throughput improvement

#### Why Not Immediate?

1. **Volume** — 10M docs or terabytes can't be written instantly
2. **Sorting** — requires reading all data first
3. **Batching** — one doc at a time is slow
4. **Backpressure** — target shard has write limits

---

### Step 3: Async Replication

Replicate all writes that happened AFTER time T from source to target.

#### Why Both Steps Are Needed

```
Imagine migrating a chunk with 10 million documents:

Timeline:
─────────────────────────────────────────────────────►
     │                              │
   Time T                        Time T+6hrs
   (snapshot taken)              (bulk import done)


Bulk Import: Copies 10M docs that existed AT time T
             (historical data — frozen point in time)

Async Replication: Catches up the 50-200K writes
                   that happened DURING those 6 hours
```

**Analogy:** Moving apartments
- **Bulk import** = movers transport all your furniture (takes a day)
- **Async replication** = forwarding mail and groceries that arrived during the move

#### How It Works

```
During bulk import (6 hours):

Source Shard                         Kafka Queue
     │                                   │
     │──── UPDATE Doc B ────────────────►│
     │──── UPDATE Doc C ────────────────►│  Messages just
     │──── INSERT Doc D ────────────────►│  pile up here
     │                                   │

After bulk import completes:

Kafka Queue                         Target Shard
     │                                   │
     │──── UPDATE Doc B ────────────────►│ ✓ (exists)
     │──── UPDATE Doc C ────────────────►│ ✓ (exists)
     │──── INSERT Doc D ────────────────►│ ✓ (new doc)
```

#### Important: Sequential, Not Parallel

> "**Once we have imported the data onto the target shard**, we begin replicating writes starting at time T"

This avoids race conditions — what if you try to UPDATE a doc that hasn't been bulk imported yet?

```
✗ WRONG (parallel):
  Bulk import running...
  Replication: UPDATE Doc B  ← Doc B doesn't exist yet! FAIL

✓ CORRECT (sequential):
  Bulk import complete (Doc B exists)
  Replication: UPDATE Doc B  ← Works!
```

#### How Fast Is Async Replication?

Very fast — minutes, not hours.

```
Bulk import duration: ~6 hours
Writes during those 6 hours: ~50,000 - 200,000 ops

Kafka consumer throughput: ~5,000 ops/second

Time to replay: 200,000 ÷ 5,000 = 40 seconds
```

The accumulated historical data is massive. The rate of new writes is comparatively tiny.

---

### Step 4: Correctness Check

Compare point-in-time snapshots to verify data integrity.

```
Source Shard                    Target Shard
     │                               │
     ▼                               ▼
┌─────────────┐                ┌─────────────┐
│ Snapshot at │                │ Snapshot at │
│ time T2     │                │ time T2     │
└─────────────┘                └─────────────┘
     │                               │
     ▼                               ▼
  Compute                        Compute
  checksums                      checksums
     │                               │
     └───────────► Compare ◄─────────┘
                     │
              Are they equal?
                 │       │
                YES      NO
                 │       │
                 ▼       ▼
            Continue   Investigate
```

**What goes into checksums:**
- Document count
- Sum of document hashes (or merkle tree)
- Index metadata

**Why point-in-time?** Avoids impacting ongoing queries.

---

### Step 5: Traffic Switch (Versioned Gating)

The most critical step — atomically switch traffic from source to target.

```
The Protocol:

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. Bump version token on source shard (42 → 43)        │
│     └── Source now REJECTS all version 42 requests      │
│                                                         │
│  2. Wait for replication to catch up                    │
│     └── Any writes before bump get replicated           │
│                                                         │
│  3. Update chunk metadata service                       │
│     └── "Chunk X now lives on target, version 43"       │
│                                                         │
│  4. Proxies refresh routing                             │
│     └── Get new routes + version 43                     │
│     └── Future requests go to target                    │
│                                                         │
└─────────────────────────────────────────────────────────┘

Total time: < 2 seconds
```

#### Visual Timeline

```
                    Source bumps to v43
                           │
───────────────────────────│─────────────────────────────►
     │            │        │         │
  Request A    Request B   │      Request B
  (v42→source) (v42→source)│      RETRY
  SUCCESS      REJECTED    │      (v43→target)
                           │      SUCCESS
                           │
                    < 2 seconds
```

**What happens to failed requests?**
- Client retries automatically
- Proxy fetches fresh routes (now pointing to target)
- Retry succeeds on target

#### Why Bidirectional Replication?

During migration, they replicate BOTH ways:

```
Source ◄──────────────► Target
```

This allows rollback if something goes wrong:
1. Bump version on target
2. Update routes back to source
3. Writes that hit target get replicated back to source

It's a safety net.

---

### Step 6: Chunk Migration Deregistration

Cleanup:
1. Mark migration as complete in metadata service
2. Drop chunk data from source shard
3. Free up resources

---

## Key Concepts Explained

### CDC (Change Data Capture) Pipeline

Every write to MongoDB gets logged in the **oplog** (operations log):

```
oplog entries:
├── INSERT: {_id: 123, user: "alice", amount: 500}
├── UPDATE: {_id: 456, $set: {amount: 750}}
├── DELETE: {_id: 789}
└── ...
```

The CDC pipeline transports this:

```
MongoDB oplog → Kafka (streaming) → S3 (archive)
```

**Why read from Kafka instead of source DB?**

1. **No impact on source** — user queries don't compete with replication
2. **Decoupled** — if target is down, events safely stored in Kafka
3. **Replayable** — can restart from any checkpoint
4. **Unlimited history** — MongoDB oplog is capped, S3 is not

---

### What is Kafka?

Kafka is a **distributed event streaming platform** — like a queue on steroids.

#### Traditional Queue vs Kafka

```
Traditional Queue (RabbitMQ):
                              
Producer → [msg1, msg2, msg3] → Consumer
                              
           Message DELETED after read
           One consumer per message
```

```
Kafka:

Producer → [msg1, msg2, msg3, msg4, msg5] → Consumer A (offset 3)
                                          → Consumer B (offset 1)
                                          → Consumer C (offset 5)

           Messages RETAINED (days/weeks/forever)
           Multiple consumers at different positions
```

#### Key Differences

| Traditional Queue | Kafka |
|-------------------|-------|
| Message deleted after read | Messages retained |
| One consumer per message | Multiple consumers |
| No replay | Replay from any point |
| Single server | Distributed |

#### The Log Concept

```
Offset:    0     1     2     3     4     5
         ┌─────┬─────┬─────┬─────┬─────┬─────┐
         │ msg │ msg │ msg │ msg │ msg │ msg │
         └─────┴─────┴─────┴─────┴─────┴─────┘
                       ▲                 ▲
              Consumer A            Consumer B
              (offset 2)            (offset 5)
```

Each consumer tracks its own **offset**. This enables:
- Pause replication → offset stays put
- Resume later → continue from same offset
- Replay everything → reset offset to 0

#### Multiple Consumers, One Stream

```
MongoDB oplog → Kafka → Replication service (for migrations)
                  │
                  ├──→ Analytics pipeline (for dashboards)
                  │
                  ├──→ Search indexer (update Elasticsearch)
                  │
                  └──→ Archiver (store in S3)
```

One write, many consumers — each independent.

---

### Versioned Gating Deep Dive

The problem: How do you atomically switch traffic without losing writes?

#### The Setup

Every proxy includes a **version token** with requests:

```
Proxy → Shard: "Query + version token 42"
Shard: "I accept tokens ≥ 42, OK here's your data"
```

#### Normal Operation

```
┌─────────┐                    ┌─────────┐
│  Proxy  │ ── version 42 ──► │  Shard  │
│         │ ◄── response ──── │ (≥42 OK)│
└─────────┘                    └─────────┘
```

#### During Traffic Switch

```
Step 1: Bump source to version 43

┌─────────┐                    ┌─────────┐
│  Proxy  │ ── version 42 ──► │ Source  │
│         │ ◄── REJECTED ──── │ (≥43 OK)│
└─────────┘                    └─────────┘
                                    │
                          "Your version is stale!"

Step 2: Proxy refreshes routes

┌─────────┐     ┌──────────┐
│  Proxy  │ ──► │ Metadata │
│         │ ◄── │ Service  │
└─────────┘     └──────────┘
     │
     │ Gets: target=shard_5, version=43
     ▼

Step 3: Retry goes to target

┌─────────┐                    ┌─────────┐
│  Proxy  │ ── version 43 ──► │ Target  │
│         │ ◄── SUCCESS ───── │ (≥43 OK)│
└─────────┘                    └─────────┘
```

The entire switch takes **< 2 seconds**.

---

## Complete Timeline Example

```
Monday 2:00 AM   Snapshot taken (time T)
                 Note Kafka offset = 1000
                      │
Monday 2:00 AM   Start reading snapshot + sorting
                      │
Monday 2:30 AM   Start writing batches to target
                      │
                 ... bulk import running ...
                 ... Kafka offset grows to 1500 ...
                      │
Monday 8:00 AM   Bulk import COMPLETE (6 hours)
                 Target has all docs from time T
                      │
Monday 8:00 AM   Async replication STARTS
                 "Give me Kafka messages 1000-1500"
                      │
Monday 8:05 AM   Replication caught up
                      │
Monday 8:06 AM   Correctness check passes
                      │
Monday 8:07 AM   TRAFFIC SWITCH
                 - Bump version on source
                 - Wait for last writes to replicate
                 - Update routes in metadata
                 (takes < 2 seconds)
                      │
Monday 8:07 AM   All traffic now hitting target
                      │
Monday 8:10 AM   Cleanup: drop data from source
                      │
                 MIGRATION COMPLETE
```

---

## Key Takeaways

### Architectural Patterns

1. **Proxy Layer Abstraction**
   - Apps talk to proxies, not databases
   - Enables routing, admission control, auth
   - Makes migrations transparent to applications

2. **Chunk-Based Sharding**
   - Data split into logical chunks
   - Chunks can move independently
   - Enables fine-grained rebalancing

3. **CDC for Replication**
   - Use change log (oplog → Kafka → S3)
   - Don't read from source DB
   - Zero impact on production traffic

4. **Versioned Gating**
   - Version tokens for atomic traffic switch
   - Source rejects stale versions
   - Failed requests retry on new route

5. **Bidirectional Replication**
   - Replicate both ways during migration
   - Enables safe rollback

### The Numbers

| Metric | Value |
|--------|-------|
| Bulk import time | Hours |
| Async replication time | Minutes |
| Traffic switch time | < 2 seconds |
| Data migrated (2023) | 1.5 petabytes |
| Shard reduction (2023) | ~75% |

### When to Use These Patterns

- Zero-downtime migrations
- Horizontal scaling (split/merge shards)
- Database version upgrades
- Multi-tenant to single-tenant transitions
- Traffic rebalancing

---

## Further Reading

- [Original Stripe Blog Post](https://stripe.com/blog/how-stripes-document-databases-supported-99.999-uptime-with-zero-downtime-data-migrations)
- [MongoDB Sharding Explained](https://www.mongodb.com/resources/products/capabilities/database-sharding-explained)
- [MongoDB Oplog Documentation](https://www.mongodb.com/docs/manual/core/replica-set-oplog/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)

---

*Created while studying distributed systems and database infrastructure.*