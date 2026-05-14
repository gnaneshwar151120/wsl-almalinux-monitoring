# WSL AlmaLinux Monitoring Setup Guide

This guide explains how to recreate the monitoring stack used in this project.

## Architecture

```text
WSL AlmaLinux → Telegraf → InfluxDB on Windows → Grafana on Windows
```

## Components

| Component | Purpose |
|---|---|
| AlmaLinux on WSL | Linux server environment |
| Telegraf | Metrics collection agent |
| InfluxDB | Time-series metrics database |
| Grafana | Dashboard and visualization layer |
| Nginx | Web server used for service monitoring |
| PostgreSQL 18 | Database used for DB monitoring |

---

## 1. Start InfluxDB and Grafana on Windows

Open these URLs in your browser:

```text
http://localhost:8086
http://localhost:3000
```

Create or confirm the InfluxDB bucket:

```text
server_metrics
```

Create or copy an InfluxDB API token. Do not commit the real token to GitHub.

---

## 2. Install Telegraf on AlmaLinux

```bash
sudo dnf update -y

cat <<EOF | sudo tee /etc/yum.repos.d/influxdata.repo
[influxdata]
name = InfluxData Repository - Stable
baseurl = https://repos.influxdata.com/stable/\$basearch/main
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdata-archive_compat.key
EOF

sudo dnf install telegraf -y
telegraf --version
```

---

## 3. Configure Telegraf Output to InfluxDB

Edit the Telegraf config:

```bash
sudo nano /etc/telegraf/telegraf.conf
```

Configure the InfluxDB v2 output:

```toml
[[outputs.influxdb_v2]]
  urls = ["http://WINDOWS_IP:8086"]
  token = "YOUR_INFLUXDB_TOKEN"
  organization = "YOUR_ORG"
  bucket = "server_metrics"
```

Replace `WINDOWS_IP`, `YOUR_INFLUXDB_TOKEN`, and `YOUR_ORG`.

---

## 4. Enable System Metrics

In `/etc/telegraf/telegraf.conf`, enable or add:

```toml
[[inputs.cpu]]
  percpu = true
  totalcpu = true

[[inputs.mem]]

[[inputs.disk]]

[[inputs.diskio]]

[[inputs.system]]

[[inputs.processes]]

[[inputs.kernel]]

[[inputs.swap]]

[[inputs.net]]
```

---

## 5. Install and Configure Nginx

```bash
sudo dnf install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Create an Nginx status endpoint:

```bash
sudo nano /etc/nginx/conf.d/status.conf
```

Paste:

```nginx
server {
    listen 8090 default_server;

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        allow all;
    }
}
```

Test and restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
curl http://127.0.0.1:8090/nginx_status
```

Add Nginx input to Telegraf:

```toml
[[inputs.nginx]]
  urls = ["http://127.0.0.1:8090/nginx_status"]
```

---

## 6. Configure PostgreSQL Monitoring

Add the PostgreSQL input to Telegraf:

```toml
[[inputs.postgresql]]
  address = "host=127.0.0.1 user=postgres password=YOUR_PASSWORD sslmode=disable"
```

Do not commit the real password to GitHub.

---

## 7. Configure systemd Service Health Monitoring

Add:

```toml
[[inputs.systemd_units]]
  pattern = "*.service"
```

Restart Telegraf:

```bash
sudo systemctl restart telegraf
sudo systemctl status telegraf
```

Verify data is being written:

```bash
sudo journalctl -u telegraf -n 50
```

---

## 8. Connect Grafana to InfluxDB

In Grafana:

```text
Connections → Data sources → Add data source → InfluxDB
```

Use:

| Setting | Value |
|---|---|
| Query language | Flux |
| URL | http://localhost:8086 |
| Organization | your InfluxDB org |
| Token | your API token |
| Default bucket | server_metrics |

Click **Save & Test**.

---

## 9. Import Dashboard

Exported dashboard JSON should be stored at:

```text
grafana/dashboard.json
```

To import:

```text
Grafana → Dashboards → New → Import → Upload JSON
```

---

## 10. Recommended Dashboard Rows

Use these sections:

```text
1. Overview
2. System Performance
3. Nginx Monitoring
4. PostgreSQL Monitoring
5. Linux Service Health
6. Network Monitoring
```

---

## 11. Useful Flux Queries

### CPU Usage

```flux
from(bucket: "server_metrics")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "cpu")
  |> filter(fn: (r) => r.cpu == "cpu-total")
  |> filter(fn: (r) => r._field == "usage_idle")
  |> last()
  |> map(fn: (r) => ({
      _value: 100.0 - r._value
  }))
```

### Memory Usage

```flux
from(bucket: "server_metrics")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "mem")
  |> filter(fn: (r) => r._field == "used_percent")
  |> last()
```

### Nginx Active Connections

```flux
from(bucket: "server_metrics")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "nginx")
  |> filter(fn: (r) => r._field == "active")
  |> last()
```

### PostgreSQL Connections

```flux
from(bucket: "server_metrics")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "postgresql")
  |> filter(fn: (r) => r._field == "numbackends")
  |> group()
  |> sum(column: "_value")
```

### PostgreSQL Transactions/sec

```flux
from(bucket: "server_metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "postgresql")
  |> filter(fn: (r) => r._field == "xact_commit")
  |> derivative(unit: 1s, nonNegative: true)
  |> aggregateWindow(every: 1m, fn: mean)
```

### Linux Service Health

```flux
from(bucket: "server_metrics")
  |> range(start: -10m)
  |> filter(fn: (r) => r._measurement == "systemd_units")
  |> filter(fn: (r) => r._field == "active_code")
  |> filter(fn: (r) =>
      r.name == "nginx.service" or
      r.name == "postgresql-18.service" or
      r.name == "telegraf.service"
  )
  |> group(columns: ["name"])
  |> last()
  |> map(fn: (r) => ({
      Service: r.name,
      Status: r.active,
      SubStatus: r.sub,
      Loaded: r.load
  }))
```

### Network Traffic

```flux
from(bucket: "server_metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "net")
  |> filter(fn: (r) => r._field == "bytes_recv" or r._field == "bytes_sent")
```

---

## 12. WSL Notes

Because AlmaLinux runs inside WSL:

- Disk metrics may be lower or different from a physical Linux server.
- Network metrics come from the WSL virtual network interface.
- Uptime refers to the WSL instance, not the Windows host.
- Some hardware-level metrics may not behave like a cloud VM or bare-metal server.

---

## 13. Security Notes

Never commit:

```text
InfluxDB token
PostgreSQL password
Real internal credentials
Private secrets
```

Use placeholders in GitHub examples:

```text
YOUR_INFLUXDB_TOKEN
YOUR_PASSWORD
YOUR_ORG
```

---

## 14. Project Outcome

This project demonstrates:

- Linux observability
- Telegraf metric collection
- InfluxDB time-series storage
- Grafana dashboard design
- Nginx service monitoring
- PostgreSQL database monitoring
- systemd service health monitoring
- dashboard variables and reusable queries
