# Prometheus + Grafana Monitoring Stack

![Linux](https://img.shields.io/badge/Linux-000000?style=for-the-badge&logo=linux&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Monitoring](https://img.shields.io/badge/Monitoring-3178C6?style=for-the-badge)

> Linux Server Monitoring Stack — CentOS / RHEL Based Setup
> Multi-node monitoring with Prometheus, Node Exporter and Grafana Dashboard

---

## What is Prometheus?

Prometheus is an open-source tool that collects and stores data about your servers and applications.
It records real-time numbers like CPU usage, memory usage, disk space and number of users using an application.

It works on a pull model, which means it actively goes to your servers and checks their current state and records that information every few seconds or minutes.
This makes it easy to know if any node or machine is down — if Prometheus cannot reach it, something is wrong.

---

## What is Grafana?

Grafana is a visualization platform. It connects to various data sources like Prometheus, Elasticsearch, Loki and turns that stored data into dashboards with graphs, charts, tables and alerts.

For example: We can see a graph showing how our CPU has been doing in the past 24 hours.

---

## Why Prometheus and Grafana are Used Together

Prometheus collects and stores metrics. Grafana visualizes them.
Prometheus does not have a strong visualization layer, and Grafana does not store metrics.
Together they form the standard monitoring stack in DevOps environments.

---

## What is Node Exporter?

Node Exporter is a small program that runs on Linux machines. It collects information about the server — CPU, memory, disk, network — and exposes them on port 9100 for Prometheus to collect.

---

## What is Alert Manager?

Alert Manager handles alerts. When Prometheus finds a problem, it sends a message to Alert Manager. Alert Manager decides what to do — it can send an email, a Slack message, or group many alerts together.

---

## Workflow

```
Linux Server (Node Exporter :9100) --> Prometheus (Metrics DB :9090) --> Grafana Dashboard (:3000)
```

---

## Important Concepts

**a) Pull Model**
Prometheus pulls metrics from nodes. It does not wait for nodes to send data. This makes it easy to detect when a target is down — if Prometheus cannot pull metrics, the node is down.

**b) Time-Series Data**
Each data point has a timestamp and value. This allows us to see trends — like how CPU usage changed over the last hour or how disk space decreased over the last week.

**c) Labels**
Labels are key-value pairs attached to metrics. For example: CPU metrics can have labels like `instance="10.0.0.1"` and `mode="idle"`. This makes it easy to filter and group data.

**d) PromQL (Prometheus Query Language)**
With PromQL we can extract and manipulate metrics. A simple query like `node_memory_MemAvailable_bytes` gives us available memory.

**e) Metrics**
Metrics is nothing but the state of a Linux machine — CPU usage percentage, memory usage, disk space, network traffic, system load, and uptime.

---

## Monitoring Flow — How It All Works

**Step 1:** We install Node Exporter on each server we want to monitor. It constantly collects system metrics.

**Step 2:** We configure Prometheus with the IP addresses of our Linux machines. At regular intervals it makes HTTP requests to each Node Exporter, pulls the metrics and stores them in its time-series database.

**Step 3:** Each metric is stored with a timestamp and labels like `server=vm1` or `cpu=0`. This makes it easy to filter and group data later.

**Step 4:** When we want to see a particular metric like CPU usage of last 1 hour, we write a query using PromQL.

**Step 5:** Grafana connects with Prometheus and runs these queries on our behalf. We build dashboards where each graph or table is backed by a PromQL query. Grafana runs the query and shows the result.

**Step 6:** We can define alerting rules in Prometheus. For example: CPU staying above 80% for 5 minutes. Prometheus sends an alert to Alert Manager which then notifies the team.

---

## Common Metrics to Monitor

| Metric | Description |
|--------|-------------|
| `node_cpu_seconds_total` | Total CPU time spent in different modes |
| `node_memory_MemAvailable_bytes` | Amount of memory available for new processes |
| `node_filesystem_avail_bytes` | Free disk space on mounted filesystems |
| `node_network_receive_bytes_total` | Total network traffic received |
| `node_load1, node_load5, node_load15` | System load averages over 1, 5 and 15 minutes |

---

## VM Setup

| VM | Role | IP |
|----|------|----|
| LVM1 | Prometheus + Grafana Server | 192.168.253.128 |
| LVM2 | Target Machine 1 (Node Exporter) | 192.168.253.129 |
| LVM3 | Target Machine 2 (Node Exporter) | 192.168.253.130 |

---

## Port Summary

| Service | Port |
|---------|------|
| Prometheus | 9090 |
| Node Exporter | 9100 |
| Grafana | 3000 |
| Alert Manager | 9093 |

---

# Practical

## Step 1 — Prepare the System

```bash
# Update all packages
dnf update -y
reboot
```

```bash
# Disable SELinux so it does not block Prometheus
setenforce 0

# To make this permanent across reboots
vi /etc/selinux/config
# change SELINUX=enforcing to SELINUX=permissive
```

