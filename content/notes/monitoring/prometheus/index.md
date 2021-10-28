---
title: Prometheus stack notes
weight: 10
menu:
  notes:
    name: Prometheus
    identifier: notes-monitoring-prometheus
    parent: notes-monitoring
    weight: 10
---

{{< note title="Node Exporter installation" >}}
# create folder for exporter apps
sudo mkdir /usr/local/bin/metrics
sudo mkdir /etc/metrics
sudo useradd -rs /bin/false exporter
sudo chmod -R 777 /usr/local/bin/metrics
sudo chmod -R 777 /etc/metrics
sudo chown -R exporter:exporter /usr/local/bin/metrics
sudo chown -R exporter:exporter /etc/metrics

# Install node_exporter
curl -O -L https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
tar xzvf node_exporter-*.*-amd64.tar.gz
sudo mv node_exporter-*.*-amd64 /usr/local/bin/node_exporter

# create service
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

# start and enable service
sudo systemctl start node_exporter.service
sudo systemctl status node_exporter.service
sudo systemctl enable node_exporter.service

# open firewall on server
sudo ufw allow from $PANOPTICON proto tcp to any port 9100
{{< /note >}}


{{< note title="Install Promtail" >}}
# Install Promtail
# from https://github.com/grafana/loki/releases/
curl -O -L https://github.com/grafana/loki/releases/download/v2.3.0/promtail-linux-amd64.zip
sudo apt install unzip
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail

# make conf file
sudo vim /etc/metrics/promtail.yml

server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /etc/metrics/positions.yaml

clients:
  - url: http://137.184.43.210:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: system logs
      host: $HOSTNAME
      __path__: /var/log/*log

# create service
sudo vim /etc/systemd/system/promtail.service

[Unit]
Description=Promtail service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/promtail -config.file /etc/metrics/promtail.yml

[Install]
WantedBy=multi-user.target

# start and enable service
sudo systemctl start promtail.service
sudo sytemctl status promtail.service
sudo systemctl enable promtail.service

# open firewall on LOKI server
sudo ufw allow from $REMOTE_IP proto tcp to any port 3100
{{< /note >}}


{{< note title="Installing Blackbox Exporter" >}}
# Installing BlackBox Exporter
curl -O -L https://github.com/prometheus/blackbox_exporter/releases/download/v0.19.0/blackbox_exporter-0.19.0.linux-amd64.tar.gz
tar xzf blackbox_exporter-*.*.linux-amd64.tar.gz
sudo mv /blackbox_exporter /usr/local/bin/metrics/blackbox_exporter

# make conf file
sudo vim /etc/metrics/blackbox.yml

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

# create blackbox service
sudo vim /etc/systemd/system/blackbox_exporter.service

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

# enable service
sudo systemctl start blackbox_exporter.service
sudo systemctl status blackbox_exporter.service
sudo systemctl enable blackbox_exporter.service

# open UFW port on server
sudo ufw allow from 137.184.43.210 proto tcp to any port 9115

# add target to prometheus.yml file on panopticon server
{{< /note >}}
