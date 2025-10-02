# High-Level Technology Stack & Frameworks

## Client
- **Web:** Next.js (React framework) HTTP/3, code-splitting, image optimization (webp)
- **Mobile:** React Native (TypeScript); push notifications FCM/APNs;

## Edge & API
- BFF: Node.js - (REST/GraphQL); gRPC/REST to services, 
- Gateway/WAF: API Gateway, WAF, rate limiting, auth offload, CDN.

## Identity (IAM)
- OAuth2.1/OIDC (JWT) Keycloak or (Auth0 or AWS Cognito); Role Based Access Control

## Services (microservices)
- **Languages:** Java (Spring Boot, Spring Reactive Web - Flux,Mono)  / Node.js (BFF). / React JS
- **Inter-service:** REST (external). Optional GraphQL at BFF. Microservices with Sagas

## Messaging
- Kafka + Schema Registry; DLQ; consumer lag SLOs; stream processing (Kafka Streams/Flink).
- **Streaming/Event Bus:** Kafka; Event sourcing (CQRS)

## Data Engineering
- **Relational DB:** Postgres/Aurora(If AWS) for users, restaurants, catalog, payouts, etc.
- **Cache:** Redis Cluster + **RedisGeo** for proximity; sessions, carts, hot menus, rate limits.
- **Search:** OpenSearch/Elasticsearch for restaurant/menu discovery & geo-distance sorts.
- **Analytics:** ClickHouse/BigQuery for near real-time metrics; Spark/Flink for stream processing.
- **Object Store:** S3 (images, invoices, exports) or similar storage based on cloud provider.

## Machine Learning - Engineering
- Databricks if budget allows instead of juggling with set of opensource libraries for maintenance
- Python for Machine Learning (sklearn,stats models) & Deep Learning (Pytorch)

## Geo & Routing
- **Providers:** [OSRM](https://project-osrm.org/)/[GraphHopper](https://www.graphhopper.com/) + Google/Mapbox APIs; cached segments; ETA service.

## Payments & Notifications
- **Payments:** Razorpay/Stripe/Paytm with tokenization;
- **Notifications:** FCM/APNs/Email (SES/SNS) or Twillio/SendGrid; optional WhatsApp via API

## Platform & Ops
- **Kubernetes** Docker containers **Istio/Linkerd** Service Mesh (retries with backoff, circuit breaking), **ArgoCD** (GitOps).
- **CI/CD:** GitHub Actions; rollout/canary; feature flags.
- **Observability:** OpenTelemetry + Prometheus/Grafana/Loki;SLOs & alerts/dashboards; PagerDuty. Datadog as central platform (if client budget permits)
- **Security:** Secrets manager, encryption, DPDP compliance, PCI