---

## Step 2 — Create Prometheus User and Directories

```bash
# Create prometheus user with no home directory and no login shell
# This user runs the service with minimal privileges
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
```

```bash
# /etc/prometheus holds config files
# /var/lib/prometheus holds metrics data
mkdir /etc/prometheus
mkdir /var/lib/prometheus
```

```bash
# Set ownership so prometheus user can read/write these directories
chown -R prometheus:prometheus /etc/prometheus
chown -R prometheus:prometheus /var/lib/prometheus
```

---

## Step 3 — Download and Install Prometheus

```bash
cd /tmp

# Download Prometheus tarball from GitHub releases
wget https://github.com/prometheus/prometheus/releases/download/v3.10.0/prometheus-3.10.0.linux-amd64.tar.gz

# Extract
tar -xf prometheus-3.10.0.linux-amd64.tar.gz

cd prometheus-3.10.0.linux-amd64
```

```bash
# Copy binaries to /usr/local/bin so they can be run from anywhere
cp prometheus /usr/local/bin/
cp promtool /usr/local/bin/

# Set ownership to prometheus user
chown prometheus:prometheus /usr/local/bin/prometheus
chown prometheus:prometheus /usr/local/bin/promtool
```

```bash
# Copy web console files used by Prometheus web UI
cp -r consoles /etc/prometheus/
cp -r console_libraries /etc/prometheus/
cp prometheus.yml /etc/prometheus/prometheus.yml
```

---

## Step 4 — Create Prometheus Systemd Service

```bash
vi /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl start prometheus
systemctl enable prometheus
systemctl status prometheus
```

---

## Step 5 — Open Firewall for Prometheus

```bash
# Open port 9090 so Prometheus web UI is accessible from browser
firewall-cmd --permanent --add-port=9090/tcp --zone=public
firewall-cmd --reload
```

