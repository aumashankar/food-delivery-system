# Architecture Diagrams & Flows

## Context
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

##  Data Ownership & Read Replicas

```mermaid
flowchart LR
  ORD[(Orders Events in Kafka)]
  SNAP[(Orders Snapshot DB)]:::db
  ORD --> SNAP
  ORD --> ORD_PROJ[(Orders Read Model)]:::db

  PAYDB[(Postgres: Payments Primary)]:::db --> PAYRR[(Read Replica)]:::rr
  CATDB[(Postgres: Catalog Primary)]:::db --> CATR[(Read Replica)]:::rr
  REVDB[(Postgres: Reviews Primary)]:::db --> REVRR[(Read Replica)]:::rr

  K[(Kafka)] --> OSE[(OpenSearch-Catalog Search)]:::search
  K --> REV_PROJ[(Review Cache/Projection)]:::db

  REDIS[(Redis + RedisGeo)]:::cache

  classDef db stroke:#aa0;
  classDef rr stroke:#aa0,stroke-dasharray: 3 3;
  classDef search stroke:#36c;
  classDef cache stroke:#999;
```

## Order State
```mermaid
stateDiagram-v2
    [*] --> CREATED
    CREATED --> PENDING_PAYMENT: PaymentIntentCreated
    PENDING_PAYMENT --> CONFIRMED: PaymentAuthorized|PaymentReceived
    CREATED --> CONFIRMED: CashOnDelivery
    CONFIRMED --> ACCEPTED: OrderAcceptedByRestaurant
    CONFIRMED --> REJECTED: OrderRejectedByRestaurant
    ACCEPTED --> PREPARING: FoodPreparing
    PREPARING --> READY_FOR_PICKUP: OrderReadyForPickup
    READY_FOR_PICKUP --> ASSIGNED: DriverAssigned
    ASSIGNED --> EN_ROUTE_TO_RESTAURANT: DriverEnRouteToRestaurant
    EN_ROUTE_TO_RESTAURANT --> AT_RESTAURANT: DriverArrivedAtRestaurant
    AT_RESTAURANT --> PICKED_UP: OrderPickedUp
    PICKED_UP --> EN_ROUTE_TO_CUSTOMER: DriverEnRouteToCustomer
    EN_ROUTE_TO_CUSTOMER --> AT_CUSTOMER: DriverArrivedAtCustomer
    AT_CUSTOMER --> DELIVERED: OrderDelivered
    DELIVERED --> SETTLED: PaymentCaptured
    SETTLED --> [*]
    CREATED --> CANCELLED
    PENDING_PAYMENT --> CANCELLED
    CONFIRMED --> CANCELLED
    ACCEPTED --> CANCELLED
```

## Order Flow - Order CQRS (Event Sourcing)

```mermaid
sequenceDiagram
  participant UI as BFF/Client
  participant CMD as Orders Command API
  participant K as Kafka (events)
  participant WM as Orders Write Model (State/Snapshot DB)
  participant PROJ as Orders Projector
  participant RDB as Orders Read Model

  UI->>CMD: Create/Accept/Deliver (command + idempotency key)
  CMD->>K: Append OrderCreated/Accepted/Delivered
  K-->>WM: Apply events (update snapshot)
  K-->>PROJ: Orders events stream
  PROJ->>RDB: Upsert denormalized views
  UI->>RDB: GET /orders/{id}
  RDB-->>UI: Current order state
```

## Catalog/Reviews CRUD + CDC

```mermaid
flowchart LR
  API[Catalog API] --> CATDB[(Postgres: Catalog Primary)]
  CATDB --> CATR[(Read Replica)]
  CDC_CAT[CDC Connector] --> K[(Kafka)]
  CATDB -- WAL --> CDC_CAT
  K --> CATPROJ[Catalog - OpenSearch Projector]
  CATPROJ --> OSE[(OpenSearch Index)]

  RAPI[Review API] --> RDB[(Postgres: Reviews Primary)]
  RDB --> RREP[(Read Replica)]
  CDC_REV[CDC Connector] --> K
  RDB -- WAL --> CDC_REV
  K --> RAGG[Review Aggregator]
  RAGG --> REV_PROJ[(Review Read Model Cache)]
```

## Dispatch Flow (low-latency)

```mermaid
flowchart TD
  A[OrderReadyForAssignment] --> B{Nearby drivers in cells?}
  B -- no --> C[Expand radius / neighbor cells]
  C --> B
  B -- yes --> D[Fetch candidates from RedisGeo k-nearest]
  D --> E[Score: ETA, distance, load, SLA, batching, surge]
  E --> F[Soft-assign top driver]
  F --> G{Driver responds within T?}
  G -- no --> H[Timeout - > requeue - > widen radius]
  H --> B
  G -- yes --> I[Confirm assignment]
  I --> J[Notify driver, restaurant, customer]
  J --> K[Update ETA & route start tracking]
```


## Tracking (pings → WebSocket)

```mermaid
sequenceDiagram
  participant D as Driver App
  participant TRK as Tracking Svc
  participant RG as RedisGeo
  participant K as Kafka
  participant C as Customer App

  D->>TRK: Location ping (2–5s)
  TRK->>RG: Update driver geo position
  TRK->>K: Emit tracking.update
  C-->>TRK: WebSocket map/ETA
```