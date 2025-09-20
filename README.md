# Monitoring FortiGate with Prometheus, VictoriaMetrics, and Grafana

This repository documents how to set up monitoring for a **FortiGate firewall** using the [FortiGate Exporter](https://github.com/prometheus-community/fortigate_exporter), integrated with **VictoriaMetrics (or Prometheus)** and visualized in **Grafana**.

---

## 1. Create FortiGate API User and Token

On the **FortiGate dashboard**:

1. Go to **System → Administrators**
2. Click **Create New → REST API Admin**
3. Assign **read-only permissions** for monitoring
4. Copy the generated **API token**
<img width="3018" height="1640" alt="4f1f61ea-cabd-422d-848d-50a66ea639d8_3018x1640" src="https://github.com/user-attachments/assets/bb3064f8-2969-43f3-bb05-958d353d7808" />
---

## 2. Download and Configure FortiGate Exporter

```bash
# Create a directory for the exporter
sudo mkdir -p /opt/fortigate_exporter

# Download the latest Linux AMD64 binary (v1.24.1 - 08.09.2025)
sudo wget https://github.com/prometheus-community/fortigate_exporter/releases/download/v1.24.1/fortigate-exporter.linux.amd64 \
  -O /opt/fortigate_exporter/fortigate_exporter

# Make the binary executable
sudo chmod +x /opt/fortigate_exporter/fortigate_exporter
```

3. Create the Configuration File
```
cd /etc/fortigate/
vim fortigate-key.yaml
```
```
"https://[Add your fortigate IP address]": 
  token: "[Generated fortigate API token]"
  tls:
    insecure_skip_verify: true

  # Optionally, limit collection to specific probes:
  # probes:
  #   include:
  #     - System
  #     - System/Interface
  #     - VPN
```
4. Create a Systemd Service
```
cd /etc/systemd/system/
vim fortigate_exporter.service
```
```
[Unit]
Description=FortiGate Exporter
After=network.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/opt/fortigate_exporter/fortigate_exporter \
  -auth-file /etc/fortigate/fortigate-key.yaml \
  -listen=:9710 -insecure
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
Reload and start:
```
systemctl daemon-reload
systemctl enable fortigate_exporter
systemctl start fortigate_exporter
systemctl status fortigate_exporter
```
5. Configure Prometheus / VictoriaMetrics scrape config snippet:
```
cd /etc/victoriametrics/ 
vim victoriametrics-scrape.yaml
```
```
scrape_configs:
  - job_name: "fortigate"
    metrics_path: /probe
    scheme: http
    params:
      token: ["fortigate-api-token"]
    static_configs:
      - targets: ["https://fortigate-ip-address"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: "grafana-node-ip:9710"

  - job_name: "fortigate_exporter"
    metrics_path: /metrics
    static_configs:
      - targets: ["grafana-node-ip:9710"]
```
Restart and check the service and target::
```
systemctl restart victoria
systemctl status victoria
```

Check the target is UP:
```
http://grafana-node-ip:8428/targets
```

6. Grafana Dashboard Setup
Navigate to Configuration → Data Sources → Add data source → select Prometheus URL:

http://grafana-node-ip:9090 (Prometheus)

http://grafana-node-ip:8428 (VictoriaMetrics)

Save & test the data source connection.

Then import a Grafana dashboard using the community template:

```
Dashboard ID: 14011
```

This is the official FortiGate Prometheus Exporter Dashboard, designed for the fortigate_exporter and supports multi-instance setups via variables, Grafana Labs.

To import:

Go to Dashboards → Import in Grafana.

Paste the ID 14011 or upload the .json file.

Map your data source, then import.

You’ll get out-of-the-box graphs for CPU/memory usage, interfaces, VPN, sessions, and more. That’s it, FortiGate monitored successfully.

<img width="3012" height="1628" alt="854331e4-ba6f-49c3-95e3-6b1f5d3db377_3012x1628" src="https://github.com/user-attachments/assets/8fc0df46-dd05-4f2e-9129-36b39a152657" />

