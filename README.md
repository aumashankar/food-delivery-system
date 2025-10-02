# Food Delivery Platform — Scalable & Efficient Architecture

**Date:** Sep 27, 2025

A cloud-native, event-driven platform for food delivery with geo-aware dispatch, real-time tracking, and resilient payment flows. Designed to meet **99.8% availability**, **≤2s end-user page load**, and scale from **1,000 orders/sec** (Year 1 peak) to **~3,700 orders/sec** by Year 5 (30% CAGR).

## Quick Links
- [Assumptions](docs/01-assumptions.md)
- [Architecture Diagrams](docs/02-architecture.md)
- [Tech Stack & Frameworks](docs/03-tech-stack.md)
- [Milestone Plan](docs/04-milestones.md)
- [Staffing Plan](docs/05-staffing.md)
- [Architecture Decision Records](adr/0001-record-architecture-decisions.md)
---

## Executive Snapshot

- **Throughput:** Start 1,000 orders/sec (peak); 30% YoY → ~3,713 orders/sec (Yr5).
- **Latency SLO:** p95 page load ≤ 2s; API p95 ≤ 300–500ms on core apis.
- **Availability SLO:** 99.8% (Downtime ≈ 86.4 min/month).
- **Architecture:** Microservices, CQRS (for orders), Kafka streaming, Redis (& RedisGeo), Postgres OLTP, OpenSearch for discovery, ClickHouse/BigQuery for analytics, WebSockets for live tracking.
- **Reliability:** Multi‑Availability Zones, rollout/canary deploys, circuit breakers, retries with backoff, DR in secondary region.
- **Security:** OIDC/JWT, TLS everywhere, tokenized payments, DPDP (India) & PCI

### Context Diagram

```mermaid

flowchart LR
    C[Customer App] --- W[Web] --- D[Driver App] --- R[Restaurant Console]
    C & W & D & R --> BFF[Backend-for-Frontend]
    BFF --> GW[API Gateway/WAF]

    GW --> ORD[Order Commands Event-Sourced]
GW --> PAY[Payment Svc]
GW --> DSP[Dispatch Svc]
GW --> TRK[Tracking Svc]
GW --> CAT[Catalog Svc]
GW --> REV[Review Svc]

K[(Kafka)]
ORD -- domain events --> K
PAY -- events --> K
DSP -- events --> K
TRK -- events --> K
CAT -- CDC --> K
REV -- CDC --> K

PROJ_ORD[Orders Projectors] --> ORD_PROJ[(Orders Read Model)]
PROJ_CAT[Catalog Projector] --> OSE[(OpenSearch)]
PROJ_REV[Review Aggregator] --> REV_PROJ[(Review Read Model)]
PROJ_TRK[Tracking Projector] --> TRK_PROJ[(Tracking Read Model)]

K --> PROJ_ORD
K --> PROJ_CAT
K --> PROJ_REV
K --> PROJ_TRK

DSP --> REDIS[(Redis + RedisGeo)]
TRK --> REDIS


```

### Milestones (Gantt)

---