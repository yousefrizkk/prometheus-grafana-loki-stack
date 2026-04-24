# 📡 AWS Monitoring Stack

> Full-stack observability platform built on AWS EC2 — Metrics, Logs, Alerts, and Dashboards in one unified setup.

![Stack](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-F5A800?style=for-the-badge&logo=grafana&logoColor=white)
![AWS](https://img.shields.io/badge/AWS_EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)

---

## 🧱 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     AWS EC2 Instances                    │
│                                                         │
│  ┌──────────────┐     scrape      ┌──────────────────┐  │
│  │  Web Server  │ ──────────────► │   Prometheus     │  │
│  │              │  :9100 (Node)   │   :9090          │  │
│  │  Titan App   │  :5000 (App)    └────────┬─────────┘  │
│  │  Node Exporter│                         │            │
│  │  Grafana Alloy│ ──── logs ──►  ┌────────▼─────────┐  │
│  └──────────────┘                 │      Loki        │  │
│                                   │      :3100       │  │
│                                   └────────┬─────────┘  │
│                                            │            │
│                                   ┌────────▼─────────┐  │
│                                   │     Grafana      │  │
│                                   │     :3000        │  │
│                                   └────────┬─────────┘  │
│                                            │            │
│                                   ┌────────▼─────────┐  │
│                                   │   Slack Alerts   │  │
│                                   └──────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 🖥️ EC2 Instances

| Instance         | Role                              | Key Ports  |
| ---------------- | --------------------------------- | ---------- |
| `grafana-ec2`    | Visualization & Alerting          | 3000       |
| `prometheus-ec2` | Metrics Collection & Storage      | 9090       |
| `loki-ec2`       | Log Aggregation                   | 3100       |
| `web-ec2`        | Titan App + Node Exporter + Alloy | 5000, 9100 |

---

## 🔧 Stack Components

### Metrics Pipeline

- **Prometheus** — scrapes system and app metrics every 15s
- **Node Exporter** — exposes OS-level metrics (CPU, RAM, Disk, Network)
- **Titan App** — custom Python web app exposing `/metrics` endpoint on `:5000`

### Logs Pipeline

- **Grafana Alloy** — agent that reads logs from `/var/log/titan` and ships to Loki
- **Loki** — log storage and querying engine, stores chunks locally

### Visualization & Alerting

- **Grafana** — unified dashboards for metrics (Prometheus) and logs (Loki)
- **Slack Integration** — alert delivery via Incoming Webhooks

---

## 📁 Repository Structure

```
aws-monitoring-stack/
├── README.md
├── scripts/
│   ├── grafana-setup.sh          # Grafana EC2 userdata
│   ├── prometheus-setup.sh       # Prometheus EC2 userdata
│   ├── loki-setup.sh             # Loki EC2 userdata
│   └── webserver-setup.sh        # Web EC2 userdata (App + Node Exporter + Alloy)
├── config/
│   ├── prometheus.yml            # Scrape jobs configuration
│   └── alloy-config.alloy        # Log shipping config (Alloy agent)
└── dashboards/
    └── titan-monitoring.json     # Grafana dashboard export
```

---

## 🚀 Setup Guide

### Prerequisites

- AWS Account with EC2 access
- 4 EC2 instances (Ubuntu 22.04 recommended, t2.micro or t3.small)
- Security Groups configured per instance (see below)

### Security Group Rules

| Instance   | Inbound Rules                                        |
| ---------- | ---------------------------------------------------- |
| Grafana    | SSH (My IP), TCP 3000 (My IP)                        |
| Prometheus | SSH (My IP), TCP 9090 (My IP + Grafana SG)           |
| Loki       | SSH (My IP), TCP 3100 (Anywhere)                     |
| Web Server | SSH (My IP), TCP 5000 + 9100 (My IP + Prometheus SG) |

### Deployment Steps

**1. Launch EC2 instances with userdata scripts**

```bash
# Paste the content of each script into EC2 → Advanced → User data
scripts/grafana-setup.sh      → Grafana EC2
scripts/prometheus-setup.sh   → Prometheus EC2
scripts/loki-setup.sh         → Loki EC2
scripts/webserver-setup.sh    → Web EC2
```

**2. Update Prometheus scrape targets**

```yaml
# config/prometheus.yml — replace with your Web EC2 private IP
- targets: ["<WEBSERVER_PRIVATE_IP>:9100"] # Node Exporter
- targets: ["<WEBSERVER_PRIVATE_IP>:5000"] # Titan App
```

**3. Update Alloy config with your endpoints**

```
# config/alloy-config.alloy
# Replace Loki IP and Prometheus remote_write URL
url = "http://<PROMETHEUS_PRIVATE_IP>:9090/api/v1/write"
url = "http://<LOKI_PRIVATE_IP>:3100/loki/api/v1/push"
```

**4. Restart Prometheus after config update**

```bash
sudo systemctl restart prometheus.service
```

**5. Verify Loki is healthy**

```bash
curl http://localhost:3100/ready
curl -s http://localhost:3100/metrics | grep loki_build_info
```

---

## 📊 Grafana Dashboards

### titan prod Dashboard

| Panel                       | Query                                                                                                    | Type        |
| --------------------------- | -------------------------------------------------------------------------------------------------------- | ----------- |
| HTTP Request Rate (req/sec) | `rate(http_requests_total{job="webserver-appstat"}[1m])`                                                 | Time Series |
| Available Memory (%)        | `(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100`                                    | Gauge       |
| CPU Utilization (%)         | `100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`                       | Time Series |
| Root FileSystem Usage (%)   | `(1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100` | Gauge       |
| Total Requests by Endpoint  | `sum by (endpoint) (http_requests_total{job="webserver-app"})`                                           | Time Series |

### Dashboard Variables

- `$endpoint` — dynamic filter using `label_values(http_requests_total, endpoint)`

---

## 🚨 Alerting

### Alert Rule: RootDiskAlert

- **Condition:** Disk usage > 65%
- **Evaluation:** every 1 minute
- **Contact Point:** Slack (`#titan-prod` channel)
- **Notification Policy:** matched by label `alertname=RootdiskAlert`

### Slack Integration

1. Create workspace + channel in Slack
2. Go to [api.slack.com/apps](https://api.slack.com/apps) → Create App → Incoming Webhooks
3. Copy Webhook URL → Grafana → Alerting → Contact Points

---

## 📈 Load Testing

```bash
# Generate CPU/network load on Web EC2
nohup /usr/local/bin/load.sh > /dev/null 2>&1 &

# Generate multi-app log files
nohup /usr/local/bin/generate_multi_logs.sh &

# Verify load script is running
ps -ef | grep load.sh
```

---

## 🧰 Tech Stack

| Tool          | Version       | Purpose                      |
| ------------- | ------------- | ---------------------------- |
| Prometheus    | Latest stable | Metrics collection & storage |
| Grafana       | Latest stable | Dashboards & alerting        |
| Loki          | Latest stable | Log aggregation              |
| Grafana Alloy | Latest stable | Log shipping agent           |
| Node Exporter | Latest stable | OS metrics                   |
| AWS EC2       | Ubuntu 22.04  | Infrastructure               |
| Slack         | —             | Alert notifications          |

---

## 📝 Notes

- All IPs in config files are placeholders — replace before deployment
- Loki storage: `/var/lib/loki/chunks` and `/var/lib/loki/rules`
- Prometheus storage: `/var/lib/prometheus`
- Titan App logs: `/var/log/titan`

---

## 👤 Author

**Yousef** — DevOps & Cloud Engineering Student  
🔗 [LinkedIn](#) | [GitHub](#)

---

> _Built as a hands-on observability project to practice real-world monitoring architecture on AWS._