Access Prometheus dashboard at `http://192.168.253.128:9090`
![Prometheus Targets](https://raw.githubusercontent.com/dpatil787/linux-server-configurations/main/Prometheus/images/1-target.png)

Go to **Status → Targets** — Prometheus should be scraping itself. This confirms installation is successful.

---

## Step 6 — Install Node Exporter

Node Exporter runs on every Linux server we want to monitor. Install this on LVM1 first, then repeat on LVM2 and LVM3.

```bash
mkdir -p /var/lib/prometheus/node_exporter

cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
tar xvf node_exporter-1.10.2.linux-amd64.tar.gz
cd node_exporter-1.10.2.linux-amd64

cp node_exporter /usr/local/bin/
```

```bash
# Create dedicated user and set permissions
useradd --no-create-home --shell /usr/sbin/nologin node_exporter
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

```bash
vi /usr/lib/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter
```

```bash
# Open firewall for Node Exporter — port 9100
firewall-cmd --permanent --add-port=9100/tcp --zone=public
firewall-cmd --reload
```

Verify Node Exporter metrics endpoint at `http://192.168.253.128:9100/metrics` — we should see a long list of metrics.

![Node Exporter Metrics](https://raw.githubusercontent.com/dpatil787/linux-server-configurations/main/Prometheus/images/2-metrics.png)

---

## Step 7 — Add Node Exporter to Prometheus Config

```bash
vi /etc/prometheus/prometheus.yml
```

At the end of the file add this block. YAML is space-sensitive so make sure indentation is correct.

```yaml
  - job_name: "node_exporter"
    static_configs:
      - targets: ["192.168.253.128:9100"]
```


```bash
# Always validate config before restarting
promtool check config /etc/prometheus/prometheus.yml

systemctl restart prometheus
systemctl status prometheus
```

Check Prometheus Targets page — both prometheus and node_exporter should show as UP.

```
http://192.168.253.128:9090/targets
```
![Prometheus Dashboard](https://raw.githubusercontent.com/dpatil787/linux-server-configurations/main/Prometheus/images/3-prometheus.png)


---

## Step 8 — Install Grafana

For Prometheus we used tarball installation because no official DNF repository exists for RHEL systems. For Grafana, an official repository exists so we use that.

```bash
vi /etc/yum.repos.d/grafana.repo
```

```ini
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

```bash
# Force DNF to read the new repo and cache available packages
dnf makecache -y

dnf install grafana -y

systemctl start grafana-server
systemctl enable grafana-server
systemctl status grafana-server
```

```bash
# Open firewall for Grafana — port 3000
firewall-cmd --permanent --add-port=3000/tcp
firewall-cmd --reload
```

Access Grafana at `http://192.168.253.128:3000`

Default login: `admin / admin` — change password on first login.

---

## Step 9 — Add Prometheus as Data Source in Grafana

- Left sidebar → **Connections** → **Data Sources**
- Click **Add data source** → select **Prometheus**
- In URL field enter: `http://192.168.253.128:9090`
- Click **Save & Test**

![Grafana Prometheus Integration](https://raw.githubusercontent.com/dpatil787/linux-server-configurations/main/Prometheus/images/4-grafanaprometheus.png)

Green message should appear confirming data source is working.

![Data Source Verified](https://raw.githubusercontent.com/dpatil787/linux-server-configurations/main/Prometheus/images/5-integration.png)

---

## Step 10 — Import Dashboard 1860

Dashboard 1860 is a community-built Grafana dashboard for Node Exporter. It shows CPU, RAM, disk, network and load in a clean visual format.

Direct import by ID may fail if Grafana cannot reach grafana.com from the VM. Download the JSON on your Windows machine and upload manually.

Download JSON:
```
https://grafana.com/api/dashboards/1860/revisions/latest/download
```
![Dashboard JSON Import](https://raw.githubusercontent.com/dpatil787/linux-server-configurations/main/Prometheus/images/6-dashboardjsonimport.png)

- Go to **Dashboards** → **Import**
- Click **Upload JSON file** → select downloaded file
- Select **Prometheus** as data source
- Click **Import**

![Local Dashboard](https://raw.githubusercontent.com/dpatil787/linux-server-configurations/main/Prometheus/images/7-localdashboard.png)

---

# Adding More VMs for Monitoring

## Install Node Exporter on LVM2 and LVM3

Run these steps on each additional VM:

```bash
dnf update -y

useradd --no-create-home --shell /usr/sbin/nologin node_exporter

cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
tar -xvf node_exporter-1.10.2.linux-amd64.tar.gz
cd node_exporter-1.10.2.linux-amd64

cp node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

```bash
vi /usr/lib/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter

firewall-cmd --permanent --add-port=9100/tcp --zone=public
firewall-cmd --reload
```

Verify both VMs are accessible:

```
http://192.168.253.129:9100/metrics
http://192.168.253.130:9100/metrics
```

---

## Add LVM2 and LVM3 to Prometheus Config

On the Prometheus server (LVM1):

```bash
vi /etc/prometheus/prometheus.yml
```

Add both VMs at the end:

```yaml
  - job_name: "node_exporter_LVM2"
    static_configs:
      - targets: ["192.168.253.129:9100"]

  - job_name: "node_exporter_LVM3"
    static_configs:
      - targets: ["192.168.253.130:9100"]
```
![prometheus.yml Config](https://raw.githubusercontent.com/dpatil787/linux-server-configurations/main/Prometheus/images/8-prometheusyml.png)

```bash
promtool check config /etc/prometheus/prometheus.yml
systemctl restart prometheus
```

Check targets page — all 3 VMs should show as UP.

```
http://192.168.253.128:9090/targets
```

![Updated Targets](https://raw.githubusercontent.com/dpatil787/linux-server-configurations/main/Prometheus/images/9-targetupdated.png)

---

## View All VMs in Grafana

The 1860 dashboard has a **job** dropdown at the top. After adding more VMs, each VM appears as a separate option. Select any VM to see its metrics. No changes needed in Grafana — Prometheus feeds the data automatically.

![Linux Nodes in Grafana](https://raw.githubusercontent.com/dpatil787/linux-server-configurations/main/Prometheus/images/10-linuxnodes.png)

---

# Output

- Prometheus running and scraping metrics on port 9090
- Node Exporter exposing Linux system metrics on port 9100
- All 3 targets showing UP in Prometheus targets page
- Grafana connected to Prometheus as data source
- Dashboard 1860 showing CPU, memory, disk, and network metrics in real time
- Job dropdown in Grafana showing all 3 VMs separately

---

## Key Files and Directories

| Path | Description |
|------|-------------|
| `/etc/prometheus/prometheus.yml` | Main Prometheus config file |
| `/var/lib/prometheus` | Metrics data storage |
| `/etc/systemd/system/prometheus.service` | Prometheus service file |
| `/usr/lib/systemd/system/node_exporter.service` | Node Exporter service file |
| `/etc/yum.repos.d/grafana.repo` | Grafana repository file |
| `/var/log/grafana/grafana.log` | Grafana logs |

---

# Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Prometheus service fails to start | Wrong permissions on directories | `chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus` |
| Targets showing DOWN in Prometheus | Firewall blocking port 9100 | `firewall-cmd --add-port=9100/tcp --zone=public --permanent` |
| Grafana cannot reach Prometheus | Wrong URL in data source | Use `http://<server-ip>:9090` not localhost |
| Dashboard 1860 import fails - Bad Gateway | Grafana cannot reach grafana.com | Download JSON on Windows, upload manually |
| Node Exporter not showing metrics | Service not running | `systemctl status node_exporter` and check logs |
| promtool check fails | YAML indentation error in prometheus.yml | Check spacing — YAML is space-sensitive |

---

*Document prepared as part of DevOps Home Lab — Linux Server Configurations*
