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

[Prometheus](https://prometheus.io/) is an open-source application created for systems monitoring and alerting. The Prometheus server application can produce limited visualizations of data that it collects and stores from Exporters which send it the data. Prometheus and it's Exporters are good for measuring numerical values over time, things such as: Data Storage/Memory Usage, Network Stats, Process Monitoring.

Prometheus is often paired with the open-source visualization software [Grafana](https://grafana.com) which improves the end-user experience by making queries easier and better looking. In addition to the visualization app, Grafana also makes a log aggregation system called [Loki](https://grafana.com/oss/loki/). Loki is a powerful tool for monitoring activity on remote servers and makes identifiying possible issues or threats much easier.

Through this article series I'll go through setting up a Prometheus server on a remote VPS (In this case a 'droplet' on DigitalOcean). The series will cover Installing Prometheus, Installing Grafana, Installing Node Exporter on both local and remote sources, Installing Black Box locally, Installing Loki, and Finally Installing Promtail on both local and remote sources.

## Getting Started

*For the purposes of this article you will need SSH access to a VPS or local machine running Ubuntu 20.04 LTS*

There are a few different ways to handle getting Prometheus started. The easiest way to get going quickly is to use Prometheus as a Docker image, Promehteus has [instructions on it's website](https://prometheus.io/docs/prometheus/latest/installation/) for doing this if you would like to go that route. For the purposes of this article I'll go through installing Prometheus 'locally', locally in this instance refers to a DigitalOcean VPS. I'll use this setup for the rest of the series so if you do something different your experience may differ slightly.

{{< alert type="info" >}}
**DigitalOcean VPS **\
For this article I am installing Prometheus on a DigitalOcean 2vCPU, 4GB Memory Droplet. Prometheus seems to have relatively low memory usage compared to other monitoring apps I tried (looking at you Elastic), you should be able to follow these directions on any comparable Cloud VPS running Ubuntu 20.04 or the Debian equivalent.
{{< /alert >}}

## Installing Prometheus

To install Prometheus we'll first create a new user on our server to run the app, and create some necessary folders, then grant the correct permissions. From there we'll download Prometheus and move it to the correct location to run from, setup a config file, and then create a service to run the app.

### Creating the User

We'll start with creating a user named 'prometheus'.
```
sudo useradd -rs /bin/false prometheus
```
The '-rs /bin/false' modifier defines this user as a 'system' account, and setting the shell as /bin/false prevents this user from being logged into.

Next let's make the folders Prometheus will need and assign their ownership to the prometheus user and group.
```
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```


### Download Prometheus

With the user made and the folders in place we're ready to download Prometheus. For this we'll navigate to the /tmp folder, download Prometheus, uncompress the tarball, move the files to their correct locations on the server, and then update the permissions to grant access to our 'prometheus' user and group.
```
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.13.1/prometheus-2.13.1.linux-amd64.tar.gz
tar -xzvf prometheus-2.13.1.linux-amd64.tar.gz prometheus/
sudo mv prometheus/prometheus /usr/local/bin/
sudo mv prometheus/promtool /usr/local/bin/

sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus
```

### Configure Prometheus

To make Prometheus work we need to define a configuration file. This file will define some of Prometheus' general settings as well as how, which, and where the Exporters are that Prometheus should listen to.

First create the configuration file.
```
sudo vim /etc/prometheus/prometheus.yml
```


Here is a basic configuration to get us started, it will pull the basic metrics that Prometheus itself collects. Later this can be expanded on to add additional Exporters to monitor. Copy what's below into the prometheus.yml file.
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
The Prometheus configuration file has quite a few options, you can see them all in the [Prometheus configuration documentation](https://prometheus.io/docs/prometheus/latest/configuration/configuration/). We'll start with this simple configuration for now, and add on to this throughout this series.

Next we need to update the permissions of the configuration file for our 'prometheus' user and group.
```
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

### Create Service

With the other pieces in place we're ready to create a service to handle running the Prometheus app. The service will tell Ubuntu which user to use when running Prometheus as well as tell Prometheus where it can find the configuration file and the data to monitor. Create the service file with the following command.
```
sudo touch /etc/systemd/system/prometheus.service
```

Fill in the prometheus.service file with what's below.
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

Now Prometheus should be ready to run. First we'll get the Prometheus service running.
```
sudo systemctl start prometheus.service
```

Then we should check if there are any errors with what we have done so far.
```
sudo systemctl status prometheus.service
```

If the status update returns an error code, you'll need to retrace your steps and fix the issue before continuing. If the status comes back 'active' then we should be set to 'enable' the service. Enabling the service sets it so that it will automatically restart when the server does.
```
sudo systemctl enable prometheus.service
```

We should now have an operating Prometheus app, accessible through it's default of http://localhost:9090.

### Open Firewall
If you are using a firewall app to secure your server (You should be) you will need to open port 9090 to TCP traffic. You can do this with UFW with the following command.
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
### Setup a service for Grafana
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

## Install Node Exporter Locally
### create folder for exporter apps
```
sudo mkdir /usr/local/bin/node_exporter
sudo mkdir /etc/prometheus/node_exporter
sudo useradd -rs /bin/false exporter
sudo chmod -R 777 /usr/local/bin/node_exporter
sudo chmod -R 777 /etc/prometheus/node_exporter
sudo chown -R exporter:exporter /usr/local/bin/node_exporter
sudo chown -R exporter:exporter /etc/prometheus/node_exporter
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

## Add Node Exporter to Prometheus
```
sudo vim /etc/prometheus/prometheus.yml
```
```
- job_name: 'node_exporter'
  scrape_interval: 5s
  static_configs:
    - targets: ['localhost:9100']
```
