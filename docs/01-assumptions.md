# Assumptions

## High level design tries to cover the major parts of problem

## Traffic & Scale
- **Year 1 peak:** 1,000 orders/sec. Average ≈ peak/3 due to lunch/dinner peaks.
- **Growth:** 30% YoY for 5 years → **~3,713 orders/sec** at Year 5 peak.
- **Event fanout:** ~6–10 internal events/order → Yr5 ≈ 25k–40k events/sec.
- **Estimations:** [Estimations Google Sheet](https://docs.google.com/spreadsheets/d/1eTeTL9LQVQokM4e1JBeC9hl1yfHHnq68qMI9jJmtFR4/edit?usp=sharing) 

## Availability & Latency
- **Availability SLO:** 99.8% (≈ 86.4 min downtime/month).
- **Latency SLO:** p95 page load ≤ 2s; API p95 ≤ 300–500ms for backend api.

## Geography & Rollout
- Launch in 1–2 metros; multi-city within 12–18 months.
- Single cloud region with **3 Availability Zones**; DR in secondary region.

## Payments & Compliance
- Payment gateway integration (Indian context) (Razorpay/ Cashfree /Paytm); US (Paypal/Stripe) EU (Mollie) etc.,
- DPDP (India) compliant PII handling; audit trails. PCI

## Database
- Initially start with Postgres and introduce Cassandra (Dispatch tracking,driver/location pings) in future for more Horizontal scaling

## Mobile
- React Native apps (Customer & Driver); push notifications; 
- real-time map tracking; driver pings every 2–5s.

## DevOps
- Rollout deployment; infra as code (Terraform); 24×7 support aligned to SLOs

## User Journeys
- Touched upon few user journeys

## Tech stack
- Tech stack suggestions are generic, not fully dependent on one particular cloud provider. As choosing cloud provider itself a decision to made which depends on many factors cost/pricing model, ecosystem,team skill set etc.,