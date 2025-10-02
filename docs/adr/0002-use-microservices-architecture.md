# 0002 – Use Microservices Architecture for Scalability and Flexibility
*Status:* Accepted  
*Deciders:* Umashankar Ankuri, Abhinandan Bansal  
*Date:* 2025-09-28  

## Context
We must scale hot paths (orders/payments/tracking), deploy frequently, and isolate failures. A monolith slows independent scaling and releases.

## Decision Drivers
- Scalability of hot domains
- Faster, safer deployments
- Fault isolation & team autonomy
- Clear domain boundaries (Catalog, Order, Payment, Dispatch, Tracking, Restaurant Ops, Foundation)

## Considered Options
- Monolithic Architecture  
- **Microservices Architecture** (chosen)  
- Serverless-first

## Decision
Adopt **microservices** with domain-aligned services, REST/gRPC for service APIs, and **event-driven** integration for state changes.

## Consequences
- Pros: Independent scaling/deploys, smaller blast radius, tech flexibility.  
- Cons: Distributed complexity (timeouts, retries), eventual consistency, higher observability needs.

## Risks & Mitigations
- Cross-service failures → service mesh (retries with backoff, circuit-breakers); sane timeouts; bulkheads.  
- Data consistency → **Sagas + outbox**; idempotent consumers; per-order partitioning.

## Scope
MVP starts with 5–8 services (Foundation services, Catalog, Order, Payment, Dispatch, Tracking, Restaurant API). Split further as scale grows.

## Links
