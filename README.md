
# 🔍 EC2 Monitoring POC with Prometheus, Node Exporter & Grafana Cloud

## 📌 Objective

Set up a monitoring system on AWS EC2 instances using:

- **Node Exporter** on 2 EC2 Linux machines (`machine1`, `machine2`)
- **Prometheus** on a 3rd EC2 Linux machine to scrape metrics
- **Grafana Cloud** to visualize and monitor metrics

---

## 🧾 Architecture

```
+---------------------+      +---------------------+      +---------------------+
|     Machine 1       |      |     Machine 2       |      |     Machine 3       |
|  Node Exporter      |      |  Node Exporter      |      |  Prometheus         |
|  Port: 9100         |      |  Port: 9100         |      |  Port: 9090         |
+---------------------+      +---------------------+      +---------------------+
                                     \                     /
                                      \                   /
                                       +-----------------+
                                       |  Grafana Cloud  |
                                       +-----------------+
```

---

## 🔧 Step 1: Install Node Exporter on Machine1 & Machine2

SSH into the EC2 machines:
```bash
ssh ec2-user@<machine-ip>
```

Run the following commands:
```bash
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.8.0.linux-amd64.tar.gz
tar xvf node_exporter-*.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/

sudo useradd -rs /bin/false node_exporter

sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
EOF

sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

Verify Node Exporter is working by visiting:
```
http://<machine-ip>:9100/metrics
```

---

## 🔧 Step 2: Install Prometheus on Machine3

SSH into the Prometheus EC2:
```bash
ssh ec2-user@<machine3-ip>
```

Install Prometheus:
```bash
wget https://github.com/prometheus/prometheus/releases/latest/download/prometheus-2.52.0.linux-amd64.tar.gz
tar xvf prometheus-*.tar.gz
cd prometheus-*
sudo mv prometheus promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```

### ✍️ Update Prometheus Config `/etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'machine1'
    static_configs:
      - targets: ['<machine1-private-ip>:9100']
        labels:
          instance: 'machine1'

  - job_name: 'machine2'
    static_configs:
      - targets: ['<machine2-private-ip>:9100']
        labels:
          instance: 'machine2'
```

Create Prometheus systemd service:
```bash
sudo useradd --no-create-home --shell /bin/false prometheus

sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus Monitoring
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF

sudo mkdir -p /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus

sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
```

Access Prometheus UI:
```
http://<machine3-ip>:9090
```

---

## 🌐 Step 3: Set Up Grafana Cloud

1. Sign up at [Grafana Cloud](https://grafana.com)
2. Create a **Grafana Cloud Stack**
3. Enable **Prometheus integration**
4. Copy the:
   - `remote_write` URL
   - Username
   - API Key

---

## 🔧 Step 4: Connect Prometheus to Grafana Cloud

Update `/etc/prometheus/prometheus.yml` to include the following `remote_write` block:

```yaml
remote_write:
  - url: https://<your-grafana-remote-write-url>
    basic_auth:
      username: <your-grafana-username>
      password: <your-api-key>
```

Restart Prometheus:
```bash
sudo systemctl restart prometheus
```

---

## 📊 Step 5: Monitor Metrics in Grafana

1. Log in to your Grafana Cloud dashboard
2. Go to **Explore** and query:
   ```
   node_cpu_seconds_total{instance="machine1"}
   ```
3. Build dashboards using metrics like:
   - `node_memory_Active_bytes`
   - `node_filesystem_avail_bytes`
   - `node_network_receive_bytes_total`

---

## ✅ Security Group Recommendations

| Port | Description          | Apply To           |
|------|----------------------|--------------------|
| 9100 | Node Exporter        | Machine1 & 2       |
| 9090 | Prometheus UI        | Machine3           |
| 22   | SSH Access           | All Machines       |

---

## 🧠 Notes

- Make sure all EC2 machines are in the same **VPC** or can reach each other.
- Open necessary ports in **Security Groups**.
- You can add more machines by updating Prometheus config with new `job_name` blocks.

---

## 📂 Optional: Automate with Ansible or Terraform

Ask for a playbook or script to automate this setup if needed!
