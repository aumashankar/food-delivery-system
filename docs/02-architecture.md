# Architecture Diagrams & Flows

## Context
```mermaid
flowchart LR
%% Clients & Edge
    C[Customer App] --- W[Web] --- D[Driver App] --- R[Restaurant Console]
    C & W & D & R --> BFF[Backend-for-Frontend]
    BFF --> GW[API Gateway/WAF]
    GW --> AUTH[Auth Service OIDC/JWT]
AUTH <--> IDP[[Identity Provider]]

%% Core services (behind gateway)
GW --> ORD[Orders Event-Sourced]
GW --> PAY[Payment Svc]
GW --> DSP[Dispatch Svc]
GW --> TRK[Tracking Svc]
GW --> CAT[Catalog Svc]
GW --> REV[Reviews Svc]

%% Event backbone
K[(Kafka)]
%% Domain events (intent)
ORD -- domain events --> K
PAY -- events --> K
DSP -- events --> K
TRK -- events --> K
%% CDC streams (state change)
CAT -- CDC --> K
REV -- CDC --> K

%% Projections / read models
K --> PROJ_ORD[Orders Projectors] --> ORD_PROJ[(Orders Read Model)]
K --> PROJ_CAT[Catalog Projector] --> OSE[(OpenSearch)]
K --> PROJ_REV[Review Aggregator] --> REV_PROJ[(Review Read Model)]
K --> PROJ_TRK[Tracking Projector] --> TRK_PROJ[(Tracking Read Model)]

%% Low-latency stores
DSP --> REDIS[(Redis + RedisGeo)]
TRK --> REDIS

```
### Authentication - Event Sourcing - CDC
- Authentication: Clients hit BFF → API Gateway, which validates tokens with Auth Service (OIDC/JWT) backed by your IdP; downstream services trust JWTs. 
- Event sourcing: Only Orders is event-sourced (commands → domain events). Catalog & Reviews are CRUD with CDC to Kafka; other services publish events but are not event-sourced.
- Payments: PSPs are the source of truth; we need a deterministic ledger & reconciliation over complex event streams. A ledger + idempotent state machine + CDC/outbox gives auditability and PCI-friendly control without ES overhead. 
- Dispatch/Tracking: Ultra-low-latency, geo-indexed ephemeral state (Redis/RedisGeo) — event sourcing adds write/rehydration latency with little benefit; we still emit events for history/analytics. 
- Catalog/Reviews: Mostly CRUD with heavy reads; Postgres + read replicas + CDC → Kafka keeps it simple and scalable, while ES would complicate write paths without clear ROI.

## Authentication - RBAC

```mermaid
flowchart LR
  %% Clients
  C[Customer App] --- W[Web] --- D[Driver App] --- R[Restaurant Console]

  %% Edge & Auth
  C & W & D & R --> EDGE[Edge / API Gateway]
  EDGE --> WAF[WAF & Rate Limiter]
  WAF --> AUTH[Auth Service - OIDC Client]
  AUTH <--> IDP[[Identity Provider - OIDC]]

  %% Identity & Role Management
  AUTH <--> UMS[User Manager Service - profiles, roles, tenants]
  UMS --> RBACMAP[Role Mapping - customer/restaurant/driver]

  %% OIDC login
  IDP -->|OIDC Code + PKCE| AUTH
  AUTH -->|Validate + Exchange| IDP
  AUTH -->|OIDC ID/Refresh Tokens| EDGE

  %% Token conversion for downstream services
  EDGE --> CONV[OIDC --> JWT Token Converter]
  CONV -->|Mint short-lived Internal JWT| EDGE
  CONV --> JWKS[(JWKS / Key Mgmt)]
  CONV --> AUD[Audit & Access Logs]

  %% Centralized authorization (PDP)
  EDGE --> ACS[Access Control Service - PDP/OPA]
  RBACMAP --> ACS
  UMS --> ACS
  ACS --> POLDEC[Permit / Deny decision]

  %% Routing to services (JWT attached)
  EDGE --> ORD[Orders Svc]
  EDGE --> PAY[Payments Svc]
  EDGE --> CAT[Catalog Svc]
  EDGE --> DSP[Dispatch Svc]
  EDGE --> TRK[Tracking Svc]
  EDGE --> REV[Reviews Svc]

  %% Services validate JWT locally
  JWKS --> ORD
  JWKS --> PAY
  JWKS --> CAT
  JWKS --> DSP
  JWKS --> TRK
  JWKS --> REV

  %% Extras
  AUTH --> MFA[(MFA / Device Binding)]
  AUTH --> REVOC[(Revocation / Introspection)]
  EDGE --> TKNCACHE[(Token Cache)]

```
### User Manager and Access Control

