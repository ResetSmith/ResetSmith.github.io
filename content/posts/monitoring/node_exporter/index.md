---
title: "Installing Node Exporter"
date: 2021-08-19T10:03:55-07:00
hero: bitnami_node_exporter.png
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Installing Node Exporter
    identifier: monitoring-node_exporter
    parent: monitoring
    weight: 60
---

### create folder for exporter apps
```
sudo mkdir /usr/local/bin/metrics
sudo mkdir /etc/metrics
sudo useradd -rs /bin/false exporter
sudo chmod -R 777 /usr/local/bin/metrics
sudo chmod -R 777 /etc/metrics
sudo chown -R exporter:exporter /usr/local/bin/metrics
sudo chown -R exporter:exporter /etc/metrics
```

### Install node_exporter
```
curl -O -L https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
tar xzvf node_exporter-*.*-amd64.tar.gz
sudo mv node_exporter-*.*-amd64 /usr/local/bin/node_exporter
```

### create service
```
sudo vim /etc/systemd/system/node_exporter.service

[Unit]
Description=Node exporter for Prometheus
After=network.target

[Service]
User=exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter/node_exporter

[Install]
WantedBy=multi-user.target
```

### start and enable service
```
sudo systemctl start node_exporter.service
sudo systemctl status node_exporter.service
sudo systemctl enable node_exporter.service
```

### open firewall on server
```
sudo ufw allow from $PROMETHEUS_SERVER proto tcp to any port 9100
```
