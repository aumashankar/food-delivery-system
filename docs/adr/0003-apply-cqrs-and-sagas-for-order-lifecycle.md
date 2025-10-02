# 0003 – Apply CQRS and Sagas for Order Lifecycle
*Status:* Review  
*Deciders:* Umashankar
*Date:* 2025-09-30  

## Context
Order flows span multiple services (Order, Payment, Restaurant, Dispatch, Tracking). We need clear write/read separation, an audit trail, and reliable cross-service coordination.

## Decision Drivers
- High write throughput (1k→3.7k orders/sec peak in 5 yrs)
- Auditability & replay ability of order states
- Loose coupling across services
- Resilience to partial failures

## Considered Options
1) Single DB + synchronous calls  
2) Event sourcing without read/write separation  
3) **CQRS with Sagas + Outbox** (chosen)

## Decision
- Use **CQRS** for the **Order** domain:  
  - **Write model** publishes immutable events (e.g., `orders.created`, `orders.accepted`, `orders.delivered`). 
  - **Read models** are denormalized: quick queries for apps/APIs (e.g., Redis/ElasticSearch/SQL projections).  
- Use **Sagas** to orchestrate cross-service transitions (payment auth → order confirm → dispatch assign → delivery → capture).  
- Use the **Outbox Pattern** to persist events with the same transaction as state changes; a relay publishes to Kafka.

## Consequences
- Pros: Scalability, audit log, clear state transitions, flexible read models.  
- Cons: Complexity in maintaining projections; eventual consistency; need idempotency everywhere.



