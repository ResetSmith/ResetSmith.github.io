---
title: "Installing Node Exporter"
date: 2021-11-09
hero: 66260.jpg
author:
  name: Reset_Smith
  # image: /images/authors/
menu:
  sidebar:
    name: Installing Node Exporter
    identifier: monitoring-node_exporter
    parent: monitoring
    weight: 60
draft: false
---
---

Previously we installed Node Exporter on our Prometheus host. However in practice you will likely be installing Node Exporter on another VPS or other remote machine. The process is largely similar to installing it locally with a few extra steps.

These directions go through the process I use for installing Node Exporter on remote virtual private servers running Ubuntu 20.04. To begin we'll assume you have access to a remote machine/VPS through SSH or other 'terminal' access with 'sudo' permissions.

### Create a system user account

Like we did previously on our Prometheus server we'll need to first create a system user account to run the Node Exporter app. These commands will create the 'exporter' system user, make some necessary folder, and then update the permissions for the new folders to give the exporter user access.
```
sudo useradd -rs /bin/false exporter

sudo mkdir /usr/local/bin/metrics
sudo mkdir /etc/metrics

sudo chown -R exporter:exporter /usr/local/bin/metrics
sudo chown -R exporter:exporter /etc/metrics

sudo chmod -R 777 /usr/local/bin/metrics
sudo chmod -R 777 /etc/metrics

```

### Download Node Exporter

Once we have our user and folders in place we can download the Node Exporter app. Typically I will change directories to the /tmp folder to download the tarball and move to the correct location, but I will leave that to the user to decide where to run these commands from. Regardless we'll first download the Node Exporter tarball file, then uncompress the file, and then move the file to it's working directory.

{{< alert type="info" >}}
**Current Releases**\
When writing this article Node Exporter v1.2.2 was the most recent release. However you should check the Node Exporter releases page for the most recent release. You can find the list on the [Node Exporter github page.](https://github.com/prometheus/node_exporter/releases)
{{< /alert >}}

```
curl -O -L https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
tar xzvf node_exporter-*.*-amd64.tar.gz
sudo mv node_exporter-*.*-amd64 /usr/local/bin/node_exporter
```

### Create a service

Now that we have Node Exporter copied to its working directory we can create the service that will run it.

First let's create the service file itself.

```
sudo touch /etc/systemd/system/node_exporter.service
```

Next we need to fill in the content for the service file. I use vim, but you can use whichever text editor you wish. Fill in the service file with the content below.
```
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

Now that we have the service created we should test run it before enabling it. Run the `systemctl start` command to get the service running, and then check for any errors using the `systemctl status` command.
```
sudo systemctl start node_exporter.service
sudo systemctl status node_exporter.service
```

If the status comes back as 'active' you are ok to move forward enabling the service. Otherwise if you receive any errors during the status check you need to fix them before moving on.
```
sudo systemctl enable node_exporter.service
```

### Open firewall to allow Prometheus traffic

Prometheus checks on Node Exporters by adding them to your list of 'targets'. Prometheus sends requests to the addresses listed in its target list and then the corresponding node replies with the information. So in order to complete this handshake we need to open the firewall on our target server (the server we are installing Node Exporter on now). In this example I'm using the UFW firewall baked into Ubuntu. I use the following command to open the firewall to allow specifically traffic coming from my Prometheus server to check the port that Node Exporter is broadcasting on, all other requests should be blocked. In this example replace the `$PROMETHEUS_SERVER` variable with the IP address of your Prometheus server.
```
sudo ufw allow from $PROMETHEUS_SERVER proto tcp to any port 9100
```

---

### Add target in Prometheus

*To be done on the Prometheus server*

The last thing we must do is add the server we just installed Node Exporter on to our list of Targets for Prometheus. Connect to your Prometheus server and then open up your prometheus.yml file. *If you followed my [Promtheus Installation article] it will be located in /etc/prometheus/prometheus.yml.*

Within the prometheus.yml file scroll down to the 'scrape_configs' section. Within that section there should be a sub-section for `job_name: 'node_exporter'`. If you followed my previous article it will look something like this
```
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']
```

We're going to modify the - targets section just a bit to allow for better readability. Instead of listing the targets within the brackets we're going to make a list. Replace what you currently have for the 'node_exporter' job with the code below, be sure to replace `$NEW_TARGET` with the URL/IP of your new target server.
```
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      -targets:
        # targets listed here need to include the port
        # open port 9100 on target firewall
        - localhost:9100
        - $NEWTARGET:9100
```

Once that's done save your changes, and then restart the Prometheus server to start collecting metrics from Node Exporter. You can use the following command to restart the Prometheus service
```
sudo systemctl restart Prometheus.service
```