- User Manager Service (UMS): source of truth for profiles, roles, tenants/cities, restaurant memberships; feeds role mapping and attributes (e.g., restaurant_id, driver_id) used in ABAC. 
- Access Control Service (ACS): policy decision point (PDP) using OPA/rego or similar; evaluates RBAC + ABAC with inputs from UMS and request context; Edge asks ACS for permit/deny before routing.

##  Login Flow - (login → authorize → route)
```mermaid
sequenceDiagram
  participant App as Client/App
  participant Edge as Edge/API GW
  participant Auth as Auth Svc (OIDC)
  participant IdP as IdP
  participant UMS as User Manager
  participant Conv as OIDC→JWT Converter
  participant ACS as Access Control (PDP)
  participant ORD as Orders Svc

  App->>Edge: /orders (no token)
  Edge->>Auth: Start OIDC (code+PKCE)
  Auth->>IdP: Exchange code
  IdP-->>Auth: ID/Access/Refresh tokens
  Auth->>UMS: Get roles/attributes
  UMS-->>Auth: role=customer, tenant=city_12
  Edge->>Conv: Mint internal JWT (embed roles/attrs)
  Conv-->>Edge: JWT(short-lived, aud=orders)
  Edge->>ACS: is_allowed(user, action="read:order", attrs)
  ACS-->>Edge: PERMIT
  Edge->>ORD: GET /orders/{id} (Authorization: Bearer <JWT>)
  ORD-->>Edge: 200 OK
  Edge-->>App: 200 OK

```
### JWT claims
- sub, exp, role, scopes, tenant, city_id, customer_id|restaurant_id|driver_id (as applicable), app_version.

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

## User Journeys

```mermaid
sequenceDiagram
  title 1) Customer Browse → Order → Pay (Happy Path)
  actor C as Customer App/Web
  participant BFF as BFF / Edge
  participant DISC as Discovery Svc (OpenSearch)
  participant CAT as Catalog Svc (RO)
  participant CART as Cart Svc
  participant QUOTE as Quote/Pricing Svc
  participant PAY as Payments Svc / PG
  participant ORD as Orders Svc (CQRS)
  participant KAFKA as Event Bus (Kafka)
  participant NOTIF as Notification Svc

  C->>BFF: Detect location & request restaurants
  BFF->>DISC: Search(H3, facets, sort)
  DISC-->>BFF: Search results
  BFF-->>C: Restaurant list

  C->>BFF: Open restaurant → get menu
  BFF->>CAT: Fetch menu+prices
  CAT-->>BFF: Menu
  BFF-->>C: Render menu

  C->>BFF: Add items / Update cart
  BFF->>CART: Upsert(cartId, items, idemKey)
  CART-->>BFF: Cart snapshot
  BFF-->>C: Show cart

  C->>BFF: Checkout → quote & fees
  BFF->>QUOTE: Compute(ETA, delivery fee, surge, coupon)
  QUOTE-->>BFF: Quote result
  BFF-->>C: Show payable

  C->>BFF: Pay now
  BFF->>PAY: Create PaymentIntent (PENDING_PAYMENT)
  PAY-->>C: PG redirect/SDK
  C-->>PAY: Complete auth
  PAY-->>BFF: PaymentAuthorized

  BFF->>ORD: Create Order(PLACED)
  ORD->>KAFKA: OrderPlaced
  KAFKA->>NOTIF: Enqueue confirmation
  NOTIF-->>C: Order confirmed (ETA)

```

