# Assumptions

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
- Single cloud region with **3 AZs**; DR in secondary region.

## Payments
- Payment gateway integration (Indian context) (Razorpay/ Cashfree /Paytm); US (Paypal/Stripe) EU (Mollie) etc.,

## Compliance
- DPDP (India) compliant PII handling; audit trails. PCI 

## Mobile
- Native or RN apps; push notifications; real-time map tracking.

## DevOps
- Rollout deployment; infra as code; 24×7 support aligned to SLOs