# Media Asset Ingestion at Scale: A Complete Engineering Reference

> **Engineering Whitepaper — v1.0**
> **Classification:** Internal Reference
> **Audience:** Engineering leaders, staff/principal engineers, technical product managers, platform architects
> **Applicability:** Any consumer-scale media platform (image sharing, video streaming, digital art, e-commerce product imagery, social media, news media, educational content, digital signage)

---

## Preface: How to Use This Document

This whitepaper is a **technology-agnostic, pattern-driven reference** for designing and operating media ingestion pipelines at consumer internet scale (1 M–1 B+ assets, 10 K–100 K+ uploads/hour).

It is organized in three parts:

- **Part I (Chapters 1–3):** Foundational concepts — why ingestion is hard, the universal pipeline anatomy, and the design principles that separate resilient pipelines from fragile ones.
- **Part II (Chapters 4–12):** Stage-by-stage deep dives — each stage of a complete pipeline with decision matrices, failure modes, and concrete engineering guidance.
- **Part III (Chapters 13–17):** Cross-cutting concerns — state machines, observability, cost modeling, team topology, and a decision checklist for product managers.

**For product managers:** Start with the [Executive Summary](#executive-summary), then skip to [Chapter 15: The PM Decision Checklist](#15-the-pm-decision-checklist) and [Chapter 14: Cost Modeling](#14-cost-modeling-at-scale). Refer to individual stages as needed during feature planning.

**For engineers:** Read front-to-back on first pass. Use individual chapters as reference during implementation.

---

## Executive Summary

Every consumer media platform — whether it displays photographs, streams video, hosts podcasts, or renders digital art — must solve the same fundamental problem: **accepting raw media from producers, transforming it into a format optimized for every consumer device, enriching it with metadata for discovery, and delivering it globally with sub-second latency.**

This is deceptively complex. A single photo upload can trigger 30+ downstream operations across storage, compute, AI/ML, databases, caches, CDNs, and search indexes. A platform processing 50,000 uploads per hour must coordinate millions of these operations daily, handle partial failures gracefully, and do so at a per-asset cost measured in fractions of a cent.

This whitepaper distills the ingestion pipeline into **nine discrete, event-driven stages**:

```
 UPLOAD        VERIFY       METADATA      MEDIA         AI
 INITIATION  → INTEGRITY  → VALIDATION  → PROCESSING  → ENRICHMENT
                                                            │
 INDEXING &   ← CDN         ← FINAL       ← RIGHTS/     ← LOCALIZATION
 DISCOVERY      PROPAGATION    STORAGE       COMPLIANCE
```

Each stage is independently scalable, retryable, and observable. The document provides:

- **Decision matrices** for technology choices at each stage
- **Failure mode catalogs** with retry strategies and graceful degradation paths
- **Cost models** with per-asset breakdowns at scale
- **Best practices** validated across production systems handling billions of assets
- **Anti-patterns** with real-world consequences

The core insight: **an ingestion pipeline is not a monolithic batch job — it is a distributed, event-driven manufacturing line where every station must be idempotent, observable, and independently deployable.**

---

## Table of Contents

### Part I: Foundations
1. [Why Media Ingestion Is a Distinct Engineering Discipline](#1-why-media-ingestion-is-a-distinct-engineering-discipline)
2. [Universal Pipeline Anatomy](#2-universal-pipeline-anatomy)
3. [Foundational Design Principles](#3-foundational-design-principles)

### Part II: Stage-by-Stage Reference
4. [Stage 1 — Upload Initiation & Admission Control](#4-stage-1--upload-initiation--admission-control)
5. [Stage 2 — Binary Transfer & Integrity Verification](#5-stage-2--binary-transfer--integrity-verification)
6. [Stage 3 — Metadata Ingestion & Validation](#6-stage-3--metadata-ingestion--validation)
7. [Stage 4 — Media Processing & Variant Generation](#7-stage-4--media-processing--variant-generation)
8. [Stage 5 — AI/ML Enrichment](#8-stage-5--aiml-enrichment)
9. [Stage 6 — Localization](#9-stage-6--localization)
10. [Stage 7 — Rights, Compliance & Access Control](#10-stage-7--rights-compliance--access-control)
11. [Stage 8 — Final Storage Layout & CDN Propagation](#11-stage-8--final-storage-layout--cdn-propagation)
12. [Stage 9 — Indexing & Discoverability](#12-stage-9--indexing--discoverability)

### Part III: Cross-Cutting Concerns
13. [The Asset State Machine](#13-the-asset-state-machine)
14. [Cost Modeling at Scale](#14-cost-modeling-at-scale)
15. [The PM Decision Checklist](#15-the-pm-decision-checklist)
16. [Observability & Operational Readiness](#16-observability--operational-readiness)
17. [Team Topology & Parallel Development](#17-team-topology--parallel-development)

### Appendices
- [A: Platform Archetype Mapping](#appendix-a-platform-archetype-mapping)
- [B: Glossary](#appendix-b-glossary)

---

# Part I: Foundations

## 1. Why Media Ingestion Is a Distinct Engineering Discipline

Media ingestion is not a CRUD operation. It is a **manufacturing pipeline** where raw material (producer uploads) passes through multiple transformation, enrichment, and quality-control stations before becoming a finished product (a discoverable, deliverable, monetizable asset).

### 1.1 What Makes It Hard

| Challenge | Why It's Different From Typical Backend Work |
|---|---|
| **Heterogeneous input** | Producers upload in dozens of formats, resolutions, color spaces, and codecs. A single pipeline must normalize all of them. A 50 KB JPEG and a 50 GB ProRes file follow the same logical path but require radically different resource profiles. |
| **Combinatorial output** | One input asset may produce 20–40 output variants (thumbnails, previews, display resolutions, streaming segments, watermarked copies). Each variant is a separate object to store, cache, and deliver. |
| **Asymmetric compute** | Transcoding a 4K video requires GPU-hours. Generating a thumbnail requires milliseconds of CPU. AI tagging requires an LLM API call with unpredictable latency. These workloads cannot share the same scaling strategy. |
| **Cost dominance** | At scale, ingestion processing cost can exceed all other backend costs combined. A pipeline that costs $0.20 per video at 100 K videos/day is $20 K/day — $7.3 M/year. Every cent matters. |
| **Partial failure tolerance** | Unlike a database transaction, a 10,000-asset bulk upload cannot be all-or-nothing. The pipeline must support partial success: 9,847 assets ready, 120 with warnings, 33 failed with actionable errors. |
| **Time-to-visible pressure** | Producers expect uploads to be visible quickly. But "quickly" varies: a social media post needs seconds; a museum bulk upload can tolerate hours. The pipeline must support both modes. |

### 1.2 Platform Archetypes

Different platforms emphasize different stages. Understanding your archetype determines where to invest engineering effort:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     PLATFORM ARCHETYPE SPECTRUM                        │
├─────────────┬────────────┬─────────────┬──────────────┬────────────────┤
│  Social     │  E-Commerce│  Streaming  │  Digital Art │  Enterprise   │
│  Media      │  Product   │  Video      │  / Gallery   │  DAM          │
│  (Instagram,│  Imagery   │  (YouTube,  │  (Pinterest, │  (Brand asset │
│   TikTok)   │  (Shopify) │   Vimeo)    │   museums)   │   management) │
├─────────────┼────────────┼─────────────┼──────────────┼────────────────┤
│ SPEED       │ ACCURACY   │ TRANSCODE   │ FIDELITY     │ COMPLIANCE    │
│ dominates   │ dominates  │ dominates   │ dominates    │ dominates     │
│             │            │             │              │               │
│ Sub-second  │ Background │ GPU-heavy,  │ Color-exact, │ Approval      │
│ publish,    │ removal,   │ ABR ladder, │ 8K, ICC      │ workflows,    │
│ lightweight │ white      │ long        │ profiles,    │ audit trails, │
│ processing  │ balancing, │ processing  │ provenance   │ rights mgmt   │
│             │ SKU match  │ times OK    │ tracking     │               │
└─────────────┴────────────┴─────────────┴──────────────┴────────────────┘
```

### 1.3 The Cost of Getting It Wrong

| Failure | Business Impact |
|---|---|
| Slow time-to-visible | Producer churn. A creator who uploads a photo and doesn't see it in their gallery within a reasonable time will switch platforms. |
| Poor variant quality | Consumer experience degrades. Blurry thumbnails, color-shifted displays, or buffering video directly impact engagement metrics. |
| No cost controls | Runaway cloud bills. AI enrichment at scale without token budgets, or storing redundant variants without lifecycle policies, can 10× expected cost. |
| Brittle error handling | Silent data loss. An asset stuck in `PROCESSING` forever — never visible, never flagged — is the worst outcome. |
| Missing observability | Blind troubleshooting. When a producer reports "my upload didn't work," the team needs per-asset trace-level visibility across every stage. |

---

## 2. Universal Pipeline Anatomy

Every media ingestion pipeline, regardless of platform, follows the same nine-stage structure. Some platforms skip or simplify stages (a social media app may skip localization and rights management), but the logical flow is universal.

### 2.1 The Nine Stages

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        MEDIA INGESTION PIPELINE                              │
│                                                                              │
│   ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌────────┐ │
│   │   1.     │    │   2.     │    │   3.     │    │   4.     │    │  5.    │ │
│   │ UPLOAD   │───▶│ VERIFY   │───▶│ METADATA │──┬▶│ MEDIA    │    │  AI    │ │
│   │ INITIATE │    │ INTEGRITY│    │ VALIDATE │  │ │ PROCESS  │    │ENRICH  │ │
│   └─────────┘    └──────────┘    └──────┬───┘  │ └──────────┘    └───┬────┘ │
│                                         │      │                     │      │
│                                         └──────┼─────────────────────┘      │
│                                                │       (depends on 3)       │
│   ┌─────────┐    ┌──────────┐    ┌──────────┐  │ ┌──────────┐              │
│   │   9.    │    │   8.     │    │   7.     │  │ │   6.     │              │
│   │ INDEX & │◀───│  CDN     │◀───│ RIGHTS & │◀─┘ │LOCALIZE  │◀─────────────┘│
│   │DISCOVER │    │PROPAGATE │    │COMPLIANCE│    └──────────┘              │
│   └─────────┘    └──────────┘    └──────────┘                              │
│                                                                              │
│   ──────▶  Synchronous dependency                                           │
│   ──┬───▶  Parallel execution (stages 3+4 run concurrently)                 │
│   ──────▶  Event-driven (message broker between every stage)                │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Stage Responsibility Matrix

| # | Stage | Input | Output | Dominant Resource | Can Skip? |
|---|---|---|---|---|---|
| 1 | Upload Initiation | API request | Pre-signed URLs + job record | Network + DB | No |
| 2 | Integrity Verification | Raw binary in object storage | Verified binary + technical metadata | CPU + I/O | No |
| 3 | Metadata Validation | Producer-supplied metadata | Normalized, validated metadata record | CPU (light) | No |
| 4 | Media Processing | Verified binary | Multiple renderable variants | CPU/GPU (heavy) | No |
| 5 | AI/ML Enrichment | Preview variant + metadata | Tags, descriptions, embeddings | GPU / API call | Yes (degraded) |
| 6 | Localization | Enriched metadata | Translated metadata + multilingual embeddings | API call | Yes (single-locale) |
| 7 | Rights & Compliance | Asset + license config | Watermarks, DRM policies, access rules | CPU | Platform-dependent |
| 8 | CDN Propagation | Final variants | Edge-cached, globally accessible files | Network + CDN | No |
| 9 | Indexing & Discovery | Complete asset record | Searchable, browsable, serveable asset | DB + Search index | No |

### 2.3 Data Flow Principle: References, Not Payloads

A critical architectural principle: **binary media data (pixels, frames, audio samples) never flows through the message broker or the database.** Only references (object storage keys, URLs, asset IDs) flow through the event backbone.

```
  ┌─────────────┐                    ┌─────────────────────┐
  │             │  REFERENCE ONLY    │                     │
  │   Message   │◄─────────────────  │   Worker Process    │
  │   Broker    │   { asset_id,      │                     │
  │             │     r2_key,        │   Reads/writes      │
  │             │     status }       │   binary data       │
  │             │                    │   directly to/from  │
  └─────────────┘                    │   object storage    │
                                     │                     │
                                     └────────┬────────────┘
                                              │
                                     ┌────────▼────────────┐
                                     │   Object Storage    │
                                     │   (binary data)     │
                                     └─────────────────────┘
```

**Why this matters:**
- Message brokers are optimized for small messages (KB). Routing multi-GB video files through them causes backpressure, memory exhaustion, and cascading failures.
- Object storage provides built-in durability (11 nines), multipart upload, and range reads. Duplicating these capabilities in a message broker is wasteful.
- Workers can stream data from object storage — processing a 50 GB video does not require 50 GB of worker memory.

---

## 3. Foundational Design Principles

These ten principles apply to every stage. They are not aspirational — they are **non-negotiable requirements** for a pipeline that operates reliably at scale.

### Principle 1: Every Stage Must Be Idempotent

**Definition:** Processing the same asset through the same stage multiple times produces the same result without side effects (no duplicate records, no duplicate storage objects, no duplicate notifications).

**Implementation pattern:**
- Use the `asset_id` (a globally unique identifier, preferably UUIDv7 for time-sortability) as the idempotency key.
- Before performing work, check if the output already exists (e.g., variants already in object storage, metadata already in the database).
- Use upsert semantics (`INSERT ... ON CONFLICT DO UPDATE`) rather than blind inserts.

**Why it matters:** In a distributed system, messages will be delivered more than once. Network partitions, worker crashes, and broker redelivery guarantees make at-least-once delivery the norm. Without idempotency, duplicate processing creates orphaned files, double-counted metrics, and corrupted state.

### Principle 2: Events, Not Orchestrators

**Definition:** Each stage emits a domain event upon completion. Downstream stages subscribe to these events. There is no central orchestrator that "drives" the pipeline.

```
  ANTI-PATTERN: Central Orchestrator          PATTERN: Choreography
  ─────────────────────────────────           ──────────────────────

  ┌────────────┐                              Stage 1 ──event──▶ Stage 2
  │Orchestrator│──call──▶ Stage 1             Stage 2 ──event──▶ Stage 3
  │            │──call──▶ Stage 2             Stage 3 ──event──▶ Stage 4
  │            │──call──▶ Stage 3             (each stage is autonomous)
  │            │──call──▶ Stage 4
  │ (single    │
  │  point of  │
  │  failure)  │
  └────────────┘
```

**Why choreography over orchestration:**
- No single point of failure. If the integrity verification worker crashes, only that stage stalls — all other stages continue processing their queues.
- Independent scaling. The media processing stage (GPU-heavy) scales differently from the metadata validation stage (CPU-light). An orchestrator would need to understand all scaling profiles.
- Independent deployment. Teams can deploy changes to their stage without coordinating with an orchestrator release.

**When orchestration IS appropriate:** Long-running, human-in-the-loop workflows (e.g., manual content moderation review). Use a dedicated workflow engine (Temporal, AWS Step Functions) for these — but keep it separate from the high-throughput ingestion pipeline.

### Principle 3: Separation of Hot and Cold Paths

**Definition:** Distinguish between operations that must happen before an asset is visible to consumers (the hot path) and operations that can happen after (the cold path).

```
  HOT PATH (blocks visibility)              COLD PATH (runs after READY)
  ────────────────────────────              ────────────────────────────
  Upload → Verify → Process variants →     AI enrichment improvements
  Basic metadata → Minimal indexing →       Full localization (tier 2 locales)
  CDN propagation                           Analytics aggregation
                                            Re-enrichment with newer AI models
                                            Archival compression
                                            Popularity score recalculation
```

**Why it matters:** Time-to-visible is a critical producer experience metric. Every stage on the hot path directly adds to the time between "upload complete" and "asset visible in the platform." Moving optional enrichment to the cold path reduces this latency dramatically.

**Best practice:** Define your hot path explicitly. Document it. Measure it. Set an SLO on it. Every proposal to add a new stage to the hot path must justify the latency cost.

### Principle 4: Graceful Degradation Over Hard Failure

**Definition:** If a non-critical stage fails after exhausting retries, the pipeline promotes the asset to the next stage in a degraded state rather than blocking it permanently.

| Stage Failure | Degraded Behavior | Impact |
|---|---|---|
| AI enrichment fails | Asset is published without AI-generated tags | Search quality slightly reduced for this asset |
| Localization fails | Asset is published in source language only | Non-source-language users see untranslated metadata |
| 8K variant fails | Asset is published without 8K variant | 8K display users see upscaled 4K instead |
| Edge cache warming fails | Asset served from origin on first request | First consumer sees slightly higher latency |

**What should NOT degrade:** Integrity verification, basic metadata validation, at least one renderable variant, and the primary CDN path. These are hard requirements — failure here blocks publishing.

### Principle 5: Cost Attribution Per Asset

**Definition:** Every processing operation records its cost (compute time, storage bytes, API tokens consumed, egress) against the asset's `ingestion_id`. This enables per-asset cost visibility and per-producer cost allocation.

**Implementation:**
```
asset_cost_ledger (
  asset_id          UUID,
  stage             TEXT,          -- e.g., 'media_processing', 'ai_enrichment'
  resource_type     TEXT,          -- e.g., 'gpu_seconds', 'llm_tokens', 'storage_bytes'
  quantity          FLOAT,
  unit_cost_usd     FLOAT,
  total_cost_usd    FLOAT,
  recorded_at       TIMESTAMPTZ
)
```

**Why it matters:** Without per-asset cost attribution, cost optimization is guesswork. With it, teams can identify that "video transcoding is 60% of ingestion cost" and focus optimization efforts accordingly.

### Principle 6: Schema Versioning on Events

**Definition:** Every event emitted between stages carries a schema version. Consumers must handle multiple versions during migration windows.

```json
{
  "event_type": "AssetUploaded",
  "schema_version": 2,
  "asset_id": "019abc...",
  "mime_type": "image/tiff",
  "dimensions": { "width": 8000, "height": 6000 },
  "color_space": "adobe_rgb",
  "file_size_bytes": 145000000
}
```

**Why:** In a long-lived pipeline, the schema of events between stages will evolve. Adding a new field (e.g., `hdr_metadata`) should not require simultaneous deployment of all producers and consumers. Version the schema; write consumers that handle both v1 and v2; deprecate v1 after all producers have migrated.

### Principle 7: Correlation IDs Everywhere

**Definition:** Every operation on an asset — across every stage, every worker, every database write, every API call — carries the same correlation ID (`ingestion_id` or `asset_id`). This enables end-to-end distributed tracing.

**Why:** When a producer reports "my upload from 3 hours ago isn't visible," the support team must be able to trace the asset through every stage in seconds. Without correlation IDs, this requires log-diving across dozens of services.

### Principle 8: Separate Storage Tiers

**Definition:** Use distinct storage buckets/containers for each lifecycle phase.

| Bucket | Purpose | Access Pattern | Lifecycle |
|---|---|---|---|
| `originals` | Immutable producer uploads | Write-once, rare reads | Retain forever (archival tier after 90 days) |
| `processing` | Scratch space for intermediate artifacts | High read/write during processing | Delete after pipeline completion |
| `serving` | Final variants served via CDN | Read-heavy, CDN origin | Retain while asset is active |
| `quarantine` | Flagged/malicious content | Rare access, security team only | Review + delete or restore |

**Why:** Mixing lifecycle stages in a single bucket creates operational nightmares — you can't set different retention policies, access controls, or storage tiers (hot vs. cold) per file if they're all in the same bucket.

### Principle 9: Backpressure-Aware Scaling

**Definition:** Each worker pool monitors its input queue depth and scales horizontally based on that signal — but also respects downstream capacity limits.

```
  Queue depth → Worker scaling → BUT respects:
                                   - Max GPU instances (cost cap)
                                   - LLM API rate limits
                                   - Database connection pool limits
                                   - Object storage request rate limits
```

**Anti-pattern:** Auto-scaling workers without considering downstream limits. Scaling media processing workers to 100 instances when the database connection pool supports 50 causes cascading connection failures.

### Principle 10: Immutable Variants

**Definition:** Once a variant is written to the serving bucket, it is never modified in place. If a variant needs to be regenerated (new encoder, quality improvement), a new version is written to a new key, and the manifest is updated atomically.

**Why:**
- CDN caches can serve the variant with `Cache-Control: immutable` — maximum cache efficiency.
- No race conditions between a CDN edge serving a half-written file.
- Rollback is trivial — point the manifest back to the previous key.

---

# Part II: Stage-by-Stage Reference

## 4. Stage 1 — Upload Initiation & Admission Control

### 4.1 Purpose

This is the **front door** of the pipeline. It authenticates the producer, validates that the upload is permissible (quotas, rate limits, file type restrictions), registers the intent in the database, and returns the information the client needs to begin the binary transfer.

### 4.2 Why Pre-Signed URLs (Not Direct Upload to Your API)

The most consequential decision at this stage is: **does the binary data flow through your API servers, or directly to object storage?**

```
  ANTI-PATTERN: Proxy Upload                PATTERN: Direct-to-Storage
  ──────────────────────────                ────────────────────────────

  Client ──bytes──▶ API Server ──bytes──▶   Client ──bytes──▶ Object Storage
                    (bottleneck,                               (infinite scale,
                     memory pressure,                           built-in multipart,
                     timeout risk)                              resumable)
                                            API Server issues pre-signed URL only
                                            (lightweight, fast, stateless)
```

**Always use direct-to-storage uploads with pre-signed URLs.** This is not a preference — it is a hard requirement at scale. Proxying multi-gigabyte files through API servers wastes memory, bandwidth, and money. Object storage services are purpose-built for high-throughput binary ingestion with features (multipart upload, resumable transfers, server-side checksums) that you would have to reimplement poorly.

### 4.3 Admission Control Checklist

Every upload request must pass through these gates before a pre-signed URL is issued:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ADMISSION CONTROL GATES                           │
│                                                                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│  │   AUTH    │──▶│  QUOTA   │──▶│   RATE   │──▶│   FILE   │──▶ PASS │
│  │          │   │  CHECK   │   │  LIMIT   │   │ VALIDATE │         │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘         │
│       │              │              │              │                 │
│     401/403        429            429            400                │
│   Unauthorized   Over quota    Too fast     Invalid format/size     │
└──────────────────────────────────────────────────────────────────────┘
```

| Gate | Check | Rejection Response | Best Practice |
|---|---|---|---|
| **Authentication** | Valid token (JWT, API key, OAuth2) | 401 Unauthorized | Validate token signature locally (no DB call). For API keys: HMAC-sign the request body to prevent replay. |
| **Authorization** | Producer has `upload` or `bulk_upload` scope | 403 Forbidden | Scope-based access. Separate `upload` (single) from `bulk_upload` (batch) permissions. |
| **Quota** | Monthly upload count < plan limit | 429 + `Retry-After` + quota info | Store quota counters in cache (fast path) with periodic sync to DB (source of truth). Return remaining quota in response headers. |
| **Rate Limit** | Request rate < per-producer limit | 429 + `Retry-After` | Token bucket algorithm. Burst allowance: 2× steady-state rate for 30 seconds. |
| **File Type** | Declared MIME type in allowlist | 400 + supported types list | Allowlist, not blocklist. Explicitly enumerate supported types. Validate again at Stage 2 against actual file bytes. |
| **File Size** | Declared size ≤ maximum per type | 400 + max size per type | Different limits per media type (stills vs. video). For video: much higher limits (50–100 GB). |
| **Batch Size** | Asset count ≤ maximum per request | 400 + max batch size | Prevent a single request from creating excessive DB records. Typical limit: 100–1,000 per request. |

### 4.4 The Ingestion Job Record

Every upload — single or batch — creates a parent `IngestionJob` record. This is the unit of progress tracking for the producer.

```
ingestion_job (
  id              GLOBALLY_UNIQUE_ID,      -- UUIDv7 recommended (time-sortable)
  producer_id     FOREIGN_KEY,
  asset_count     INTEGER,
  status          ENUM (
                    'INITIATED',           -- URLs issued, awaiting upload
                    'UPLOADING',           -- At least one asset uploaded
                    'PROCESSING',          -- All assets uploaded, processing underway
                    'READY',               -- ≥ threshold % of assets ready
                    'PARTIAL_FAILURE',     -- Some assets failed, others ready
                    'FAILED'               -- All assets failed
                  ),
  success_count   INTEGER DEFAULT 0,
  failure_count   INTEGER DEFAULT 0,
  created_at      TIMESTAMP,
  updated_at      TIMESTAMP,
  expires_at      TIMESTAMP               -- Pre-signed URL expiry
)
```

**Best practice:** Define a success threshold for batch jobs. A 10,000-asset upload where 9,847 succeed and 33 fail should report `READY` (not `PARTIAL_FAILURE`) if the success rate exceeds 95%. The 33 failures are reported individually with actionable errors.

### 4.5 Pre-Signed URL Generation

| Parameter | Recommendation | Rationale |
|---|---|---|
| **Expiry** | 24 hours | Long enough for large batch uploads over slow connections. Short enough to limit exposure of signed URLs. |
| **Conditions** | Content-Type must match declared MIME, Content-Length ≤ declared size | Prevents producers from uploading a different file type or exceeding declared size. Object storage enforces these server-side. |
| **Multipart threshold** | 100 MB | Files above this should use multipart upload. Each part is independently retryable. Recommended part size: 100 MB. |
| **Key structure** | `originals/{producer_id}/{job_id}/{asset_id}/{filename}` | Producer-partitioned for access control. Job-grouped for batch cleanup. Asset-keyed for uniqueness. |

### 4.6 Response Contract

The response to a successful upload initiation should include everything the client needs to begin uploading without another round-trip:

```json
{
  "job_id": "019...",
  "status": "INITIATED",
  "expires_at": "2026-03-11T12:00:00Z",
  "assets": [
    {
      "asset_id": "019...",
      "upload_url": "https://storage.example.com/...",
      "upload_method": "PUT",
      "multipart": true,
      "part_size_bytes": 104857600,
      "required_headers": {
        "Content-Type": "video/mp4",
        "x-amz-checksum-sha256": "base64-encoded-hash"
      }
    }
  ],
  "progress_url": "/v1/ingestion-jobs/019.../status"
}
```

### 4.7 Emitted Event

```
UploadInitiated {
  job_id, producer_id, asset_count, expires_at
}
```

Consumed by: Upload timeout monitor (detects abandoned uploads).

---

## 5. Stage 2 — Binary Transfer & Integrity Verification

### 5.1 Purpose

Confirm that the binary file arrived intact, is what the producer claimed it to be, is free of malware, and extract technical metadata embedded in the file.

### 5.2 Upload Completion Detection

How does the pipeline know an upload is complete? Three patterns:

| Pattern | Mechanism | Latency | Recommendation |
|---|---|---|---|
| **Storage event notification** | Object storage emits an event (S3 Event Notification, R2 Event Notification, GCS Pub/Sub) when an object is created. | Sub-second | Preferred. Fastest, most reliable. |
| **Client callback** | Client calls an API endpoint after upload completes. | Depends on client | Required as a fallback (event notifications can be delayed). |
| **Polling** | Worker periodically checks for new objects. | Seconds to minutes | Last resort. Use only if storage events are unavailable. |

**Best practice:** Use storage event notifications as the primary trigger, with client callback as a confirmation. If the event arrives but no callback within 5 minutes, proceed anyway (the event is authoritative). If the callback arrives but no event, verify the object exists in storage before proceeding.

### 5.3 Integrity Verification Steps

```
┌────────────────────────────────────────────────────────────────────────────┐
│              INTEGRITY VERIFICATION PIPELINE                               │
│                                                                            │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐           │
│  │   SIZE    │──▶│ CHECKSUM  │──▶│  FORMAT   │──▶│  MALWARE  │──▶ PASS   │
│  │  CHECK    │   │  VERIFY   │   │  VERIFY   │   │   SCAN    │           │
│  └───────────┘   └───────────┘   └───────────┘   └───────────┘           │
│       │               │               │               │                   │
│     REJECT          REJECT          REJECT         QUARANTINE              │
│  (size mismatch) (corrupt file)  (type mismatch) (malware detected)       │
└────────────────────────────────────────────────────────────────────────────┘
```

| Step | What It Does | Implementation Notes |
|---|---|---|
| **Size check** | Compare actual uploaded bytes to declared `expected_bytes`. | Allow ±1% tolerance for encoding overhead (some clients report content size before transport encoding). If actual size is 0: reject immediately (empty file). |
| **Checksum verification** | Stream the object, compute SHA-256, compare to producer-declared checksum. | **Stream the object** — do not load the entire file into memory. For a 50 GB video, this means reading in chunks. If no checksum was declared: compute and store it (but log a warning — producers should always declare checksums). |
| **Format verification** | Inspect the actual file bytes (magic bytes / file signature) and compare to declared MIME type. | Use a magic-byte library (e.g., `libmagic`, `file` command, `mimetype` detection). Do NOT trust the declared MIME type alone — a malicious actor could declare `image/jpeg` but upload an executable. |
| **Malware scan** | Scan the binary with an antivirus engine. | ClamAV (open-source) or a cloud scanning service. For high-volume pipelines: run ClamAV as a daemon with shared socket for low-latency scanning. Update virus definitions daily. |

### 5.4 Technical Metadata Extraction

After integrity verification passes, extract the technical metadata embedded in the file. This metadata is used by downstream stages (media processing needs resolution and color space; AI enrichment benefits from camera/lens data).

**For still images** (extract via ExifTool or equivalent):

| Field | Source | Used By |
|---|---|---|
| Width × Height (pixels) | EXIF / image header | Media processing (variant sizing), search (aspect ratio filter) |
| DPI | EXIF | Print fulfillment (is this asset high enough resolution for large-format printing?) |
| Color space | ICC profile / EXIF | Media processing (color conversion), display rendering |
| Bit depth | Image header | Media processing (HDR detection for > 8-bit) |
| ICC profile | Embedded binary | Media processing (color-accurate rendering) |
| Camera model, lens | EXIF | Search (photographers search by gear) |
| GPS coordinates | EXIF | **STRIP by default for privacy.** Store only if producer explicitly opts in. |
| Orientation | EXIF orientation tag | Media processing (auto-rotate before variant generation) |
| Creation date | EXIF DateTimeOriginal | Display, sorting, search |

**For video** (extract via FFprobe or equivalent):

| Field | Source | Used By |
|---|---|---|
| Width × Height | Stream info | Media processing (ABR ladder), search |
| Frame rate | Stream info | Media processing (frame rate conversion if needed) |
| Duration | Format info | Media processing (thumbnail extraction), cost estimation |
| Codec (video) | Stream info | Media processing (decode strategy) |
| Codec (audio) | Stream info | Media processing (audio transcode) |
| Bitrate | Format info | Media processing (target quality), cost estimation |
| Audio track count | Stream info | Media processing (stereo vs. surround) |
| Color space / transfer | Stream info | Media processing (HDR handling) |
| HDR metadata | SEI messages / container | Media processing (tone mapping, HDR variant) |

### 5.5 Privacy-Sensitive Metadata

**This is a critical compliance concern.** Many image files contain GPS coordinates, device serial numbers, and other personally identifiable information in their EXIF data. Best practices:

1. **Extract and store GPS/PII metadata in a separate, access-controlled table** — not in the general metadata visible to consumers.
2. **Strip EXIF from all output variants** — thumbnails and display variants should never contain raw EXIF data.
3. **Allow producers to opt in to location sharing** — if they want their photo's location to be searchable, they can explicitly enable it.
4. **Audit trail:** Log when PII metadata is accessed and by whom.

### 5.6 Quarantine Protocol

When malware is detected or integrity verification fails in a way that suggests malicious intent:

1. Move the object to a quarantine bucket with restricted access (security team only).
2. Mark the asset record as `QUARANTINED` with the reason.
3. Emit an `AssetQuarantined` event.
4. Notify the producer (without revealing detection details that could aid evasion).
5. Notify the security team for review.
6. **Do not delete the quarantined file** — it may be needed for forensic analysis.

### 5.7 Emitted Event

```
AssetUploaded {
  asset_id, mime_type, width, height, file_size_bytes,
  color_space, duration_sec (video only), has_audio (video only),
  has_hdr (video only), technical_metadata_key
}
```

**Important:** This single event triggers **two downstream stages in parallel** — metadata validation (Stage 3) and media processing (Stage 4). Both stages are independent and do not need to wait for each other.

---

## 6. Stage 3 — Metadata Ingestion & Validation

### 6.1 Purpose

Accept, validate, normalize, and store the **descriptive** metadata provided by the producer (title, description, attribution, tags, license choice). This is distinct from the **technical** metadata extracted in Stage 2.

### 6.2 Metadata Delivery Mechanisms

Producers can provide metadata through multiple channels. The pipeline must support all of them:

```
┌─────────────────────────────────────────────────────────────────┐
│                METADATA DELIVERY MECHANISMS                      │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │   INLINE       │  │   SIDECAR      │  │   POST-UPLOAD  │    │
│  │   (in upload   │  │   MANIFEST     │  │   API CALL     │    │
│  │    request)    │  │   (.csv/.json)  │  │   (later)      │    │
│  └────────────────┘  └────────────────┘  └────────────────┘    │
│                                                                  │
│  Best for:           Best for:            Best for:             │
│  Single uploads,     Bulk uploads         Interactive           │
│  API integrations    (museums, DAMs)      uploads (artists      │
│                                           filling form in UI)   │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Validation: The Three Layers

Metadata validation is not a single check — it is three layers of increasing depth:

```
Layer 1: SYNTACTIC           Layer 2: SEMANTIC            Layer 3: CROSS-REFERENCE
─────────────────           ──────────────────           ────────────────────────
- Required fields present?  - Does value make sense?     - Does creator_id exist?
- String length within      - Is creation_year ≤         - Does collection_id
  limits?                     current year?                belong to this producer?
- No HTML/script injection? - Does locale match          - Is external_id unique
- Valid enum values?          BCP 47 format?               within this producer?
- UTF-8 encoding correct?  - Do tags match controlled   - Does license_type match
                              vocabulary (fuzzy)?           producer's plan?

Rejection:                  Warning + normalize:          Rejection:
Hard fail, 400              Fix silently or flag          Hard fail, 400/409
```

### 6.4 Normalization Best Practices

| Field | Normalization | Rationale |
|---|---|---|
| **All text** | Unicode NFC normalization, trim leading/trailing whitespace, collapse internal whitespace | Prevents duplicate entries differing only in Unicode representation or whitespace. |
| **Title** | Sentence case normalization (optional, depends on platform style) | Consistency in display. Some platforms (YouTube) preserve original case; others normalize. |
| **Tags** | Lowercase, deduplicate, sort alphabetically | Consistent tag comparison and display. |
| **Category/medium** | Fuzzy match against controlled vocabulary | "oil on canvas", "Oil painting", "oil/canvas" should all map to `OIL_ON_CANVAS`. Use Levenshtein distance or embedding similarity for fuzzy matching. |
| **Names (creator)** | Store both original (display) and normalized (search) forms | "René Magritte" → display: "René Magritte", search: "rene magritte". Enables accent-insensitive search while preserving correct display. |
| **Dates** | ISO 8601 | "March 2024", "03/24", "2024" → all normalized to structured date representation. |
| **Locale** | BCP 47 validation | "english" → `en`. "jp" → `ja` (common mistake). Invalid → default to platform primary locale. |

### 6.5 Sidecar Manifest Processing

For bulk uploads (museums uploading 10,000+ assets with a CSV or JSON manifest):

1. **Parse** the manifest and validate the header/schema.
2. **Match** each row to an uploaded asset by `filename` or an `external_id`.
3. **Validate** each row independently — partial success is the norm for large batches.
4. **Return a validation report:**

```json
{
  "total": 10000,
  "valid": 9847,
  "warnings": 120,
  "errors": 33,
  "error_details": [
    {
      "row": 1042,
      "filename": "painting_0573.tiff",
      "field": "creation_year",
      "value": "1890s",
      "error": "Expected integer year, got range. Normalized to 1895 (midpoint).",
      "severity": "warning"
    }
  ]
}
```

**Best practice:** Never reject an entire 10,000-row manifest because of one bad row. Validate independently, report comprehensively, and process the valid rows immediately.

### 6.6 Audit Trail

Store the **raw, unprocessed metadata** alongside the normalized version. This enables:
- Debugging normalization issues ("why was my title changed?")
- Re-processing if normalization rules are updated
- Compliance audits ("what did the producer originally submit?")

### 6.7 Emitted Event

```
AssetMetadataValidated {
  asset_id, locale, has_description, has_attribution,
  tag_count, license_type, validation_warnings_count
}
```

Consumed by: AI enrichment (Stage 5) — which needs to know what metadata already exists to avoid redundant generation.

---

## 7. Stage 4 — Media Processing & Variant Generation

### 7.1 Purpose

Transform the original uploaded asset into all renderable variants needed for every supported device class, resolution tier, and format.

This is typically the **most compute-intensive and cost-dominant** stage in the pipeline.

### 7.2 The Variant Strategy: Why One Size Does Not Fit All

A media platform serves consumers on wildly different devices. Serving the same file to all of them is either wasteful (sending 8K to a phone) or degraded (sending a tiny thumbnail to a 4K TV).

```
┌─────────────────────────────────────────────────────────────────────┐
│              DEVICE LANDSCAPE (example)                              │
│                                                                      │
│  Phone (375px)    Tablet (1024px)    Desktop (1920px)    4K TV       │
│  ┌─────┐          ┌─────────┐        ┌────────────┐     ┌────────┐  │
│  │     │          │         │        │            │     │        │  │
│  │thumb│          │ preview │        │ display_fhd│     │display │  │
│  │ _md │          │         │        │            │     │  _4k   │  │
│  │     │          │         │        │            │     │        │  │
│  └─────┘          └─────────┘        └────────────┘     └────────┘  │
│  WebP              WebP/AVIF         AVIF/WebP/JPEG     AVIF/JPEG   │
│  85 KB             250 KB            800 KB              3.5 MB     │
│                                                                      │
│  Smart TV / Digital Frame          Digital Pillar       8K Display   │
│  ┌──────────────────────┐          ┌──────┐            ┌──────────┐ │
│  │                      │          │      │            │          │ │
│  │     display_4k       │          │pillar│            │display_8k│ │
│  │                      │          │      │            │          │ │
│  └──────────────────────┘          │      │            │          │ │
│  AVIF/JPEG, 3.5 MB                │      │            └──────────┘ │
│                                    └──────┘            AVIF, 12 MB  │
│                                    AVIF, 1.5 MB                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.3 Still Image Processing

#### Recommended Variant Matrix

This matrix is a starting point. Adapt the specific dimensions and formats to your platform's device landscape.

| Variant ID | Max Bounding Box | Format(s) | Quality | Primary Use Case |
|---|---|---|---|---|
| `thumb_sm` | 200 × 200 | WebP | 75–80 | Grid thumbnails, search results, chat previews |
| `thumb_md` | 600 × 600 | WebP | 80–85 | Card previews, list items, social sharing |
| `preview` | 1200 × 1200 | WebP + AVIF | 82–85 | Detail view, web browsing, modal overlays |
| `display_fhd` | 1920 × 1080 | WebP + AVIF + JPEG | 88–92 | Full HD monitors, phones in landscape, tablets |
| `display_4k` | 3840 × 2160 | AVIF + JPEG | 90–95 | 4K monitors, TVs, high-res tablets |
| `display_8k` | 7680 × 4320 | AVIF + JPEG | 92–95 | 8K displays, large-format digital signage |
| `custom_aspect` | Varies | AVIF + WebP | 88–92 | Non-standard aspect ratios (vertical pillars, ultra-wide) |
| `original` | Original | Original | Lossless | Archival, downstream re-processing, print fulfillment |

#### Format Selection Guide

| Format | Strengths | Weaknesses | When to Use |
|---|---|---|---|
| **AVIF** | Best compression (30–50% smaller than JPEG at same quality), HDR support, wide color gamut | Slow encode (5–10× slower than JPEG), limited legacy browser support | Primary format for modern browsers/apps. Always generate alongside a fallback. |
| **WebP** | Good compression (25–35% smaller than JPEG), universal browser support, animation support | Not as efficient as AVIF, no HDR | Universal fallback. Generate for every variant. |
| **JPEG** | Universal compatibility (every device, every browser, every OS) | Largest file size, no transparency, no HDR | Legacy fallback. Generate for display variants only (not thumbnails). Required for Smart TVs, digital frames, older devices. |
| **PNG** | Lossless, transparency | Very large files | Only for assets that require transparency (logos, UI elements). Not recommended for photographs. |

#### Processing Engine Comparison

| Engine | Strengths | Weaknesses | Recommendation |
|---|---|---|---|
| **libvips** | Streaming architecture (very low memory), fastest for batch resize/encode, excellent quality | Less intuitive API, fewer format plugins than ImageMagick | **Primary choice.** Memory-safe for 500 MP+ images. |
| **ImageMagick** | Feature-rich, handles edge cases, supports 200+ formats | Memory-hungry (loads full image), slower | Fallback for exotic formats that libvips can't handle. |
| **Sharp** (Node.js/libvips binding) | Excellent developer experience, libvips performance | Node.js only | If your workers are Node.js/TypeScript. |
| **Pillow** (Python) | Easy to use, good for prototyping | Slow, memory-hungry at scale | Prototyping only. Not recommended for production pipelines. |

#### Processing Steps (Best Practice Order)

1. **Stream original** from object storage (never load entire file into memory).
2. **Auto-orient** based on EXIF orientation tag.
3. **Color management:** Convert to sRGB for display variants (universal compatibility). Preserve original ICC profile for archival and print variants.
4. **Resize** using Lanczos3 (best quality for downscaling) or Mitchell-Netravali (good balance of quality and speed).
5. **Smart crop** for thumbnails: Use saliency detection (attention-based cropping) to center on the most visually interesting region, rather than center-cropping.
6. **Encode** to target formats. AVIF encode is CPU-intensive — parallelize across cores.
7. **Generate blur hash** (compact string representation of a blurred placeholder) for progressive loading.
8. **Extract dominant colors** (k-means clustering on thumbnail, top 5 colors) for UI theming.
9. **Strip metadata** from all output variants (privacy). Do not carry EXIF/IPTC/XMP into consumer-facing files.
10. **Write variants** to processing scratch bucket.

### 7.4 Video Processing

Video transcoding is **the most expensive operation in the pipeline** (GPU-hours, not CPU-seconds). Cost control is paramount.

#### Adaptive Bitrate (ABR) Ladder

An ABR ladder defines the set of resolution/bitrate pairs that the video player can switch between based on the viewer's available bandwidth.

| Profile | Resolution | Codec | Target Bitrate | Use Case |
|---|---|---|---|---|
| `v_240p` | 426 × 240 | H.264 | 400 Kbps | Extreme low bandwidth, preview |
| `v_360p` | 640 × 360 | H.264 | 800 Kbps | Mobile on cellular |
| `v_480p` | 854 × 480 | H.264 | 1.5 Mbps | Standard definition |
| `v_720p` | 1280 × 720 | H.265/HEVC | 2.5 Mbps | Mobile on Wi-Fi, standard web |
| `v_1080p` | 1920 × 1080 | H.265/HEVC | 5 Mbps | Full HD displays |
| `v_1440p` | 2560 × 1440 | H.265/HEVC | 10 Mbps | QHD monitors |
| `v_4k` | 3840 × 2160 | H.265/HEVC | 15 Mbps | 4K TVs |
| `v_4k_hdr` | 3840 × 2160 | H.265 (HDR10) | 20 Mbps | HDR-capable displays |
| `v_8k` | 7680 × 4320 | AV1 | 30 Mbps | Future-proofing |

**Best practices:**
- **Only generate profiles ≤ source resolution.** A 720p source video should not get a 4K profile.
- **Per-title encoding:** Analyze source complexity and adjust bitrates. A static slideshow needs less bitrate than an action scene at the same resolution. Netflix pioneered this approach — it can reduce storage by 20–30%.
- **Use hardware-accelerated encoding** (NVENC, Intel QSV, Apple VideoToolbox) for H.264/H.265. Use software encoding for AV1 (hardware AV1 encoders are still maturing).

#### Streaming Packaging

| Protocol | Format | Compatibility | Recommendation |
|---|---|---|---|
| **HLS** | `.m3u8` + `.ts` segments | Universal (iOS, Android, web, TVs) | **Primary.** Required for Apple devices. |
| **DASH** | `.mpd` + `.m4s` segments | Android, web, TVs (not iOS Safari) | Secondary. Better for DRM (Widevine). |

- **Segment duration:** 6 seconds (industry standard). Shorter segments = faster quality switching but more HTTP requests. Longer = fewer requests but slower adaptation.
- **Generate both HLS and DASH manifests** from the same encoded segments to avoid re-encoding.

#### Audio Processing

| Consideration | Best Practice |
|---|---|
| **Loudness normalization** | Normalize to -14 LUFS (EBU R128 / ITU-R BS.1770). This prevents jarring volume differences between videos. Spotify, YouTube, and Apple Music all use this standard. |
| **Codec** | AAC-LC is universally supported. Encode at 128 Kbps stereo (primary) and 64 Kbps mono (low bandwidth). Opus is technically superior but has limited Smart TV support. |
| **No audio** | If the source has no audio track, omit the audio track from output (saves bandwidth). Flag in metadata. |

#### Video Thumbnail Generation

1. Extract a keyframe at ~10% duration (avoids black intro screens and end credits).
2. Generate 10 evenly-spaced preview frames for scrub/preview sprites.
3. Compose sprites into a single WebP sprite sheet (reduces request count for scrub preview).
4. Process the primary thumbnail through the still image variant pipeline (generate `thumb_sm`, `thumb_md`, `preview`).

#### HDR Handling

```
┌──────────────────────────────────────────────────────────────────────┐
│                 HDR VIDEO PROCESSING FLOW                            │
│                                                                      │
│  Source has HDR metadata?                                            │
│  ┌─────┐                                                            │
│  │ YES │──▶ Preserve HDR in dedicated profile (v_4k_hdr)            │
│  └──┬──┘    ──▶ Generate SDR tone-mapped versions for all other     │
│     │           profiles (use Hable or BT.2446 tone mapping)        │
│     │                                                                │
│  ┌──▼──┐                                                            │
│  │ NO  │──▶ Process normally (all profiles are SDR)                  │
│  └─────┘                                                            │
│                                                                      │
│  IMPORTANT: Tone mapping is lossy. Always preserve the original     │
│  HDR source in the archival bucket for future re-processing.        │
└──────────────────────────────────────────────────────────────────────┘
```

### 7.5 Variant Manifest

After all variants are generated, create a machine-readable manifest that catalogs everything produced:

```json
{
  "asset_id": "019abc...",
  "original": {
    "format": "image/tiff",
    "width": 8000,
    "height": 6000,
    "file_size_bytes": 145000000,
    "color_space": "adobe_rgb",
    "storage_key": "originals/..."
  },
  "variants": [
    {
      "variant_id": "thumb_sm",
      "format": "image/webp",
      "width": 200,
      "height": 150,
      "file_size_bytes": 8500,
      "storage_key": "serving/v1/.../img/thumb_sm.webp",
      "blur_hash": "LEHV6nWB2yk8pyo0adR*.7kCMdnj"
    }
  ],
  "color_palette": [
    { "hex": "#2C5F8A", "name": "steel blue", "percentage": 0.32 },
    { "hex": "#D4A574", "name": "warm beige", "percentage": 0.24 }
  ],
  "processing_cost": {
    "cpu_seconds": 12.4,
    "gpu_seconds": 0,
    "storage_delta_bytes": 2400000
  }
}
```

### 7.6 Emitted Event

```
AssetVariantsGenerated {
  asset_id, variant_count, total_variant_size_bytes,
  has_video, has_hdr, processing_duration_sec
}
```

---

## 8. Stage 5 — AI/ML Enrichment

### 8.1 Purpose

Generate machine-derived metadata that enhances discoverability: classification tags, natural-language descriptions, content moderation signals, and vector embeddings for semantic search and visual similarity.

### 8.2 AI Enrichment Tasks

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     AI/ML ENRICHMENT TASKS                               │
│                                                                          │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐             │
│  │  CLASSIFICATION│  │  DESCRIPTION   │  │   CONTENT      │             │
│  │  & TAGGING     │  │  GENERATION    │  │   MODERATION   │             │
│  │                │  │                │  │                │             │
│  │  Style, mood,  │  │  Natural-lang  │  │  NSFW, violence│             │
│  │  subject,      │  │  description   │  │  hate speech,  │             │
│  │  color, era    │  │  of the asset  │  │  PII in image  │             │
│  └────────────────┘  └────────────────┘  └────────────────┘             │
│                                                                          │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐             │
│  │  VECTOR        │  │  OBJECT        │  │  TEXT           │             │
│  │  EMBEDDING     │  │  DETECTION     │  │  EXTRACTION    │             │
│  │                │  │                │  │  (OCR)         │             │
│  │  For semantic  │  │  Identify      │  │  Extract text  │             │
│  │  search &      │  │  objects,      │  │  from images   │             │
│  │  similarity    │  │  faces, text   │  │  for indexing  │             │
│  └────────────────┘  └────────────────┘  └────────────────┘             │
└──────────────────────────────────────────────────────────────────────────┘
```

Not every platform needs every task. Select based on your product's discovery and safety requirements:

| Task | Pinterest | YouTube | E-Commerce | Digital Art | News Media |
|---|---|---|---|---|---|
| Classification/Tagging | Required | Required | Required | Required | Required |
| Description Generation | Optional | Optional (auto-captions) | Required (SEO) | Required | Optional |
| Content Moderation | Required | Required | Required | Required | Required |
| Vector Embedding | Required | Required | Optional | Required | Optional |
| Object Detection | Required | Optional | Required (product ID) | Optional | Required (faces) |
| OCR/Text Extraction | Optional | Required (thumbnails) | Required (labels) | Optional | Required |

### 8.3 LLM vs. Specialized Model Decision

| Approach | Strengths | Weaknesses | Best For |
|---|---|---|---|
| **LLM (multimodal)** | One model handles tagging, description, and moderation simultaneously. Nuanced understanding. Flexible prompt-based output. | Expensive per-call. Unpredictable latency. Token budgets needed. | High-value assets where nuanced understanding matters (art, editorial, premium content). |
| **Specialized CV models** | Cheap (self-hosted, amortized GPU). Predictable latency. Deterministic output. | One model per task. Less nuanced. Retraining needed for new categories. | High-volume, cost-sensitive platforms (social media, e-commerce). |
| **Hybrid** | Best accuracy. LLM for nuanced tasks, specialized models for commodity tasks. | Operational complexity — two inference stacks. | Recommended for most platforms at scale. |

### 8.4 Cost Control: The Most Important Section

AI enrichment can easily become the most expensive stage in the pipeline. Without controls, it will destroy your unit economics.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    AI COST CONTROL FRAMEWORK                             │
│                                                                          │
│  1. BATCH, DON'T STREAM                                                 │
│     ─────────────────────                                               │
│     Queue assets and submit to LLM providers in batches.                │
│     Most providers offer 50% discount on batch/async APIs.              │
│     Batch size: 50–100 assets per API call.                             │
│     Acceptable latency: minutes to hours (not real-time).               │
│                                                                          │
│  2. TIERED TOKEN BUDGETS                                                │
│     ─────────────────────                                               │
│     Classification: 300–500 output tokens max                           │
│     Description: 200–400 output tokens max                              │
│     Don't let an LLM write a 2000-word essay about a thumbnail.        │
│                                                                          │
│  3. SEMANTIC CACHING                                                    │
│     ──────────────────                                                  │
│     Hash the input (image embedding + prompt) → check cache.            │
│     If a visually similar image was recently tagged, reuse tags.        │
│     Expected hit rate for platforms with repeated content: 20–40%.      │
│                                                                          │
│  4. SKIP ENRICHMENT FOR LOW-VALUE ASSETS                                │
│     ────────────────────────────────────────                            │
│     Not every asset needs full LLM enrichment.                          │
│     If basic CV models (CLIP, object detection) produce high-           │
│     confidence tags, skip the LLM call entirely.                        │
│     LLM enrichment only for assets where CV confidence < threshold.     │
│                                                                          │
│  5. MONTHLY BUDGET CAPS                                                 │
│     ────────────────────                                                │
│     Hard cap on LLM spend per month. When exhausted:                    │
│     - Fall back to CV-only enrichment                                   │
│     - Queue for next month's budget                                     │
│     - Or: promote without AI tags (graceful degradation)                │
│                                                                          │
│  6. RE-ENRICHMENT IS CHEAP LATER                                        │
│     ──────────────────────────────                                      │
│     LLM prices drop ~50% annually. Assets enriched with basic tags      │
│     today can be re-enriched with better/cheaper models next year.      │
│     Don't over-invest in day-one enrichment.                            │
└──────────────────────────────────────────────────────────────────────────┘
```

### 8.5 Vector Embeddings for Search & Similarity

Vector embeddings are the foundation of "find more like this" and semantic search ("moody blue painting of a rainy street").

| Model | Embedding Dim | Modality | Hosting | Best For |
|---|---|---|---|---|
| **CLIP (OpenCLIP ViT-L/14)** | 768 | Image + Text (cross-modal) | Self-hosted GPU | Visual similarity + text-to-image search |
| **SigLIP** | 1024 | Image + Text | Self-hosted GPU | Higher quality than CLIP, same use case |
| **DINOv2** | 768 | Image only | Self-hosted GPU | Pure visual similarity (ignores semantics) |
| **Multilingual-E5** | 1024 | Text only (multilingual) | Self-hosted GPU/CPU | Multilingual text search |

**Best practice:** Generate **two embeddings** per asset:
1. **Image embedding** (CLIP/SigLIP): for visual similarity search ("find more like this").
2. **Text embedding** (from concatenated title + description + tags): for keyword-augmented semantic search.

Store both in a vector database with asset metadata as filterable payload.

### 8.6 Content Moderation

**This is a safety requirement, not a nice-to-have.** Every user-generated content platform must scan uploads for:

| Category | Detection Method | Action |
|---|---|---|
| NSFW content | Specialized classifier (e.g., NSFW detection model) + LLM review for edge cases | Flag, require producer to confirm content rating, restrict visibility |
| Violence/gore | Specialized classifier + LLM review | Flag, restrict or remove |
| Hate symbols | Object detection + LLM review | Flag, escalate to human moderation |
| PII in images | OCR + PII detection (credit cards, SSN patterns, faces) | Flag, blur detected PII regions |
| Copyright infringement | Perceptual hashing (pHash, dHash) against known-works database | Flag, escalate to rights team |

**Best practice:** Content moderation should run on the **hot path** — it must complete before the asset becomes visible. An asset that passes all other stages but fails moderation must not be served to consumers.

### 8.7 Emitted Event

```
AssetEnriched {
  asset_id, tag_count, has_ai_description, embedding_stored,
  moderation_result (PASS/FLAG/REJECT), enrichment_model,
  token_usage, cost_usd
}
```

---

## 9. Stage 6 — Localization

### 9.1 Purpose

Translate producer-supplied and AI-generated metadata into all supported display languages, and where necessary, create locale-specific content advisories.

### 9.2 When to Include This Stage

| Platform Type | Localization Need |
|---|---|
| Global consumer platform (100M+ users across languages) | **Required.** Users search and browse in their language. |
| Single-market platform (US-only, Japan-only) | **Skip.** Add later when expanding internationally. |
| Enterprise DAM (internal) | **Optional.** Depends on whether the organization operates multilingually. |

### 9.3 What Gets Localized

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    LOCALIZATION SCOPE                                     │
│                                                                          │
│  ALWAYS LOCALIZE:            LOCALIZE ON DEMAND:       NEVER LOCALIZE:  │
│  ─────────────────           ───────────────────       ─────────────────│
│  • Title                     • Long descriptions       • Technical       │
│  • Short description         • Provenance/history        metadata        │
│  • AI-generated tags         • Legal terms             • File names      │
│  • Category names            • Cultural advisories     • Internal IDs    │
│  • UI labels (client-side)   • Producer bios           • API responses   │
│                                                         (use Accept-     │
│                                                          Language)       │
└──────────────────────────────────────────────────────────────────────────┘
```

### 9.4 Translation Strategy: Tiered by Cost

| Content Type | Volume | Method | Cost | Quality |
|---|---|---|---|---|
| **Controlled vocabulary** (tags, categories, enums) | Low (finite set) | Pre-translated dictionary maintained by linguists | Near-zero (one-time cost) | Highest (human-curated) |
| **Short text** (titles ≤ 200 chars) | High | LLM translation in batch | Low (batch API pricing, short output) | High |
| **Long text** (descriptions, bios) | Medium | LLM translation in batch | Medium | High |
| **Domain-specific** (legal terms, provenance) | Low | Professional human translation | High | Highest (required for legal accuracy) |

**Anti-pattern:** Translating every field with an LLM call per-asset per-locale in real-time. At 100,000 assets × 10 locales = 1,000,000 LLM calls. Use batch APIs and dictionary lookups to reduce this by 90%.

### 9.5 Multilingual Search Index

For each localized metadata record, generate a **locale-specific text embedding** and store it in the vector database with `locale` as a filterable field. This enables:
- A Japanese user searching in Japanese finds French art with Japanese-translated metadata.
- Search quality is consistent across all languages (not just the source language).

### 9.6 Cultural Sensitivity

Some content may be acceptable in one culture but sensitive in another. The localization stage should produce **locale-specific content advisories** based on:

| Concern | Example | Handling |
|---|---|---|
| Nudity norms | Classical Renaissance nude painting | Advisory in conservative markets, no advisory in European markets |
| Religious imagery | Christian iconography, Hindu deities | Advisory in markets where religious imagery in commercial contexts is sensitive |
| Political symbols | Historical flags, political art | Advisory where legally required |

**Best practice:** Content advisories are metadata, not censorship. The asset is still available — the consumer sees an advisory before viewing. Only hard-block content that violates local law.

### 9.7 Emitted Event

```
AssetLocalized {
  asset_id, locales_completed: ["en","es","fr",...],
  pending_locales: ["ar"], translation_method_used
}
```

---

## 10. Stage 7 — Rights, Compliance & Access Control

### 10.1 Purpose

Attach licensing and rights metadata, generate access control artifacts (watermarks, DRM tokens, signed URL configurations), and ensure the asset complies with applicable regulations.

### 10.2 When to Include This Stage

| Platform Type | Rights Management Need |
|---|---|
| Licensed content (stock photography, digital art, music) | **Required.** Core business function. |
| User-generated content (social media) | **Minimal.** Basic copyright attribution. DMCA takedown handling. |
| Enterprise DAM | **Required.** Brand asset usage policies, approval workflows. |
| E-commerce | **Moderate.** Product image usage rights for syndication. |

### 10.3 Access Control Mechanisms

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    ACCESS CONTROL LAYERS                                  │
│                                                                          │
│  Layer 1: SIGNED URLS                                                   │
│  ─────────────────────                                                  │
│  Time-limited, cryptographically signed URLs that expire.               │
│  The CDN edge validates the signature before serving content.           │
│  Parameters: expiry time, allowed IP range (optional),                  │
│  allowed referrer (optional).                                           │
│                                                                          │
│  Layer 2: VISIBLE WATERMARKING                                          │
│  ────────────────────────────                                           │
│  Semi-transparent overlay (logo, text) on preview/display variants.     │
│  Deters casual screenshot-sharing. Removed upon license purchase.       │
│  Applied at the variant level — watermarked variants are separate       │
│  files from unwatermarked ones (immutable variant principle).           │
│                                                                          │
│  Layer 3: INVISIBLE (FORENSIC) WATERMARKING                            │
│  ───────────────────────────────────────────                            │
│  Steganographic watermark encoding license ID into pixel data.          │
│  Survives JPEG recompression, resizing, and screen capture.            │
│  Algorithms: DWT-based (Discrete Wavelet Transform), or                │
│  learned watermarks (e.g., StegaStamp).                                │
│  Enables forensic tracking of leaked content.                          │
│                                                                          │
│  Layer 4: HARDWARE DRM                                                  │
│  ─────────────────────                                                  │
│  Widevine (Google), FairPlay (Apple), PlayReady (Microsoft).           │
│  Encrypted content that only licensed players can decrypt.             │
│  Highest protection but adds significant complexity and licensing      │
│  cost. Required for premium video content (studios, broadcasters).     │
│                                                                          │
│  RECOMMENDATION: Start with Layers 1+2 for stills, Layers 1+3 for     │
│  commercial licensing. Add Layer 4 only if enterprise clients demand   │
│  hardware-level DRM.                                                    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 10.4 Compliance Considerations

| Regulation | Requirement | Pipeline Impact |
|---|---|---|
| **GDPR** (EU) | Right to deletion, data minimization, data residency | Must support full asset deletion across all storage tiers, CDN caches, vector databases, and search indexes. EU user data may need to stay in EU storage regions. |
| **DMCA** (US) | Takedown process for copyright-infringing content | Must support rapid takedown (asset status → `TAKEN_DOWN`, CDN purge, search de-index). |
| **CCPA** (California) | Right to know, right to delete | Similar to GDPR requirements. |
| **Content regulations** (varies by jurisdiction) | Age-gating, content warnings | Content moderation results from Stage 5 feed into geo-specific access policies. |

### 10.5 Asset Deletion Protocol

Deletion in a media pipeline is not a simple `DELETE FROM assets WHERE id = ?`. It requires coordination across every storage tier:

```
Asset Deletion Checklist:
  □ Mark asset as DELETED in primary database
  □ Remove from search index (vector DB + full-text)
  □ Purge from CDN edge caches (all PoPs)
  □ Delete all variants from serving bucket
  □ Delete original from originals bucket (or retain per legal hold)
  □ Remove from all feed/recommendation caches
  □ Remove from analytics (or anonymize)
  □ Revoke all active signed URLs / DRM tokens
  □ Notify downstream systems (3P integrations, webhooks)
  □ Log deletion event for audit trail
```

**Best practice:** Implement deletion as a **soft delete first** (mark as deleted, remove from all consumer-facing surfaces) followed by a **hard delete after a grace period** (30 days). This allows recovery from accidental deletions.

### 10.6 Emitted Event

```
AssetRightsAttached {
  asset_id, license_type, watermark_visible, watermark_invisible,
  drm_type, geo_restrictions, signed_url_ttl
}
```

---

## 11. Stage 8 — Final Storage Layout & CDN Propagation

### 11.1 Purpose

Move finalized variants from the processing scratch space to the serving storage tier, configure CDN caching rules, and optionally warm edge caches for high-priority assets.

### 11.2 Storage Tier Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    STORAGE TIER ARCHITECTURE                                  │
│                                                                              │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐        │
│  │   ORIGINALS     │     │   PROCESSING    │     │    SERVING      │        │
│  │   BUCKET        │     │   BUCKET        │     │    BUCKET       │        │
│  │                 │     │   (scratch)     │     │    (CDN origin) │        │
│  │  Write-once     │     │  High I/O       │     │  Read-heavy     │        │
│  │  Archival tier  │     │  Ephemeral      │     │  Hot storage    │        │
│  │  after 90 days  │     │  Deleted after  │     │  Immutable      │        │
│  │                 │     │  pipeline done  │     │  variants       │        │
│  │  Retention:     │     │  Retention:     │     │  Retention:     │        │
│  │  Forever        │     │  24 hours       │     │  While active   │        │
│  └─────────────────┘     └─────────────────┘     └─────────────────┘        │
│                                                         │                    │
│                                                    ┌────▼──────────────┐     │
│                                                    │     CDN EDGE     │     │
│                                                    │   (270+ PoPs)    │     │
│                                                    │                  │     │
│                                                    │  Immutable       │     │
│                                                    │  cache rules     │     │
│                                                    │  Format negot.   │     │
│                                                    │  Signed URL      │     │
│                                                    │  validation      │     │
│                                                    └──────────────────┘     │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 11.3 Object Key Design

The storage key (path) structure affects performance, cost, and operability. A good key structure enables:
- **Efficient listing** (all variants for an asset share a prefix).
- **Access control** (prefix-based IAM policies per producer).
- **Cache invalidation** (purge by prefix).

**Recommended structure:**
```
serving/v{api_version}/{asset_id}/
  ├── meta.json
  ├── img/{variant_id}.{format}
  ├── video/{protocol}/{profile}/...
  └── wm/{variant_id}.{format}
```

**Key design anti-patterns:**
- Using sequential numeric IDs as key prefixes (creates hot partitions in object storage).
- Deeply nested directory structures (increases `LIST` latency).
- Including mutable data in keys (breaks immutability principle).

### 11.4 CDN Cache Strategy

| Content Type | Cache-Control Header | Rationale |
|---|---|---|
| **Immutable variants** (thumbnails, display images, video segments) | `public, max-age=31536000, immutable` | Content-addressed — never changes. Maximum cache efficiency. |
| **Variant manifest** (meta.json) | `public, max-age=300` (5 min) | May update when new variants are added (e.g., 8K variant generated later). |
| **Watermarked previews** | `public, max-age=86400` (1 day) | May need to be regenerated if watermark design changes. |
| **Streaming manifests** (.m3u8, .mpd) | `public, max-age=86400` (1 day) | May update if new profiles are added. |
| **Streaming segments** (.ts, .m4s) | `public, max-age=31536000, immutable` | Content-addressed, never change. |

### 11.5 Content Negotiation at the Edge

Modern CDN edge compute (Cloudflare Workers, AWS Lambda@Edge, Fastly Compute@Edge) enables **per-request content negotiation** without origin round-trips:

```
┌──────────────────────────────────────────────────────────────────────────┐
│              EDGE CONTENT NEGOTIATION FLOW                                │
│                                                                          │
│  Client Request:                                                        │
│    Accept: image/avif, image/webp, image/jpeg                           │
│    Sec-CH-Width: 1920                                                   │
│    Sec-CH-DPR: 2                                                        │
│    Sec-CH-Viewport-Width: 960                                           │
│                                                                          │
│  Edge Worker Logic:                                                     │
│    1. Parse Accept header → client supports AVIF                        │
│    2. Effective width = viewport_width × DPR = 960 × 2 = 1920          │
│    3. Select variant: display_fhd (1920px) in AVIF format               │
│    4. Check if signed URL is required for this variant                  │
│    5. Validate signed URL token if required                             │
│    6. Serve from edge cache (or fetch from origin on miss)              │
│    7. Set Vary: Accept, Sec-CH-Width, Sec-CH-DPR                       │
│                                                                          │
│  Response:                                                               │
│    Content-Type: image/avif                                             │
│    Cache-Control: public, max-age=31536000, immutable                   │
│    X-Served-Variant: display_fhd.avif                                   │
└──────────────────────────────────────────────────────────────────────────┘
```

### 11.6 Edge Cache Warming

For high-priority assets, proactively populate edge caches before consumers request them:

| Trigger | Which Variants to Warm | Which PoPs |
|---|---|---|
| Asset added to "Featured" collection | `thumb_md`, `preview`, `display_fhd` | Top 20 PoPs by traffic volume |
| Asset trending (> N views/hour) | `preview`, `display_fhd` | PoPs where traffic is originating |
| New upload from top-tier producer | `thumb_md`, `preview` | Global top 10 PoPs |

**Anti-pattern:** Warming all variants in all PoPs for every asset. This wastes CDN origin bandwidth and cache storage. Warm selectively.

### 11.7 Scratch Cleanup

After all variants are confirmed in the serving bucket, **delete the processing scratch data** from the processing bucket. This is a cost-saving measure — scratch data can be 2–5× the size of the final variants (intermediate transcode outputs, temporary files).

Implement as a **delayed cleanup** (run 1 hour after pipeline completion) to allow for any late retry operations.

### 11.8 Emitted Event

```
AssetCDNReady {
  asset_id, serving_base_url, variant_count,
  total_serving_bytes, cdn_warm_pops
}
```

---

## 12. Stage 9 — Indexing & Discoverability

### 12.1 Purpose

Make the fully processed asset discoverable through search, browsable in feeds, available via API, and visible to the producer. This is the **terminal stage** — after this, the asset is live.

### 12.2 Index Update Checklist

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    INDEXING OPERATIONS                                    │
│                                                                          │
│  ┌─────────────────────┐  ┌─────────────────────┐                       │
│  │  1. DATABASE         │  │  2. VECTOR INDEX     │                       │
│  │  Read Model Update   │  │  Finalization        │                       │
│  │                      │  │                      │                       │
│  │  Upsert denormalized │  │  Confirm embedding   │                       │
│  │  read model with all │  │  exists in vector DB │                       │
│  │  metadata, variant   │  │  Update payload with │                       │
│  │  URLs, tags, locales │  │  final metadata,     │                       │
│  │                      │  │  locale, license     │                       │
│  └─────────────────────┘  └─────────────────────┘                       │
│                                                                          │
│  ┌─────────────────────┐  ┌─────────────────────┐                       │
│  │  3. CACHE PRIMING   │  │  4. FEED INJECTION   │                       │
│  │                      │  │                      │                       │
│  │  Prime read cache    │  │  If producer has     │                       │
│  │  for top locales     │  │  followers: inject   │                       │
│  │  (reduces first-     │  │  into follower feed  │                       │
│  │   request latency)   │  │  queues              │                       │
│  │                      │  │                      │                       │
│  └─────────────────────┘  └─────────────────────┘                       │
│                                                                          │
│  ┌─────────────────────┐  ┌─────────────────────┐                       │
│  │  5. WEBHOOK          │  │  6. STATUS UPDATE    │                       │
│  │  DELIVERY            │  │                      │                       │
│  │                      │  │  Asset: READY        │                       │
│  │  Notify 3P devs     │  │  Job: update counts  │                       │
│  │  subscribed to       │  │  Notify producer     │                       │
│  │  asset.ready events  │  │  (push + email)      │                       │
│  └─────────────────────┘  └─────────────────────┘                       │
└──────────────────────────────────────────────────────────────────────────┘
```

### 12.3 The Denormalized Read Model

For high-read-throughput platforms, the consumer-facing API should read from a **denormalized read model** — a single table/document that contains everything needed to render an asset card, search result, or detail view without joins.

**Fields to include:**
- Asset identity (ID, producer ID, creator name)
- Display metadata (title, description — in requested locale)
- Discovery metadata (tags, categories, colors)
- Visual artifacts (thumbnail URLs, preview URL, blur hash, dominant color)
- Technical info (dimensions, has video, video duration)
- Access info (license type, available locales, serving base URL)
- Ranking signals (popularity score, creation date, indexed date)

**Indexing strategy:**
- **Array/tag indexes** (GIN in PostgreSQL, multi-value in Elasticsearch) on tags and categories for containment queries.
- **Full-text search index** on `title + description` as a fallback search mechanism.
- **Composite indexes** on common filter combinations (e.g., `license_type + style_tag + creation_year`).

### 12.4 Feed Injection

When an asset becomes `READY`, it should appear in relevant feeds:

| Feed Type | Injection Logic | Storage |
|---|---|---|
| **Producer's own gallery** | Always — every `READY` asset appears in the producer's public gallery | Database query (by producer_id, ordered by created_at) |
| **Follower feeds** | If the producer has followers, fan out the asset ID into each follower's feed | Cache (sorted set, scored by timestamp). Fan-out on write for small follower counts (< 10K). Fan-out on read for large follower counts. |
| **Category/trending feeds** | If the asset's tags match trending categories | Cache (sorted set, scored by popularity). Async job evaluates trending eligibility. |
| **Personalized recommendations** | Based on user taste profile | Deferred to recommendation engine (cold path). |

### 12.5 Final Status Update

```
Asset:
  status → READY
  indexed_at → NOW()

IngestionJob:
  success_count += 1
  IF success_count + failure_count == asset_count:
    IF success_count / asset_count >= 0.95:
      status → READY
    ELSE:
      status → PARTIAL_FAILURE
```

### 12.6 Terminal Event

```
AssetReady {
  asset_id, producer_id, ingestion_job_id,
  time_to_ready_sec, stages_completed, degraded_stages
}
```

This is the **final event** in the ingestion pipeline. No further pipeline stages consume it. It is consumed by:
- Analytics (time-to-ready tracking, throughput measurement)
- Producer notification service
- 3P webhook delivery

---

# Part III: Cross-Cutting Concerns

## 13. The Asset State Machine

Every asset in the pipeline has a well-defined state. The state machine enforces valid transitions and prevents assets from entering impossible states.

```
                            ┌──────────────┐
                            │ PENDING_     │
                ┌──────────▶│ UPLOAD       │──── (expires) ──────────▶ EXPIRED
                │           └──────┬───────┘
       Upload   │                  │ binary received + verified
       initiated│           ┌──────▼───────┐
                │           │  UPLOADED     │──── (integrity fail) ──▶ QUARANTINED
                │           └──────┬───────┘
                │                  │ integrity verified
                │           ┌──────▼───────┐
                │           │  PROCESSING  │──── (all retries fail) ─▶ FAILED
                │           └──────┬───────┘
                │                  │ variants generated
                │           ┌──────▼───────┐
                │           │  ENRICHING   │──── (LLM fail, retries
                │           └──────┬───────┘     exhausted) ──▶ READY
                │                  │                             (degraded)
                │                  │ enrichment complete
                │           ┌──────▼───────┐
                │           │  LOCALIZING  │──── (fail) ──▶ READY
                │           └──────┬───────┘               (source-lang only)
                │                  │ localization complete
                │           ┌──────▼───────┐
                │           │  FINALIZING  │
                │           └──────┬───────┘
                │                  │ CDN ready + indexed
                │           ┌──────▼───────┐
                │           │    READY      │
                │           └──────────────┘

  GLOBAL TRANSITIONS (from any state):
    ──── (malware detected)    ──▶ QUARANTINED
    ──── (producer deletes)    ──▶ DELETED
    ──── (DMCA/legal takedown) ──▶ TAKEN_DOWN
    ──── (content moderation)  ──▶ MODERATION_REVIEW
```

### State Machine Rules

1. **Forward-only on the happy path.** Assets progress through states in order. No state can transition backward except via explicit operator action (e.g., re-enrichment).
2. **Global transitions override the happy path.** Quarantine, deletion, and takedown can happen from any state.
3. **Degraded promotion is intentional.** When a non-critical stage fails (enrichment, localization), the asset is promoted to `READY` in a degraded state. The missing enrichment is recorded and retried later on the cold path.
4. **State transitions are transactional.** The database state update and the next-stage event emission happen in the same transaction (or with an outbox pattern). An asset must never be in a state that disagrees with the events emitted about it.

### Error Handling by Stage

| Stage | Max Retries | Backoff Strategy | On Exhaustion |
|---|---|---|---|
| Integrity Verification | 3 | Exponential: 1s → 2s → 4s | `QUARANTINED` (suspect file) |
| Metadata Validation | 2 | Immediate | Flag for producer review |
| Media Processing | 3 | Exponential: 30s → 60s → 120s | `FAILED` + operator alert |
| AI Enrichment | 5 | Exponential: 1m → 2m → 5m → 10m → 30m | Promote to `READY` without AI tags |
| Localization | 3 | Exponential: 30s → 60s → 120s | Promote to `READY` with source language only |
| Rights/DRM | 2 | Exponential: 10s → 30s | Block publishing, operator alert |
| CDN Propagation | 5 | Exponential: 10s → 30s → 1m → 2m → 5m | Retry from scratch |
| Indexing | 3 | Exponential: 5s → 15s → 30s | `READY` but not searchable, operator alert |

### Dead Letter Queue Strategy

Every stage has a dead letter queue (DLQ). When an event exhausts its retries, it moves to the DLQ.

**DLQ requirements:**
- **Retain the full event payload** including the original message, all retry attempts, and the error from each attempt.
- **Dashboard visibility:** DLQ depth per stage must be visible in the observability dashboard with alerting when depth exceeds thresholds.
- **Replay capability:** Operators must be able to replay DLQ messages after fixing the root cause. Replay should be idempotent (replaying an already-processed message is a no-op).
- **Expiry:** DLQ messages older than 30 days are archived to cold storage (not deleted — they may be needed for post-mortems).

---

## 14. Cost Modeling at Scale

### 14.1 Per-Asset Cost Breakdown

The following table provides order-of-magnitude costs. Actual costs vary significantly by cloud provider, negotiated discounts, and instance types.

| Stage | Still Image (10 MB avg) | Video (2 GB avg, 5 min) | Cost Driver |
|---|---|---|---|
| Object storage (original) | $0.00002 | $0.003 | $/GB/month × retention |
| Integrity + extraction | $0.0001 | $0.001 | CPU seconds |
| Variant generation | $0.002 | $0.15 | CPU/GPU seconds |
| AI enrichment (LLM) | $0.003–$0.01 | $0.003–$0.01 | Tokens consumed |
| AI enrichment (embeddings) | $0.0005 | $0.0005 | GPU seconds (amortized) |
| Localization (10 locales) | $0.005–$0.01 | $0.005–$0.01 | LLM tokens + dictionary |
| Watermarking | $0.0002 | $0.001 | CPU seconds |
| Object storage (variants) | $0.0001 | $0.02 | $/GB/month × variant count |
| CDN egress | $0.00 (zero-egress provider) — $0.005 | $0.00 — $0.05 | $/GB egress × view count |
| **Total ingestion cost** | **$0.01–$0.03** | **$0.15–$0.25** | |

### 14.2 The Dominant Cost Drivers

At scale, three cost drivers dominate:

```
┌─────────────────────────────────────────────────────────────────────┐
│               COST DOMINANCE AT SCALE                               │
│                                                                     │
│  60-70%  ┌──────────────────────────────────────────────────────┐  │
│          │  VIDEO TRANSCODING (GPU hours)                        │  │
│          │  Mitigation: Per-title encoding, skip unnecessary    │  │
│          │  profiles, use hardware encoders, spot instances     │  │
│          └──────────────────────────────────────────────────────┘  │
│                                                                     │
│  15-25%  ┌──────────────────────────────────────────────────────┐  │
│          │  AI/LLM API CALLS (token costs)                      │  │
│          │  Mitigation: Batch API, semantic caching, tiered     │  │
│          │  budgets, skip LLM when CV confidence is high        │  │
│          └──────────────────────────────────────────────────────┘  │
│                                                                     │
│  10-15%  ┌──────────────────────────────────────────────────────┐  │
│          │  STORAGE + CDN EGRESS (ongoing, grows with catalog)  │  │
│          │  Mitigation: Zero-egress storage providers, archival │  │
│          │  tiers for originals, delete scratch promptly        │  │
│          └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 14.3 Cost Scaling Model

| Scale | Monthly Assets | Monthly Ingestion Cost (blended still + video) | Notes |
|---|---|---|---|
| Startup | 10,000 | $500–$2,000 | Manageable. Don't over-optimize yet. |
| Growth | 100,000 | $5,000–$20,000 | Optimization begins to matter. |
| Scale | 1,000,000 | $50,000–$200,000 | Per-asset cost reduction is now a business priority. |
| Hyper-scale | 10,000,000+ | $500,000–$2,000,000 | Dedicated cost engineering team. Spot instances, reserved GPU capacity, negotiated API pricing. |

---

## 15. The PM Decision Checklist

This checklist is for product managers planning features that affect the ingestion pipeline. Each decision has cost, timeline, and complexity implications.

### Before You Start: Questions to Answer

| # | Question | Why It Matters | Who Decides |
|---|---|---|---|
| 1 | **What media types do we support?** (stills only? video? audio? 3D?) | Each media type requires a separate processing pipeline with different compute profiles. Adding video transcoding is a 3–6 month engineering effort. | PM + Eng Lead |
| 2 | **What are the supported upload sizes?** (max file size per type) | Determines whether you need multipart upload, changes storage and processing cost estimates significantly. A 50 GB video costs 100× more to process than a 5 MB image. | PM + Eng Lead |
| 3 | **What devices do we serve?** (phones, tablets, desktops, TVs, digital signage?) | Each device class may require a dedicated variant. More devices = more variants = more storage and processing cost. | PM + Design |
| 4 | **What's the time-to-visible target?** (seconds? minutes? hours?) | Determines whether AI enrichment and localization are on the hot path or cold path. Sub-minute targets may exclude LLM enrichment from the hot path. | PM |
| 5 | **Do we need AI enrichment? What kind?** | Tagging, descriptions, embeddings, moderation — each has different cost profiles. LLM-based enrichment is 10–100× more expensive than specialized CV models. | PM + ML Lead |
| 6 | **How many languages?** | Each locale multiplies translation cost. 10 locales = 10× the metadata translation work. Tags can use pre-translated dictionaries (cheap). Descriptions need LLM or human translation (expensive). | PM + i18n Lead |
| 7 | **Do we need rights management / DRM?** | Watermarking and signed URLs add pipeline stages, storage overhead (watermarked variants), and CDN configuration complexity. Hardware DRM (Widevine) adds months of work and licensing cost. | PM + Legal |
| 8 | **What's the expected upload volume?** (assets/day, peak burst) | Determines infrastructure sizing, auto-scaling parameters, and cost projections. A 10× estimate error means either over-provisioning (wasted money) or under-provisioning (outages). | PM + Data |
| 9 | **Is bulk upload required?** | Bulk upload (10,000+ assets in a single job) requires manifest processing, partial success handling, and progress tracking — all non-trivial engineering work. | PM |
| 10 | **What does "delete" mean?** | Full deletion across all tiers is complex (see Stage 7). Legal holds may require retaining originals even after producer deletion. GDPR requires deletion within 30 days. | PM + Legal |

### Feature Cost Impact Matrix

When a PM proposes a new feature, estimate the pipeline impact using this matrix:

| Feature Request | Pipeline Stages Affected | Cost Impact | Timeline Impact |
|---|---|---|---|
| "Add 8K support" | Stage 4 (processing), Stage 8 (CDN, storage) | +$0.05/still, +$5/video (GPU hours for 8K encode) | 2–4 weeks |
| "Add video support" (to an image-only platform) | Stages 2, 4, 8 (new GPU infra, HLS/DASH packaging, video CDN) | +$0.15–$0.25/video (new cost category) | 3–6 months |
| "Add AI tagging" | Stage 5 (new stage) | +$0.003–$0.01/asset (LLM API cost) | 4–8 weeks |
| "Add 5 new languages" | Stage 6 (translation volume) | +$0.005/asset × 5 locales = +$0.025/asset | 2–4 weeks + dictionary curation |
| "Add DRM watermarking" | Stage 7 (new stage), Stage 8 (separate watermarked variants) | +$0.001/asset + storage for watermarked variants | 4–6 weeks |
| "Reduce time-to-visible from 15 min to 30 sec" | All stages — must parallelize, optimize, or skip stages | Significant engineering effort, may require faster (more expensive) AI models | 2–4 months |

---

## 16. Observability & Operational Readiness

### 16.1 The Four Pillars

| Pillar | What to Measure | Tools (Open Source) |
|---|---|---|
| **Metrics** | Queue depth per stage, processing latency per stage, error rate per stage, asset throughput, cost per asset | Prometheus / Mimir + Grafana |
| **Logs** | Structured JSON logs with `asset_id` as correlation key, error details, stage transitions | Loki / Elasticsearch |
| **Traces** | End-to-end distributed trace per asset across all stages | OpenTelemetry + Tempo / Jaeger |
| **Profiling** | CPU/memory profiling of processing workers (identify hot spots) | Pyroscope / pprof |

### 16.2 Key Dashboards

Every ingestion pipeline should have these dashboards:

| Dashboard | Audience | Key Panels |
|---|---|---|
| **Pipeline Health** | On-call engineer | Queue depth per stage (real-time), error rate per stage, DLQ depth, active worker count |
| **SLO Tracker** | Engineering leadership | Time-to-ready (p50/p95/p99), error budget burn rate, availability |
| **Cost Tracker** | Engineering + Finance | Daily/weekly cost per stage, cost per asset trend, LLM token usage, GPU utilization |
| **Producer Experience** | Product team | Upload success rate, time-to-visible distribution, error message frequency |

### 16.3 SLO Recommendations

| SLO | Target | Measurement |
|---|---|---|
| **Pipeline availability** | 99.95% | Percentage of time the upload endpoint returns 2xx for valid requests |
| **Time-to-ready (stills)** | < 15 min (p95) | Time from upload complete to `READY` status |
| **Time-to-ready (video)** | < 45 min (p95) | Time from upload complete to `READY` status |
| **Processing success rate** | > 99.5% | Percentage of uploaded assets that reach `READY` status |
| **Data durability** | 99.999999999% (11 nines) | No original ever lost (object storage guarantee) |

### 16.4 Alerting Philosophy

**Alert on SLO burn rate, not raw metrics.**

A spike in error rate that lasts 30 seconds and self-resolves is noise. A sustained elevation that burns error budget 10× faster than sustainable is a real problem.

Use multi-window, multi-burn-rate alerting:
- **Page (wake someone up):** Burning > 14.4× budget in a 1-hour window AND > 6× in a 6-hour window.
- **Ticket (next business day):** Burning > 3× budget in a 1-day window AND > 1× in a 3-day window.

---

## 17. Team Topology & Parallel Development

### 17.1 Enabling Parallel Development

A well-designed pipeline allows teams to develop stages independently. The keys:

1. **Contract-first API design:** Define the event schema between stages before implementation. Teams develop against the contract, not against each other's code.
2. **Mock servers:** Each stage's input can be simulated with a mock event producer. Teams don't need to wait for upstream stages to be complete.
3. **Schema registry:** Event schemas are versioned and stored in a shared registry (Git-based is sufficient). Breaking changes are caught in CI.

### 17.2 Recommended Team Boundaries

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TEAM OWNERSHIP MAP                                    │
│                                                                         │
│  PLATFORM / UPLOAD TEAM          MEDIA PROCESSING TEAM                  │
│  ───────────────────────        ──────────────────────                  │
│  Stage 1: Upload initiation     Stage 4: Variant generation (images)   │
│  Stage 2: Integrity verification Stage 4: Video transcoding            │
│  Stage 3: Metadata validation   Stage 8: CDN propagation               │
│                                                                         │
│  AI/ML TEAM                     TRUST & SAFETY TEAM                    │
│  ─────────                      ───────────────────                    │
│  Stage 5: AI enrichment         Stage 5: Content moderation            │
│  Stage 6: Localization (LLM)    Stage 7: Rights & compliance           │
│  Vector search infrastructure   DMCA/takedown process                  │
│                                                                         │
│  SEARCH / DISCOVERY TEAM        INFRASTRUCTURE / PLATFORM TEAM         │
│  ────────────────────────       ──────────────────────────────         │
│  Stage 9: Indexing              Message broker operations               │
│  Feed injection                 Object storage management               │
│  Recommendation engine          Observability stack                     │
│                                 CI/CD for all teams                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 17.3 Contract Testing in CI

```
  ┌─────────────────┐          ┌─────────────────┐
  │  Event Schema   │          │  API Spec        │
  │  (Protobuf or   │          │  (OpenAPI 3.1)   │
  │   JSON Schema)  │          │                  │
  └────────┬────────┘          └────────┬─────────┘
           │                            │
  ┌────────▼────────┐          ┌────────▼─────────┐
  │  Schema Linter  │          │  API Linter      │
  │  (breaking      │          │  (Spectral,      │
  │   change detect)│          │   oasdiff)       │
  └────────┬────────┘          └────────┬─────────┘
           │                            │
  ┌────────▼──────────────────────────--▼─────────┐
  │           CI PIPELINE                          │
  │  - Validate schemas                            │
  │  - Generate mock producers/consumers           │
  │  - Run contract tests                          │
  │  - Detect breaking changes → block PR          │
  └────────────────────────────────────────────────┘
```

---

# Appendices

## Appendix A: Platform Archetype Mapping

How each pipeline stage maps to common platform types. Use this to determine which stages are relevant to your platform.

| Stage | Pinterest-like | YouTube-like | E-Commerce | Digital Art / Gallery | Enterprise DAM | News / Media |
|---|---|---|---|---|---|---|
| 1. Upload Initiation | Full | Full | Full | Full | Full + approval workflow | Full |
| 2. Integrity | Full | Full | Full | Full | Full + virus scan | Full |
| 3. Metadata | Medium (light schema) | Medium | Full (SKU, pricing) | Full (provenance, attribution) | Full (taxonomy) | Full (byline, dateline) |
| 4. Image Processing | Full | Thumbnails only | Full + background removal | Full + color management | Full | Full + crop suggestions |
| 4. Video Processing | Short clips only | **Full (core value)** | Product video | Full | Optional | Full |
| 5. AI Tagging | Full | Full | Full (product classification) | Full (style, mood, era) | Optional | Full (entities, topics) |
| 5. Content Moderation | **Critical** | **Critical** | Required | Required | Optional | Required |
| 5. Embeddings | Full | Full | Optional | Full | Optional | Optional |
| 6. Localization | Optional | Full (100+ languages) | Full (markets) | Full | Minimal | Full |
| 7. Rights/DRM | Minimal (DMCA) | Full (Content ID) | Minimal | **Full (core value)** | Full (usage policies) | Full (licensing) |
| 8. CDN | Full | **Full (core value)** | Full | Full | Regional | Full |
| 9. Indexing | Full | Full | Full | Full | Full | Full + time-sensitive |

## Appendix B: Glossary

| Term | Definition |
|---|---|
| **ABR (Adaptive Bitrate)** | Streaming technique where the player switches between quality levels based on available bandwidth. |
| **AVIF** | AV1 Image File Format. Modern image codec offering ~30-50% smaller files than JPEG at equivalent quality. |
| **BCP 47** | IETF standard for language tags (e.g., `en-US`, `ja`, `zh-Hans`). |
| **BlurHash** | A compact string representation of a blurred placeholder image, used during progressive loading. |
| **CDN (Content Delivery Network)** | A distributed network of servers that cache content at edge locations close to consumers. |
| **CLIP** | Contrastive Language-Image Pre-training. A model that creates embeddings in a shared space for both images and text, enabling cross-modal search. |
| **CQRS** | Command Query Responsibility Segregation. Pattern separating write operations from read operations, allowing independent optimization. |
| **DASH** | Dynamic Adaptive Streaming over HTTP. An adaptive bitrate streaming protocol (alternative to HLS). |
| **DLQ (Dead Letter Queue)** | A queue where messages that cannot be processed after exhausting retries are stored for investigation. |
| **DRM (Digital Rights Management)** | Technologies that control access to copyrighted digital media. |
| **DWT** | Discrete Wavelet Transform. Used in invisible watermarking to embed data in the frequency domain of an image. |
| **EXIF** | Exchangeable Image File Format. Metadata embedded in image files (camera settings, GPS, orientation). |
| **FFmpeg / FFprobe** | Open-source multimedia framework for encoding, decoding, and inspecting media files. |
| **GIN Index** | Generalized Inverted Index (PostgreSQL). Efficient for indexing array columns and full-text search. |
| **HLS** | HTTP Live Streaming. Apple's adaptive bitrate streaming protocol. The de facto standard for video delivery. |
| **ICC Profile** | International Color Consortium profile. Defines how colors in an image should be mapped to display colors. |
| **Idempotent** | An operation that produces the same result regardless of how many times it is executed. |
| **Lanczos** | A high-quality image resampling algorithm used for resizing. |
| **libvips** | A fast, memory-efficient image processing library that operates in streaming mode. |
| **LUFS** | Loudness Units Full Scale. The standard for measuring audio loudness (EBU R128 targets -14 LUFS). |
| **NVENC** | NVIDIA's hardware video encoder, significantly faster than software encoding for H.264/H.265. |
| **Pre-signed URL** | A URL with embedded authentication credentials that grants time-limited access to a specific object in storage. |
| **Sidecar file** | A metadata file (CSV, JSON, XML) uploaded alongside media files, containing descriptive metadata for each asset. |
| **Smart Crop** | Cropping technique that uses saliency detection to identify the most visually important region of an image, rather than center-cropping. |
| **Tone Mapping** | The process of converting HDR content to SDR for display on standard monitors. |
| **UUIDv7** | A UUID variant that embeds a timestamp, making IDs time-sortable while remaining globally unique. |
| **Vector Database** | A database optimized for storing and querying high-dimensional vectors (embeddings) using approximate nearest-neighbor search. |
| **WebP** | Google's image format offering better compression than JPEG with broad browser support. |

---

> **Document History**
> | Version | Date | Author | Changes |
> |---|---|---|---|
> | 1.0 | 2026-03-10 | Engineering Architecture | Initial release |
