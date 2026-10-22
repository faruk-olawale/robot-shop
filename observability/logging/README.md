# Capstone 2 - Logging Architecture (Loki + Fluent Bit)

This directory contains the complete logging pipeline for **Stan's Robot Shop** microservices on Kubernetes, satisfying the requirements for **Capstone 2: Step 3 (Collect the Logs)**.

---

## 1. Architecture Overview

```
                                  Kubernetes Cluster
+---------------------------------------------------------------------------------------+
|                                                                                       |
|  +--------------------+         +--------------------+         +-------------------+  |
|  | robot-shop Pods    |         | Fluent Bit DS      |         | Loki              |  |
|  | (cart, payment,    | ------> | (DaemonSet in      | ------> | (Deployment in    |  |
|  |  web, catalogue...) | /var/log|  logging ns)       | :3100   |  logging ns)      |  |
|  +--------------------+         +---------+----------+         +---------+---------+  |
|                                           |                              |            |
|                                           | Kubernetes metadata          | PromQL-like|
|                                           | + Log level extraction       | LogQL      |
|                                           v                              v            |
|                               +-----------------------+        +-------------------+  |
|                               | Indexed stream labels |        | Grafana           |  |
|                               | service, level, pod   |        | Explore & Logs    |  |
|                               +-----------------------+        +-------------------+  |
+---------------------------------------------------------------------------------------+
```

1. **Fluent Bit (`fluent-bit.yaml`)**:
   - Runs as a `DaemonSet` on cluster nodes.
   - Reads CRI-formatted logs from `/var/log/containers/*.log`.
   - Uses the `kubernetes` filter to enrich records with pod, namespace, and service metadata.
   - Applies an embedded Lua script (`extract_level.lua`) to inspect log contents and categorize log level (`error`, `warn`, `info`, `debug`).
   - Streams log records directly to Loki on port 3100.

2. **Loki (`loki.yaml`)**:
   - Deployed as a lightweight, single-binary service in the `logging` namespace.
   - Stores indexed streams and chunks in the filesystem.
   - Exposes port `3100` for ingestion (`/loki/api/v1/push`) and querying (`/loki/api/v1/query`).

3. **Grafana Data Source (`grafana-loki-datasource.yaml`)**:
   - Auto-provisioned into Grafana with UID `P8E80F9AEF21F6940`.
   - Proxy target: `http://loki.logging.svc.cluster.local:3100`.

---

## 2. Searching Logs by Service and Error Level

In **Grafana Explore** (select the **Loki** data source) or via the Loki HTTP API, use standard **LogQL** queries:

### A. Search by Service
- View all logs for `cart`:
  ```logql
  {kubernetes_labels_service="cart"}
  ```
- View all logs for `payment`:
  ```logql
  {kubernetes_labels_service="payment"}
  ```
- View all logs for `web`:
  ```logql
  {kubernetes_labels_service="web"}
  ```
- View all logs for `catalogue`:
  ```logql
  {kubernetes_labels_service="catalogue"}
  ```

### B. Search by Error Level
- View all `error` logs for the `cart` service:
  ```logql
  {kubernetes_labels_service="cart", level="error"}
  ```
- View all `error` logs for the `payment` service:
  ```logql
  {kubernetes_labels_service="payment", level="error"}
  ```
- View all `error` logs across all services in `robot-shop`:
  ```logql
  {kubernetes_namespace_name="robot-shop", level="error"}
  ```
- Search for specific error patterns or exceptions:
  ```logql
  {kubernetes_namespace_name="robot-shop"} |= "Exception"
  ```
  ```logql
  {kubernetes_labels_service="web"} |~ "(404|500|502|503)"
  ```

---

## 3. Files in This Directory

- [loki.yaml](file:///home/faruk/robot-shop/observability/logging/loki.yaml): Kubernetes Deployment, Service, and ConfigMap for Grafana Loki.
- [fluent-bit.yaml](file:///home/faruk/robot-shop/observability/logging/fluent-bit.yaml): DaemonSet, RBAC, and Fluent Bit configuration with CRI parsing and log-level extraction.
- [grafana-loki-datasource.yaml](file:///home/faruk/robot-shop/observability/logging/grafana-loki-datasource.yaml): Grafana auto-provisioning ConfigMap for the Loki data source.
- [README.md](file:///home/faruk/robot-shop/observability/logging/README.md): Operational guide and LogQL query reference.
