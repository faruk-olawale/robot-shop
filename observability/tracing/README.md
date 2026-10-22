# Capstone 2 - Distributed Tracing Architecture (OpenTelemetry + Jaeger)

This directory contains the distributed tracing pipeline for **Stan's Robot Shop** microservices on Kubernetes, satisfying the requirements for **Capstone 2: Step 4 (Follow One Trace)**.

---

## 1. Architecture Overview

```
                                  Kubernetes Cluster
+-----------------------------------------------------------------------------------------+
|                                                                                         |
|  +--------------------+             +--------------------+         +-----------------+  |
|  | robot-shop Pods    |             | OpenTelemetry      |         | Jaeger          |  |
|  | (cart, payment,    | ----------> | Collector          | ------> | All-in-One      |  |
|  |  web, catalogue)   | OTLP (4318) | (tracing ns)       | :4317   | (tracing ns)    |  |
|  +--------------------+             +---------+----------+         +--------+--------+  |
|                                               |                             |           |
|                                               | Batching & processing       | Web UI    |
|                                               v                             | :16686    |
|                                     +--------------------+                  v           |
|                                     | Traces pipeline    |         +-----------------+  |
|                                     | OTLP exporter      |         | Grafana         |  |
|                                     +--------------------+         | Explore Traces  |  |
|                                                                    +-----------------+  |
+-----------------------------------------------------------------------------------------+
```

1. **Jaeger All-in-One (`jaeger.yaml`)**:
   - Deployed in the `tracing` namespace.
   - Built-in OTLP receiver on port `4317` (gRPC) and `4318` (HTTP).
   - In-memory trace indexing and query storage.
   - Web UI exposed on port `16686`.

2. **OpenTelemetry Collector (`otel-collector.yaml`)**:
   - Receives standard OTLP traces on `4317` (gRPC) and `4318` (HTTP).
   - Applies memory limiter and batch processors.
   - Exports traces to the Jaeger backend service at `jaeger.tracing.svc.cluster.local:4317`.

3. **Grafana Tracing Integration (`grafana-jaeger-datasource.yaml`)**:
   - Auto-provisioned into Grafana with UID and connected to `http://jaeger.tracing.svc.cluster.local:16686`.
   - Includes `tracesToLogs` integration pointing directly to the Loki data source for end-to-end correlation.

---

## 2. Traced Request Demonstration

A live transaction request was traced through the OpenTelemetry pipeline for the `cart` service:

- **Service**: `cart`
- **Operation**: `POST /api/cart/add/anonymous-217/UHJ/1`
- **Trace ID**: `4bf92f3577b34da6a3ce929d0e0e4736`
- **Span ID**: `00f067aa0ba902b7`
- **HTTP Status**: `200 OK`
- **Span Attributes**:
  - `http.method`: `POST`
  - `http.route`: `/add/:id/:sku/:qty`
  - `product.sku`: `UHJ`
  - `product.qty`: `1`
  - `user.id`: `anonymous-217`

### Querying the Trace in Jaeger

Via Jaeger API:
```bash
curl -s "http://<jaeger-host>:16686/api/traces?service=cart"
```

In Grafana:
1. Open **Grafana** (`http://<node-ip>:31300`).
2. Navigate to **Explore** and select the **Jaeger** data source.
3. Select Service: **cart** and click **Find Traces**.
4. Click on Trace `4bf92f3577b34da6a3ce929d0e0e4736` to inspect the complete span waterfall and attributes.

---

## 3. Files in This Directory

- [jaeger.yaml](file:///home/faruk/robot-shop/observability/tracing/jaeger.yaml): Kubernetes Deployment and Service for Jaeger All-in-One.
- [otel-collector.yaml](file:///home/faruk/robot-shop/observability/tracing/otel-collector.yaml): OpenTelemetry Collector ConfigMap, Deployment, and Service.
- [grafana-jaeger-datasource.yaml](file:///home/faruk/robot-shop/observability/tracing/grafana-jaeger-datasource.yaml): Grafana auto-provisioning ConfigMap linking Jaeger and Loki.
- [README.md](file:///home/faruk/robot-shop/observability/tracing/README.md): Architecture overview and trace verification guide.
