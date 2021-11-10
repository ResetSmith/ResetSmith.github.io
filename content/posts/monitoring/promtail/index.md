---
title: "Installing PromTail"
date: 2021-08-17T10:03:55-07:00
hero: development2.jpg
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Installing Promtail
    identifier: monitoring-promtail
    parent: monitoring
    weight: 50
draft: true
---

Promtail is an app for transferring logs from one server to another. For our use we are transferring logs from our many different VPS to our centralized Loki server.

## Installing Promtail

Begin by downloading the promtail app and installing it
```
curl -O -L https://github.com/grafana/loki/releases/download/v2.3.0/promtail-linux-amd64.zip
sudo apt install unzip
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
```

### make conf file

Make a configuration file for Promtail
```
sudo vim /etc/metrics/promtail.yml
```

```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /etc/metrics/positions.yaml

clients:
  - url: http://$METRIC_SERVER_IP:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: system logs
      host: $HOSTNAME
      __path__: /var/log/*log
```

### create service

We need to create the service that runs Promtail
```
sudo vim /etc/systemd/system/promtail.service
```

```
[Unit]
Description=Promtail service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/promtail -config.file /etc/metrics/promtail.yml

[Install]
WantedBy=multi-user.target
```

### start and enable service

Now we will start the service and enable it so that it starts automatically on restarts
```
sudo systemctl start promtail.service
sudo sytemctl status promtail.service
sudo systemctl enable promtail.service
```

### open firewall on LOKI server

Finally we will need to open the firewall on the LOKI server that collects the logs
```
sudo ufw allow from $REMOTE_IP proto tcp to any port 3100
```

*Instructions adapted from https://github.com/grafana/loki/releases/*
