# Service Level Objective (SLO) & Error Budget Policy

**Service**: Stan's Robot Shop (Core Services: Web, Cart, Catalogue, Payment)  
**Document Version**: 1.0  
**Owner**: DevOps & Site Reliability Engineering  
**Review Cycle**: Monthly  

---

## 1. Service Description & User Journey

Stan's Robot Shop is an e-commerce platform that allows customers to browse robotic inventory, add items to a shopping cart, proceed to checkout, and process payments. 

The critical user journeys depend on the availability and responsiveness of four core services:
- **`web`**: Serves static assets, routes API traffic, and handles client ingress.
- **`catalogue`**: Provides product lookups, inventory status, and category listings.
- **`cart`**: Maintains active shopping sessions and calculates item subtotals.
- **`payment`**: Validates transactions and interfaces with the payment processing gateway.

---

## 2. Service Level Indicator (SLI) Specification

### Availability SLI
The Availability SLI measures the percentage of successful requests served across the primary ingress:

$$\text{SLI}_{\text{Availability}} = \frac{\sum \text{Successful Requests (HTTP status } < 500\text{)}}{\sum \text{Total Valid Requests}} \times 100\%$$

**Measurement Implementation**:
- Measured continuously via Prometheus scraping cAdvisor container availability, endpoint readiness, and HTTP response codes.
- Formula in PromQL:
  ```promql
  sum(rate(container_network_receive_bytes_total{namespace="robot-shop", pod=~"(web|cart|catalogue|payment)-.*"}[5m]))
  ```
  Combined with pod readiness check:
  ```promql
  avg_over_time(kube_pod_container_status_ready{namespace="robot-shop", container=~"web|cart|catalogue|payment"}[30d]) * 100
  ```

---

## 3. Service Level Objective (SLO)

| Service Tier | Target Availability | Rolling Window | Unavailability Allowed |
| :--- | :--- | :--- | :--- |
| **Production Storefront** | **99.5%** | **30 Days (Rolling)** | **0.5% (216 minutes / 3.6 hours)** |

- **Target**: **99.5%** uptime and successful transaction processing.
- **Error Budget**: **0.5%** over any rolling 30-day window.

---

## 4. Error Budget Calculations

Over a rolling 30-day window (43,200 total minutes):
- **100% Availability**: 43,200 minutes
- **99.5% SLO Target**: 42,984 minutes of expected error-free service
- **Total Error Budget (0.5%)**: **216 minutes** (3 hours, 36 minutes)

### Burn Rate Thresholds

| Burn Rate | Budget Consumed | Time to 100% Exhaustion | Severity | Action Triggered |
| :---: | :---: | :---: | :---: | :--- |
| **1x** | 100% over 30 days | 30 days | Notice | Routine review in weekly catch-up. |
| **2x** | ~10% in 36 hours | 15 days | Warning | Investigate non-critical degradation. |
| **14.4x** | 5% in 6 hours | ~50 hours | High | Page on-call engineer during business hours. |
| **28.8x** | 10% in 2.5 hours | ~25 hours | Critical | PagerDuty / Slack immediate broadcast, incident drill. |

---

## 5. Error Budget Policy & Enforcement Rules

When the error budget is consumed, development and operational priorities shift automatically according to the following thresholds:

### Threshold 1: 0% – 50% Budget Consumed (Healthy)
- **Status**: Normal Operation.
- **Rules**: Teams may ship new features, run load tests, experiment with canary releases, and deploy freely through the standard CI/CD pipeline.

### Threshold 2: 50% – 75% Budget Consumed (Caution)
- **Status**: Heightened Reliability Focus.
- **Rules**:
  1. Mandatory architectural review for all new releases affecting core services (`web`, `cart`, `payment`).
  2. Non-essential infrastructure migrations postponed.
  3. Weekly reliability triage meeting scheduled to review flaky services and logs in Loki.

### Threshold 3: 75% – 99% Budget Consumed (Critical)
- **Status**: Feature Throttling.
- **Rules**:
  1. All non-critical feature releases paused.
  2. 50% of engineering bandwidth diverted to bug fixing, automated testing, and performance optimization.
  3. Rollback triggers tightened: any deployment causing >0.1% budget drop is immediately reverted.

### Threshold 4: 100% Budget Exhaustion (Budget Depleted)
- **Status**: Feature Freeze.
- **Rules**:
  1. **Complete feature freeze**: No new business features may be merged or deployed until the rolling window recovers.
  2. Only **P0 emergency hotfixes**, security patches, and reliability improvements are permitted to ship.
  3. A mandatory **Blameless Postmortem** must be published within 48 hours for the root incident.

---

## 6. Alerting and Runbook Escalation

1. Alerts are triggered by Prometheus rules monitoring budget burn and service degradation.
2. Every alert payload links directly to the designated operational runbook:
   - **Runbook**: [runbook.md](file:///home/faruk/robot-shop/observability/runbook/runbook.md)
3. Incident communication is conducted in the dedicated Slack channel `#robot-shop-alerts`.
