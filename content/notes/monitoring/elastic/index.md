---
title: Elastic stack notes
weight: 10
menu:
  notes:
    name: Elastic
    identifier: notes-monitoring-elastic
    parent: notes-monitoring
    weight: 20
---

{{< note title="Installing ElasticSearch" >}}
install java
```
sudo apt install default-jre default-jdk
```

Install Elasticsearch  
get public GPG key
```
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
```
add elastic to sources list
```
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
```
check for updates
```
sudo apt update
```
install elasticsearch
```
sudo apt install elasticsearch
```
edit elasticsearch.yml config file
```
sudo vim /etc/elasticsearch/elasticsearch.yml
```
update network.host to read localhost
```
network.host: localhost
```
start elasticsearch service and then enable
```
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
```
IF the service fails to start due to 'Timeout' you may need to increase the timeout time

Make a folder
```
sudo mkdir /etc/systemd/system/elasticsearch.service.d
```
Now create a file and add the content to increase timeout time to 900 seconds
```
echo -e "[Service]\nTimeoutStartSec=900" | sudo tee /etc/systemd/system/elasticsearch.service.d/startup-timeout.conf
```
test that its working
```
curl -X GET "localhost:9200"
```
{{< /note >}}

{{< note title="Installing Kibana" >}}
Install Kibana
```
sudo apt install kibana
```
start and enable
```
sudo systemctl enable kibana
sudo systemctl start kibana
```
Install and configure Apache w/ ACL
{{< /note >}}

{{< note title="Installing Logstash" >}}
Install Logstash
```
sudo apt install logstash
```
create files to process input
```
sudo vim /etc/logstash/conf.d/02-beats-input.conf
```
copy below into that file
```
input {
  beats {
    port => 5044
  }
}
```
create output file
```
sudo vim /etc/logstash/conf.d/30-elasticsearch-output.conf
```
copy below into that file
```
output {
  if [@metadata][pipeline] {
    elasticsearch {
    hosts => ["localhost:9200"]
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    pipeline => "%{[@metadata][pipeline]}"
    }
  } else {
    elasticsearch {
    hosts => ["localhost:9200"]
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    }
  }
}
```
test logstash
```
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```
need to see - `Config Validation Result: OK. Exiting Logstash`  

if ok, start and enable
```
sudo systemctl start logstash
sudo systemctl enable logstash
```
{{< /note >}}

{{< note title="Installing Filebeat" >}}
Installing Filebeat
```
sudo apt install filebeat
```
configure yml file
```
sudo vim /etc/filebeat/filebeat.yml
```
from here we need to disable the elasticsearch output
```
...
#output.elasticsearch:
  # Array of hosts to connect to.
  #hosts: ["localhost:9200"]
...
```
and then enable the logstash output
```
output.logstash:
  # The Logstash hosts
  hosts: ["localhost:5044"]
```
list filebeat modules
```
sudo filebeat modules list
```
enable filebeat modules
```
sudo filebeat modules enable system apache
```
create filebeat ingest pipeline
```
sudo filebeat setup --pipelines --modules system
```
load template index into elasticsearch
```
sudo filebeat setup --index-management -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["localhost:9200"]'
```
need to see - `Index setup finished.`  

load filebeat dashboards into Kibana
```
sudo filebeat setup -E output.logstash.enabled=false -E output.elasticsearch.hosts=['localhost:9200'] -E setup.kibana.host=localhost:5601
```
start and enable filebeat
```
sudo systemctl start filebeat
sudo systemctl enable filebeat
```
{{< /note >}}

{{< note title="Adding Filebeat to other sources" >}}
Adding filebeat to other sources
```
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update
sudo apt install filebeat
```
```
vim /etc/filebeat/filebeat.yml
```
hashout the Elasticsearch output section and then fill in the logstash output section with the correct remote IP
```
sudo filebeat modules enable system apache
sudo systemctl start filebeat.service
sudo systemctl enable filebeat.service
```
{{< /note >}}
