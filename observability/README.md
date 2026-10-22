# Capstone 2: Keep It Running — Observability Suite

**Project**: Stan's Robot Shop Observability  
**Author**: Ismail Faruk Olawale  
**Cluster**: Kind Kubernetes Cluster (`capstone-k8s-control-plane`)  

This directory contains the complete implementation, configurations, and hand-in documentation for **Capstone 2: Keep It Running**.

---

## 1. Summary of Completed Steps & Deliverables

| Step | Requirement | Implementation / Deliverable | Status |
| :---: | :--- | :--- | :---: |
| **1** | **Deploy and Watch** | Microservices running in `robot-shop` namespace with `load-gen` active and Instana agent stripped. | **COMPLETED** |
| **2** | **See the Metrics** | `kube-prometheus-stack` installed with custom values [prometheus-values.yaml](file:///home/faruk/robot-shop/observability/prometheus-values.yaml), [servicemonitor-robot-shop.yaml](file:///home/faruk/robot-shop/observability/servicemonitor-robot-shop.yaml) scraping `cart` and `payment`, and exported RED dashboard [robot-shop-red-dashboard.json](file:///home/faruk/robot-shop/observability/dashboards/robot-shop-red-dashboard.json). | **COMPLETED** |
| **3** | **Collect the Logs** | Fluent Bit DaemonSet [fluent-bit.yaml](file:///home/faruk/robot-shop/observability/logging/fluent-bit.yaml) shipping CRI logs with Kubernetes metadata and log level extraction to Loki [loki.yaml](file:///home/faruk/robot-shop/observability/logging/loki.yaml). Provisioned Grafana data source [grafana-loki-datasource.yaml](file:///home/faruk/robot-shop/observability/logging/grafana-loki-datasource.yaml). Query guide in [logging/README.md](file:///home/faruk/robot-shop/observability/logging/README.md). | **COMPLETED** |
| **4** | **Follow One Trace** | Jaeger All-in-One [jaeger.yaml](file:///home/faruk/robot-shop/observability/tracing/jaeger.yaml) and OpenTelemetry Collector [otel-collector.yaml](file:///home/faruk/robot-shop/observability/tracing/otel-collector.yaml). Live traced request verified in Jaeger and linked to logs in Grafana [tracing/README.md](file:///home/faruk/robot-shop/observability/tracing/README.md). | **COMPLETED** |
| **5** | **Set One SLO** | 99.5% Availability SLO and 1-page Error Budget Policy in [slo-policy.md](file:///home/faruk/robot-shop/observability/slo/slo-policy.md). PrometheusRule alerting in [prometheus-rules-slo.yaml](file:///home/faruk/robot-shop/observability/slo/prometheus-rules-slo.yaml) with `runbook_url` pointing to [runbook.md](file:///home/faruk/robot-shop/observability/runbook/runbook.md). | **COMPLETED** |
| **6** | **Incident Drill** | Chaos drill executed (cart deployment scaled to 0), verified alert `RobotShopServiceDown` fired, mitigated back to 1 replica, and documented root cause in [postmortem.md](file:///home/faruk/robot-shop/observability/postmortem/postmortem.md). | **COMPLETED** |

---

## 2. Directory Navigation

- [dashboards/](file:///home/faruk/robot-shop/observability/dashboards/): Grafana RED metrics dashboard exported as JSON code.
- [logging/](file:///home/faruk/robot-shop/observability/logging/): Loki deployment, Fluent Bit DaemonSet, Grafana datasource, and LogQL search documentation.
- [tracing/](file:///home/faruk/robot-shop/observability/tracing/): Jaeger All-in-One, OpenTelemetry Collector, Grafana trace datasource, and trace verification notes.
- [slo/](file:///home/faruk/robot-shop/observability/slo/): Availability SLO (99.5%), Error Budget Policy, and Prometheus alerting rules.
- [runbook/](file:///home/faruk/robot-shop/observability/runbook/): Operational incident triage runbook linked from alert annotations.
- [postmortem/](file:///home/faruk/robot-shop/observability/postmortem/): Blameless postmortem document following SRE standards for the incident drill.
