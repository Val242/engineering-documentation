# Engineering Log — 2026-10-08

## Summary

Today focused on completing the **Prometheus monitoring infrastructure** for TaskFlow and preparing the Grafana monitoring layer.

The application server was also validated after resolving a Terraform configuration issue that had caused an unintended EC2 replacement.

---

## 1. EC2 / Terraform Issue Resolution

While testing instance persistence after stopping and restarting the Application EC2, the application configuration appeared to have disappeared.

Investigation revealed that Terraform was planning to replace the Application EC2 because of:

```hcl
associate_public_ip_address = false -> true
```

Terraform marked this change as:

```text
# forces replacement
```

This also caused the dependent `aws_ec2_instance_state.web-state` resource to require replacement.

### Resolution

The Terraform configuration was corrected so that the existing EC2 would not be unnecessarily replaced.

### Engineering Lesson

A change to certain EC2 attributes can force instance replacement.

Before applying infrastructure changes:

```bash
terraform plan
```

must be reviewed carefully.

A plan containing:

```text
-/+ aws_instance.web
```

indicates that Terraform intends to replace the instance and should not be applied blindly.

---

## 2. Application Server Recovery

The TaskFlow application server was restored and the systemd configuration was corrected.

The application entry point is:

```text
/opt/taskflow-api/dist/src/main.js
```

The systemd service was configured to use:

```ini
ExecStart=/usr/bin/node /opt/taskflow-api/dist/src/main.js
```

The application health endpoints were verified.

### Liveness

```text
GET /health/live
```

### Readiness

```text
GET /health/ready
```

The readiness endpoint also verifies PostgreSQL connectivity.

---

## 3. UFW and AWS Security Groups

While configuring Prometheus to scrape the application server, Prometheus initially received:

```text
context deadline exceeded
```

The AWS Security Group already contained the correct rule allowing Prometheus to reach the application on TCP port `3000`.

The issue was instead caused by **UFW**, the Linux host firewall.

The required rule was added:

```bash
sudo ufw allow 3000/tcp
```

### Key Concept

AWS Security Groups and UFW operate at different layers:

```text
Prometheus
    ↓
AWS Security Group
    ↓
Application EC2
    ↓
UFW
    ↓
NestJS
```

Both firewalls can independently allow or block traffic.

This demonstrated an important troubleshooting principle:

> A port being allowed by an AWS Security Group does not guarantee that the application is reachable if the host firewall blocks it.

---

## 4. Prometheus Installation

Prometheus was installed on the dedicated Prometheus EC2 instance.

Prometheus is exposed on its default web interface:

```text
:9090
```

The Prometheus configuration is located at:

```text
/etc/prometheus/prometheus.yml
```

Configuration validation can be performed with:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

---

## 5. Prometheus Scrape Configuration

Prometheus was configured with three monitoring targets.

### Prometheus itself

```yaml
- job_name: "prometheus"
  static_configs:
    - targets: ["localhost:9090"]
```

### Application server system metrics

Node Exporter exposes host-level metrics on:

```text
172.31.8.169:9100
```

Configuration:

```yaml
- job_name: "webserver-system-metrics"
  static_configs:
    - targets: ["172.31.8.169:9100"]
      labels:
        app: "infra-web"
```

### TaskFlow application metrics

NestJS exposes Prometheus metrics through:

```text
172.31.8.169:3000/metrics
```

Configuration:

```yaml
- job_name: "webapp-metrics"
  metrics_path: /metrics
  static_configs:
    - targets: ["172.31.8.169:3000"]
      labels:
        app: "infra-web"
```

---

## 6. Health Endpoint vs Metrics Endpoint

An important distinction was established between application health endpoints and Prometheus metrics.

TaskFlow provides:

```text
/health/live
/health/ready
/metrics
```

The health endpoint returns JSON, for example:

```json
{
  "status": "ok",
  "database": "up"
}
```

Prometheus expects Prometheus exposition format, so `/health/ready` should not be configured as a normal Prometheus scrape target.

The responsibilities remain separated:

```text
/health/live
    → Liveness

/health/ready
    → Readiness + database health

/metrics
    → Prometheus metrics
```

If health probing is required later, it can be implemented using a suitable probing mechanism such as Blackbox Exporter.

---

## 7. Prometheus System Metrics

Node Exporter system metrics were successfully exposed to Prometheus.

Examples of useful PromQL queries tested/prepared include:

### CPU usage

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Memory usage

```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```

### Root filesystem usage

```promql
100 * (1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}))
```

### Load average

```promql
node_load1
```

### Network receive rate

```promql
rate(node_network_receive_bytes_total[5m])
```

### Network transmit rate

```promql
rate(node_network_transmit_bytes_total[5m])
```

These metrics will later form the basis of the Grafana infrastructure dashboards.

---

## 8. Grafana

A dedicated Grafana EC2 instance was provisioned and Grafana was installed successfully.

Grafana configuration and dashboards will be completed in the next work session.

The intended architecture is:

```text
Application EC2
 ├── NestJS :3000
 │     └── /metrics
 │
 └── Node Exporter :9100
          │
          ▼
     Prometheus EC2
          │
          │ PromQL
          ▼
      Grafana EC2
```

Grafana will use Prometheus as its data source.

---

## 9. Current Monitoring Architecture

At the end of today's work, the monitoring architecture is:

```text
                    Application EC2
              ┌─────────────────────────┐
              │                         │
              │ NestJS :3000            │
              │   └── /metrics         │
              │                         │
              │ Node Exporter :9100    │
              └────────────┬────────────┘
                           │
                           │ Private network
                           ▼
                  ┌───────────────────┐
                  │  Prometheus EC2   │
                  │                   │
                  │ Prometheus :9090  │
                  └─────────┬─────────┘
                            │
                            │ PromQL
                            ▼
                  ┌───────────────────┐
                  │    Grafana EC2    │
                  │                   │
                  │    Grafana        │
                  └───────────────────┘
```

---

## 10. Next Steps

The next session will focus on Grafana:

1. Access Grafana.
2. Configure Prometheus as the data source.
3. Verify Prometheus connectivity.
4. Build TaskFlow application dashboards.
5. Build system/infrastructure dashboards.
6. Visualize request rate, error rate and latency.
7. Visualize CPU, memory, disk and network metrics.
8. Continue with Alertmanager configuration.

---

## Engineering Takeaways

Today's work reinforced several practical DevOps concepts:

* Terraform changes must be reviewed for resource replacement.
* EC2 replacement can destroy locally stored server configuration.
* AWS Security Groups and host firewalls are separate security layers.
* Prometheus scrapes metrics endpoints rather than arbitrary JSON APIs.
* Node Exporter provides host-level system metrics.
* Application metrics and infrastructure metrics should remain conceptually separate.
* Monitoring architecture should separate metric collection, storage/querying and visualization.
* Network troubleshooting requires checking every layer between the source and destination.
