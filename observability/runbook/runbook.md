# Operational Runbook: Robot Shop Core Service Incident Triage

**Target Services**: `web`, `cart`, `catalogue`, `payment`  
**Namespace**: `robot-shop`  
**Related Alerts**: `RobotShopServiceDown`, `RobotShopHighErrorRate`, `RobotShopErrorBudgetBurnHigh`  
**Related SLO**: [slo-policy.md](file:///home/faruk/robot-shop/observability/slo/slo-policy.md)  
**Dashboards**: [Robot Shop RED Metrics](http://localhost:31300/d/robot-shop-red-metrics/)  

---

## 1. Incident Overview & Initial Assessment

### What Triggered This Alert?
A core service in the `robot-shop` namespace has failed health/readiness checks, has restarted repeatedly, or is failing customer requests at an elevated rate exceeding the 99.5% SLO error budget.

### Immediate Impact
- **`web` failure**: Complete storefront outage; customer requests will return 502/503.
- **`cart` failure**: Users cannot add items, view carts, or proceed to checkout.
- **`catalogue` failure**: Product catalog unavailable; search returns empty lists.
- **`payment` failure**: Checkout failure; revenue-generating transactions are blocked.

---

## 2. Step-by-Step Triage & Diagnostics

### Step 1: Check Pod Status
Run the following command to determine which pod is unhealthy or crashing:
```bash
kubectl get pods -n robot-shop -o wide
```
Look for pods with status `CrashLoopBackOff`, `Error`, `Terminating`, or `0/1 Running`.

### Step 2: Check Recent Pod Events
If a pod is failing readiness or crashing, inspect its events:
```bash
kubectl describe pod -n robot-shop <pod-name>
```
Key sections to look for:
- `Last State`: Look for exit codes (`137` = OOMKilled, `1` = application exception).
- `Events`: Look for failed readiness probes or image pull errors.

### Step 3: Inspect Logs in Loki / Grafana
1. Open **Grafana Explore** (`http://<node-ip>:31300/explore`) and select **Loki**.
2. Query error logs for the impacted service:
   ```logql
   {kubernetes_labels_service="cart", level="error"}
   ```
   Or across all services:
   ```logql
   {kubernetes_namespace_name="robot-shop", level="error"}
   ```
3. Alternatively, check stdout via kubectl:
   ```bash
   kubectl logs -n robot-shop <pod-name> --tail=100
   ```

### Step 4: Check Upstream/Downstream Dependencies
Verify that stateful stores (Redis, MySQL, MongoDB, RabbitMQ) are reachable and healthy:
```bash
kubectl get pods -n robot-shop -l 'service in (redis, mongodb, mysql, rabbitmq)'
```

---

## 3. Mitigation Procedures

### Option A: Restart Stuck Pod (Transient Failure)
If the pod is hung or experiencing a memory leak:
```bash
kubectl delete pod -n robot-shop <pod-name>
```
The ReplicaSet / Deployment will immediately spin up a fresh pod instance.

### Option B: Scale Deployment (Traffic Spike or Starvation)
If the service is starved of resources during a traffic surge from `load-gen`:
```bash
kubectl scale deployment <service-name> -n robot-shop --replicas=2
```

### Option C: Rollback Bad Release
If the failure occurred immediately after a code change or Helm update:
```bash
# Check revision history
helm history robot-shop -n robot-shop

# Roll back to previous revision
helm rollback robot-shop 1 -n robot-shop
```

### Option D: Database Reconnection
If a backend pod cannot connect to its database (e.g. `Connection refused` to `mysql` or `mongodb`):
1. Verify the database service is running:
   ```bash
   kubectl get pods -n robot-shop -l service=mysql
   ```
2. Restart the dependent service once the database is confirmed ready:
   ```bash
   kubectl rollout restart deployment <service-name> -n robot-shop
   ```

---

## 4. Verification & Recovery Checklist

After applying mitigation actions:
1. Verify pod status is `1/1 Running`:
   ```bash
   kubectl get pods -n robot-shop
   ```
2. Check the RED Metrics Dashboard in Grafana:
   - Ensure **Active Robot Shop Pods** returns to `13`.
   - Ensure **Rate** (cart additions or payments) recovers.
   - Ensure **Errors** drop to 0.
3. Check Prometheus Alertmanager:
   - Confirm the alert transition from `Firing` to `Resolved`.
4. If the incident caused >0.1% error budget burn, initiate a **Blameless Postmortem** using the template at [postmortem.md](file:///home/faruk/robot-shop/observability/postmortem/postmortem.md).
