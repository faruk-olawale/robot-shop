# Blameless Postmortem: Cart Microservice Outage (Incident Drill)

**Incident Reference**: INC-20261022-CART-OUTAGE  
**Date**: October 22, 2026  
**Status**: Resolved  
**Severity**: P1 (High Severity - Core Customer Journey Blocked)  
**Incident Commander / Author**: DevOps & SRE Team  
**Related Runbook**: [runbook.md](file:///home/faruk/robot-shop/observability/runbook/runbook.md)  
**Related SLO Policy**: [slo-policy.md](file:///home/faruk/robot-shop/observability/slo/slo-policy.md)  

---

## 1. Executive Summary

On October 22, 2026, at 03:58 UTC, an incident drill was executed where the `cart` deployment in namespace `robot-shop` suffered total pod termination (replicas reduced to 0). This caused immediate failure for all users attempting to add items to their shopping cart or proceed through the storefront checkout funnel. 

Prometheus detected the replica shortage and triggered the critical alert `RobotShopServiceDown` with an embedded link to the operational runbook. Following the runbook triage steps, the on-call engineer identified the 0-replica state and restored the deployment to 1 replica. The service recovered, health checks passed, and the alert cleared at 04:01 UTC. Total outage duration was approximately 2 minutes and 30 seconds, consuming 1.15% of the monthly error budget.

---

## 2. Impact on Customers and SLO

- **Customer Impact**: Storefront visitors browsing products could not add items to their cart or view active carts. API calls to `POST /api/cart/add/*` and `GET /api/cart/*` failed.
- **SLO Target**: 99.5% monthly availability (maximum 216 minutes of allowable downtime per 30-day rolling window).
- **Downtime Incurred**: 2 minutes, 30 seconds (2.5 minutes).
- **Error Budget Consumed**: 
  $$\frac{2.5 \text{ min}}{216 \text{ min}} \times 100\% \approx 1.16\%$$
- **Error Budget Remaining**: **98.84%** (Healthy, no deployment freeze triggered).

---

## 3. Incident Timeline

| Time (UTC) | Elapsed | Event / Action Taken |
| :---: | :---: | :--- |
| **03:58:20** | T+00:00 | **Incident Initiated**: `cart` deployment scaled to 0 replicas during chaos drill. Active cart pod terminated. |
| **03:58:35** | T+00:15 | Synthetic traffic from `load-gen` records failure when contacting cart backend via ingress. |
| **03:58:55** | T+00:35 | Prometheus scrapes kube-state-metrics; evaluates rule `kube_deployment_status_replicas_available == 0`. |
| **03:59:10** | T+00:50 | Alert `RobotShopServiceDown` enters `PENDING` state. |
| **04:00:13** | T+01:53 | Alert reaches threshold (`for: 15s`) and transitions to `FIRING`. Alertmanager broadcasts critical alert with `runbook_url`. |
| **04:00:30** | T+02:10 | On-call engineer opens runbook link from alert; executes diagnostic `kubectl get pods -n robot-shop`. Identifies missing cart pod. |
| **04:01:12** | T+02:52 | **Mitigation Executed**: Engineer runs `kubectl scale deployment cart -n robot-shop --replicas=1`. |
| **04:01:23** | T+03:03 | New pod `cart-75cff6f85d-sqkls` passes readiness probe and begins handling traffic. |
| **04:01:45** | T+03:25 | Prometheus evaluates `replicas_available == 1`. Alert `RobotShopServiceDown` **RESOLVES**. |

---

## 4. Root Cause Analysis (5 Whys)

1. **Why did the shopping cart fail?**  
   The `cart` microservice was completely absent and unable to respond to HTTP requests.
2. **Why was the service absent?**  
   The deployment desired replicas was set to 0.
3. **Why was the deployment desired replicas 0?**  
   A simulated failure was injected during the Capstone 2 incident drill to validate monitoring and alerting.
4. **Why was there a single point of failure?**  
   The `cart` deployment had a replica count of 1 with no Horizontal Pod Autoscaler (HPA) or PodDisruptionBudget (PDB) configured to enforce minimum availability.
5. **Why was there no automated failover?**  
   Single-replica deployments on single-node development clusters cannot distribute replicas across failure domains.

---

## 5. What Went Well

- **Immediate Detection**: The Prometheus alert `RobotShopServiceDown` accurately fired within ~1 minute of replica loss.
- **Actionable Alerting**: The alert payload contained a valid, clickable markdown link directly to the operational runbook ([runbook.md](file:///home/faruk/robot-shop/observability/runbook/runbook.md)).
- **Clear Runbook Instructions**: The runbook diagnostic commands directly isolated the issue and provided the exact command needed to restore service.
- **Fast Recovery**: Once mitigated, the container became ready in under 12 seconds and Prometheus automatically resolved the alert without manual intervention.

---

## 6. What Went Poorly

- **Single Replica Risk**: Running `replicas: 1` meant that any pod restart or scaling anomaly immediately resulted in 100% user-facing unavailability for that microservice.
- **Lack of PodDisruptionBudget**: No Kubernetes policy prevented scaling down the critical deployment to 0.

---

## 7. Where We Got Lucky

- **Steady Load Generation**: The `load-gen` service maintained continuous traffic, making latency, errors, and throughput shifts instantly visible on the RED Metrics Grafana dashboard.
- **Quick Container Initialization**: Node.js `cart` service container cold-start took less than 12 seconds, minimizing overall downtime and error budget burn.

---

## 8. Preventive & Remedial Action Items

| ID | Action Item | Priority | Owner | Target Date |
| :---: | :--- | :---: | :---: | :---: |
| **ACT-01** | Configure minimum `replicas: 2` for core services (`web`, `cart`, `catalogue`, `payment`) in Helm values. | **P1** | Platform / Faruk | Sprint 4 |
| **ACT-02** | Implement `PodDisruptionBudget` (`minAvailable: 1`) to block accidental manual scale-downs or evictions. | **P1** | Platform / Faruk | Sprint 4 |
| **ACT-03** | Configure Horizontal Pod Autoscaler (HPA) using CPU/memory metrics to automatically handle load spikes. | **P2** | SRE Team | Sprint 5 |
| **ACT-04** | Add synthetic end-to-end blackbox probing from `web` through `cart` to capture multi-hop latency regressions. | **P2** | Observability | Sprint 5 |
