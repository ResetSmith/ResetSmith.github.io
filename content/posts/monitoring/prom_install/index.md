---
title: "Installing Prometheus"
date: 2021-10-03
hero: data_security_15.jpg
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Installing Prometheus
    identifier: monitoring-prometheus
    parent: monitoring
    weight: 10
---


## Create a User
```
sudo useradd -rs /bin/false prometheus
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

## Download Prometheus
```
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.13.1/prometheus-2.13.1.linux-amd64.tar.gz
tar -xzvf prometheus-2.13.1.linux-amd64.tar.gz prometheus/
sudo mv prometheus/prometheus /usr/local/bin/
sudo mv prometheus/promtool /usr/local/bin/

sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus
```

## Configure Prometheus
```
sudo vim /etc/prometheus/prometheus.yml
```

```
global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
```

```
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

## Create Service
```
sudo vim /etc/systemd/system/prometheus.service
```

```
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \

ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl start prometheus.service
sudo systemctl status prometheus.service
sudo systemctl enable prometheus.service
```

## Open Firewall
```
sudo ufw allow 9090
```

## Install Grafana
```
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/enterprise/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install grafana-enterprise
```

```
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server
sudo systemctl enable grafana-server
```

Check http://localhost:3000
open firewall
admin/admin
Setup reverse proxy in Apache