```mermaid
sequenceDiagram
  title 2) Restaurant Accept → Prepare → Handoff
  participant ORD as Orders Svc
  participant KAFKA as Event Bus (Kafka)
  actor RC as Restaurant Console
  participant PAY as Payments Svc
  actor DE as Driver App
  participant NOTIF as Notification Svc

  ORD->>KAFKA: OrderPlaced
  KAFKA-->>RC: New order toast
  RC->>ORD: ACCEPT order
  ORD->>PAY: Capture (per policy)
  PAY-->>ORD: Captured
  ORD->>KAFKA: OrderAcceptedByRestaurant
  KAFKA->>NOTIF: Notify customer
  NOTIF-->>Customer: "Restaurant accepted"

  RC->>ORD: READY for pickup
  ORD->>KAFKA: OrderReady
  KAFKA-->>DE: Pickup request

  DE->>ORD: PICKED_UP (QR/OTP)
  ORD->>KAFKA: OrderPickedUp
  KAFKA->>NOTIF: Notify customer
  NOTIF-->>Customer: "Picked up, on the way"

```

```mermaid
sequenceDiagram
  title 3) Delivery Exec Online → Assign → Deliver → Earnings
  actor DE as Driver App
  participant DISP as Dispatch/Assignment Svc
  participant ORD as Orders Svc
  participant MAP as Maps/ETA Svc
  participant KAFKA as Event Bus (Kafka)
  participant EARN as Earnings/Payouts Svc
  participant NOTIF as Notification Svc

  DE->>DISP: Go Online (status+geo)
  loop Location pings (3–5s)
    DE->>DISP: GPS ping(H3)
  end

  DISP->>ORD: Find order awaiting assignment
  DISP->>MAP: ETA calc (DE↔Restaurant↔Customer)
  MAP-->>DISP: ETA & route score
  DISP-->>DE: Assignment offer
  DE->>DISP: Accept

  DISP->>ORD: ASSIGNED (driverId)
  ORD->>KAFKA: OrderAssigned
  KAFKA->>NOTIF: Notify customer (driver & ETA)

  DE->>ORD: Arrived at restaurant
  DE->>ORD: PICKED_UP (OTP/QR)
  ORD->>KAFKA: OrderPickedUp

  loop En-route updates
    DE->>DISP: GPS ping
    DISP->>MAP: Recompute ETA (optional)
    MAP-->>DISP: ETA
    DISP->>ORD: Update status/ETA
  end

  DE->>ORD: DELIVERED (OTP/photo)
  ORD->>KAFKA: OrderDelivered
  KAFKA->>EARN: Compute incentive
  EARN-->>DE: Earnings updated
  KAFKA->>NOTIF: Ask for rating

```

```mermaid
sequenceDiagram
  title 4) Cancel & Refund (Customer-initiated)
  actor C as Customer App/Web
  participant BFF as BFF / Edge
  participant ORD as Orders Svc
  participant POLICY as Policy/Fees Engine
  participant PAY as Payments Svc
  participant KAFKA as Event Bus (Kafka)
  participant NOTIF as Notification Svc

  C->>BFF: Cancel order
  BFF->>ORD: Cancel request(orderId, reason)
  ORD->>POLICY: Evaluate(cancel window, fee, refund)
  POLICY-->>ORD: Decision(free/partial/deny)

  alt Eligible (pre-prep or policy allows)
    ORD->>PAY: Initiate refund (full/partial)
    PAY-->>ORD: RefundInitiated
    ORD->>KAFKA: OrderCancelled / RefundInitiated
    KAFKA->>NOTIF: Notify customer
    NOTIF-->>C: Cancellation & refund details
  else Not eligible (picked up/delivered)
    ORD-->>BFF: Deny (policy message)
    BFF-->>C: Show reason + contact support
  end

  PAY-->>ORD: RefundSettled (async T+X)
  ORD->>KAFKA: RefundSettled
  KAFKA->>NOTIF: Notify customer

```