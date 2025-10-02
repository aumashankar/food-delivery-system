# 0004 – Choose Kafka for Event Streaming and Integration
*Status:* Accepted  
*Deciders:* Umashankar 
*Date:* 2025-09-30  

## Context
Services need to communicate state changes reliably at high throughput (order events, tracking updates, dispatch assignments). We also need replay for projections/analytics.

## Decision Drivers
- Throughput (30–40k internal events/sec at Yr5)
- Durable log + replay for projections and analytics
- Partitioning by order/city for locality & ordering
- Mature ecosystem (connectors, schema registry)

## Considered Options
- RabbitMQ/ActiveMQ (broker-first)  
- Cloud pub/sub (managed, simpler, fewer guarantees)  
- **Apache Kafka / managed Kafka** (chosen)

## Decision
Use **Kafka** (or managed equivalent) with:
- **Topics** (examples):  
  - `orders.created`, `orders.accepted`, `orders.ready_for_pickup`, `orders.delivered`  
  - `payments.intent.created`, `payments.authorized`, `payments.captured`, `payments.refunded`  
  - `logistics.driver.assigned`, `tracking.location.update`  
- **Partition keys:** primarily `order_id`; also consider `city_id` for high-fanout topics.
- **Retention:** raw streams 7–14 days; compacted state topics for “latest” snapshots.  
- **Outbox** for reliable publication; **dead-letter topics** for exception messages.

## Consequences
- Pros: High throughput, durability, replayability, strong ecosystem.  
- Cons: Operational overhead (capacity, monitoring), consumer complexity.

## Risks & Mitigations
- Hot partitions (popular restaurants/regions) → key hashing with salt, sharded counters.  

