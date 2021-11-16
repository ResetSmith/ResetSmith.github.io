---
title: "Installing Blackbox"
date: 2021-11-09
hero: 66260.jpg
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Installing the Blackbox exporter for Prometheus
    identifier: monitoring-blackbox
    parent: monitoring
    weight: 30
draft: true
---

Blackbox exporter is a tool to probe remote targets over a variety of protocols. Or from the [Blackbox exporter Github](https://github.com/prometheus/blackbox_exporter) page:
> The blackbox exporter allows blackbox probing of endpoints over HTTP, HTTPS, DNS, TCP and ICMP.

In practice I use it to monitor connectivity to websites. Through the Blackbox exporter I can capture HTTPS response times, SSL certificate statuses, and some additional website metrics. Additionally using the Alertmanager tool it's possible to create alerts based on HTTPS response times, HTTPS status codes, or SSL certificate status.

## Downloade Blackbox Exporter

```
curl -O -L https://github.com/prometheus/blackbox_exporter/releases/download/v0.19.0/blackbox_exporter-0.19.0.linux-amd64.tar.gz
tar xzf blackbox_exporter-*.*.linux-amd64.tar.gz
sudo mv /blackbox_exporter /usr/local/bin/metrics/blackbox_exporter
```

## Make configuration file

```
sudo vim /etc/metrics/blackbox.yml
```
```
modules:
  http_prometheus:
     prober: http
     timeout: 5s
     http:
       valid_http_versions: ["HTTP/1.1", "HTTP/2"]
       method: GET
       fail_if_ssl: false
       fail_if_not_ssl: true
       tls_config:
         insecure_skip_verify: true
  http_2xx:
    prober: http
    http:
     preferred_ip_protocol: ip4
  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
  pop3s_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
      - send: "SSH-2.0-blackbox-ssh-check"
  irc_banner:
    prober: tcp
    tcp:
      query_response:
      - send: "NICK prober"
      - send: "USER prober prober prober :prober"
      - expect: "PING :([^ ]+)"
        send: "PONG ${1}"
      - expect: "^:[^ ]+ 001"
  icmp:
```

## Create a service

```
sudo vim /etc/systemd/system/blackbox_exporter.service
```
```
[Unit]
Description=Blackbox exporter for Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=exporter
Type=simple
ExecStart=/usr/local/bin/metrics/blackbox/blackbox_exporter \
 --config.file=/etc/metrics/blackbox.yml \
 --web.listen-address=":9115"


[Install]
WantedBy=multi-user.target
```

enable service
```
sudo systemctl start blackbox_exporter.service
sudo systemctl status blackbox_exporter.service
sudo systemctl enable blackbox_exporter.service
```

## Open port on firewall
open UFW port on server
```
sudo ufw allow from 137.184.43.210 proto tcp to any port 9115
```

## Create new job in your Prometheus configuration

add target to prometheus.yml file on panopticon server
