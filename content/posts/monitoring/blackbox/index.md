---
title: "Installing Blackbox"
date: 2021-11-16
hero: 504_Error_Gateway_Timeout-rafiki.jpg
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Installing the Blackbox exporter for Prometheus
    identifier: monitoring-blackbox
    parent: monitoring
    weight: 30
draft: false
---

Blackbox exporter is a tool to probe remote targets over a variety of protocols. Or from the [Blackbox exporter Github](https://github.com/prometheus/blackbox_exporter) page:

> The blackbox exporter allows blackbox probing of endpoints over HTTP, HTTPS, DNS, TCP and ICMP.

In practice I use it to monitor connectivity to websites. Through the Blackbox exporter I can capture HTTPS response times, SSL certificate statuses, and some additional website metrics. Additionally using the Alertmanager tool it's possible to create alerts based on HTTPS response times, HTTPS status codes, or SSL certificate status.

Blackbox exporter can be run from just about any remote machine. In the example we're working on in this article Blackbox exporter is using 'GET' requests to pull get HTTP information from remote targets. For this example I am installing Blackbox exporter on the same machine as I have installed Prometheus previously, however it could also be installed on yet another VPS and still report back to the central Prometheus server. It's important to recognize that the results you receive from Blackbox exporter are going to be relative to where you install it. Again, in this example I am installing Blackbox on a DigitalOcean VPS (the Prometheus server) and using it monitor other DigitalOcean VPS (my other websites), so the results I get for load times, HTTPS responses, etc., may not necessarily reflect those of my end users who are not in a DigitalOcean data center.

## Download Blackbox Exporter

For this installation of Blackbox exporter I will not be creating a new user, instead we will be using the same 'exporter' user we created previously. Otherwise, like before I'll leave it up to you on where do to run these commands from, but typically I like to do my downloads to the /tmp folder. Regardless of where you start, we'll first download the Blackbox exporter app, then uncompress it, and then move the app files to their working directory.

```
curl -O -L https://github.com/prometheus/blackbox_exporter/releases/download/v0.19.0/blackbox_exporter-0.19.0.linux-amd64.tar.gz
tar xzf blackbox_exporter-*.*.linux-amd64.tar.gz
sudo mv /blackbox_exporter /usr/local/bin/metrics/blackbox_exporter
```

## Make configuration file

Next we need to create a configuration file for Blackbox exporter. Within the configuration file you can set the parameters for Blackbox that determine which protocols are used. In this example we will check our target websites every 5 seconds, using 'GET' to pull HTTP statuses, including SSL status, response times, and current HTTP version. First create blackbox.yml
```
sudo touch /etc/blackbox/blackbox.yml
```
Then fill in the .yml file with the content below
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
```

## Create a service

Now we will create a service to run the Blackbox app and then 'enable' it so it will run automatically on startup. First we create the service file

```
sudo touch /etc/systemd/system/blackbox_exporter.service
```

Then we fill it in with the following
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

From here we are ready to 'enable' the service. But first we should start the service manually and then check our configuration for any errors using the following commands.
```
sudo systemctl start blackbox_exporter.service
sudo systemctl status blackbox_exporter.service
```

If the status returns 'OK' then we should go ahead and enable the service
```
sudo systemctl enable blackbox_exporter.service
```

## Create new job in your Prometheus configuration

Now that we have Blackbox running, we need to configure Prometheus to pull the data from the Blackbox exporter. Below is an example job that you should copy and add to the end of your 'prometheus.yml' configuration file we made previously. You can replace the example targets with the websites you wish to monitor. If you have installed Blackbox exporter on a different host than Prometheus, you would define the address for your Blackbox server at the bottom in the 'replacement:' line.
```
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
    - targets:
      # add/remove websites to be observed by black box here
      # each target on an individual line, spacing and dashes formatting is important
      - http://localhost            # Targets self
      - http://example.com          # http website
      - https://example.com         # https website
      - https://example.com:9000    # https website at port 9000
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
```

After this you should restart the Prometheus service to pull in the Blackbox data.
```
sudo systemctl restart prometheus.service
```
