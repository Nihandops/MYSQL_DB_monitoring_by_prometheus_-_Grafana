# Prometheus & Grafana Monitoring Stack (VM-Based)

This repository contains a **VM-based monitoring system setup** using **Prometheus**, **Node Exporter**, **Grafana**, and an external **PostgreSQL or MySQL** database for Grafana.

The setup is designed for **Ubuntu Server 24.04 LTS** and does **not use Docker or Kubernetes**, making it ideal for learning core Linux-based monitoring fundamentals.

---

## 🧱 Architecture Overview

* **Endpoint Server (VM)**

  * Node Exporter (metrics agent)

* **Monitoring Server (VM)**

  * Prometheus (metrics scraping)
  * Grafana (visualization)

* **Database Server (VM)**

  * PostgreSQL **or** MySQL (Grafana backend DB)

> ⚠️ Although this project is VM-based, the same architecture can be adapted to Docker or Kubernetes with minimal changes.

---

## 🔧 Environment & Versions

| Component     | Version          |
| ------------- | ---------------- |
| OS            | Ubuntu 24.04 LTS |
| Node Exporter | 1.9.1            |
| Prometheus    | 2.53.4           |
| Grafana       | 12.0.2           |
| PostgreSQL    | 16.9             |
| MySQL         | 8.0.42           |

---

## 📊 Prometheus Installation

### Create Prometheus User

```bash
sudo useradd --system --no-create-home --shell /bin/false prometheus
```

### Download & Extract Prometheus

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.53.4/prometheus-2.53.4.linux-amd64.tar.gz
tar -xvf prometheus-2.53.4.linux-amd64.tar.gz
cd prometheus-2.53.4.linux-amd64
```

### Setup Directories & Binaries

```bash
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles console_libraries /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus /data
```

### Create systemd Service

```bash
sudo nano /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

### Start Prometheus

```bash
sudo systemctl daemon-reexec
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```

---

## 🖥️ Node Exporter Installation

### Create User

```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter
```

### Download & Install

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
sudo tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz
sudo mv node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/
```

### systemd Service

```bash
sudo nano /etc/systemd/system/node_exporter.service
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
Restart=on-failure
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

### Start Node Exporter

```bash
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
```

### Firewall Rule

```bash
sudo ufw allow from <ip-prometheus> to any port 9100
```

---

## ⚙️ Prometheus Configuration

```bash
sudo nano /etc/prometheus/prometheus.yml
```

```yaml
- job_name: '<server-name>-node_exporter'
  static_configs:
    - targets: ['<ip-endpoint>:9100']
      labels:
        hostname: <server-name>
```

```bash
sudo promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

---

## 📈 Grafana Installation

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana
sudo systemctl enable grafana-server
sudo systemctl status grafana-server
```

---

## 🗄️ Grafana Database Setup

Grafana supports **PostgreSQL** or **MySQL** as an external database.

👉 Choose **ONE** option below.

### Option A: PostgreSQL

```bash
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo -i -u postgres
createuser <grafana-db-user>
createdb <grafana-db-name> -O <grafana-db-user>
psql
ALTER USER <grafana-db-user> WITH ENCRYPTED PASSWORD '<password>';
```

Enable remote access and whitelist Grafana IP (port 5432).

### Option B: MySQL

```bash
sudo apt install mysql-server
sudo mysql_secure_installation
sudo mysql -u root -p
CREATE DATABASE <grafana-db-name>;
CREATE USER '<grafana-db-user>'@'<grafana-ip>' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON <grafana-db-name>.* TO '<grafana-db-user>'@'<grafana-ip>';
FLUSH PRIVILEGES;
```

Enable remote access and whitelist Grafana IP (port 3306).

---

## 🔌 Configure Grafana Database

```bash
sudo nano /etc/grafana/grafana.ini
```

```ini
[database]
type = postgres | mysql
host = <db-ip>:5432 | 3306
name = <grafana-db-name>
user = <grafana-db-user>
password = <password>
```

```bash
sudo systemctl restart grafana-server
```

---

## 🔗 Integrate Prometheus with Grafana

### Firewall Rule (Prometheus Server)

```bash
sudo ufw allow from <ip-grafana> to any port 9090
```

### Connectivity Test

```bash
nc -zv <ip-prometheus> 9090
```

### Grafana UI Setup

1. Access: `http://<grafana-ip>:3000`
2. Login: `admin / admin`
3. Connections → Add data source → Prometheus
4. URL: `http://<prometheus-ip>:9090`
5. Click **Save & Test**

✅ Data source successfully connected.

---

## ✅ Result

You now have a **production-style monitoring stack** using:

* Prometheus for metrics
* Node Exporter for system monitoring
* Grafana for visualization
* External database for scalability

---

## 📌 Notes

* Replace all placeholders before running commands
* Ensure servers are in the same network/VPC
* Secure ports using firewall rules

---

Happy Monitoring 🚀
