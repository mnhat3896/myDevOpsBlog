---
title: Monitoring Server Based With Prometheus Container
date: 2021-05-20T12:00:06+09:00
description: How to run a monitoring tool (here is Prometheus) on server-based. Run Prometheus via docker container
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Nhat Le
authorEmoji: 👽
tags:
- cloud
- Monitoring
- Server-Based
categories:
- monitoring
series:
- Monitoring
image: images/post/server-based_monitoring_prometheus/prometheus-logo.png
---

{{< alert theme="warning" dir="ltr" >}}
First, thank you for visiting my blog and read my article, I appreciate it. Secondly, this is also a place to keep my idea, my knowledge. At the same time, I also want to share these understandings, in a way it can help you or you can have another perspective on your problem.

Please read and see from many angles that it is suitable for your situation or not.
If you have any suggestion or you see my mistake in any posts, please help me by contacting me via Telegram, Email or LinkedIn. Thank you :heart:
{{< /alert >}}

The question is how we can monitor our machines that are having issues when we aren't looking??. Yes, That is "continuous monitoring".
But why is Prometheus have to run on Docker?. Why we don't use any other tools that can easily install via package management on the server-based?
First, Prometheus is a popular tooling, easy to config (it uses a YAML file to tell the service what metric you want to scrape), easy to read :wink:. Secondly, I don't want to install the whole package, dependency package,...etc on my machine so here I use Docker. The last is I familiar with Docker and Prometheus, so I want to practise it :blush:.

### NOTE

* Node-Exporter repository: <https://github.com/prometheus/node_exporter>
* Prometheus repository: <https://github.com/prometheus/prometheus>
* Grafana repository: <https://github.com/grafana/grafana>

---

I assume that your server has already intalled Docker. If don't you can check this guide <https://docs.docker.com/engine/install/ubuntu/>.
To implement continuous monitoring, we need service that can scrape those metrics on the target machine, right? This service also has to send metrics that It scraped to a place to store. Then we can use a tool that has UI / human-readable to configure and read the metric.
So the plan is

* Run a scraper, here is I use Node-Exporter
* Set up Prometheus server
* Read metric via UI tool, here I use Grafana

Do you see Service Discovery? `node-exporter`'s place is there.

{{< img src="/images/post/server-based_monitoring_prometheus/prometheus-flow.png" title="<https://github.com/prometheus/prometheus#architecture-overview>" caption="" alt="flow" width="700px" position="center" >}}

## Node-Exporter

This command will open port 9100 and bind-mounted the path's root of your system to the container.
If you need a reference you can check in the `NOTE` section above

>If you start container for host monitoring, specify path.rootfs argument. This argument must match path in bind-mount of host root. The node_exporter will use path.rootfs as prefix to access host filesystem.

```command
docker run -d --name node_exporter -p 9100:9100 -v "/:/host:ro,rslave" /
--pid="host" --path.rootfs=/host prom/node-exporter
```

## Prometheus Step

1.Configuration file

* Create `.yaml` file in your working directory, example at `/your-directory/monitoring/prometheus/prometheus.yml`
* Add this setting to yaml file:

  Take a look at "targets" attribute in yaml file. This endpoint has to match with the endpoint of the node-exporter above. Here I use localhost and port 9100.

  ``` yaml
  # my global config
  global:
    scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
    evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    # scrape_timeout is set to the global default (10s).

  # Alertmanager configuration
  alerting:
    alertmanagers:
    - static_configs:
      - targets:
        # - alertmanager:9093

  # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
  rule_files:
    # - "first_rules.yml"
    # - "second_rules.yml"

  # A scrape configuration containing exactly one endpoint to scrape:
  # Here it's Prometheus itself.
  scrape_configs:
    # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
    - job_name: 'node-export'
      metrics_path: '/metrics'
      static_configs:
      - targets: ['localhost:9100']
  ```

2.Add user 'prometheus'
   `sudo useradd prometheus --system --no-create-home`
3.Give permission with the file's path you create above. To let container can read this configuration file
   `sudo chown prometheus /your-directory/monitoring/prometheus/prometheus.yml`
4.If you want to mount data Prometheus to outside, If don't skip this step (*)

* Create folder to save prometheus's data, ex: `mkdir -p /your-directory/monitoring/prometheus/data`
* Give permission for user 'nobody': `sudo chown nobody /your-directory/monitoring/prometheus/data`
* Reference Dockerfile Prometheus here, if you wonder why use I 'nobody' <https://github.com/prometheus/prometheus/blob/main/Dockerfile>

5.Run Prometheus server (only server, not include scraping service)
  If you haven't created a network, run: `docker network create XXX`. (_Remember to let container prometheus and grafana in the same network_)  

  (*) Remove line `-v /your-directory/monitoring/prometheus/data:/prometheus/data` if you don't want to mount data to outside
  
  ```command
    docker run -d -p 9090:9090 --user nobody --network my-network --name prometheus_server \
    -v /your-directory/monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
    -v /your-directory/monitoring/prometheus/data:/prometheus/data prom/prometheus:v2.26.0 \
    --config.file="/etc/prometheus/prometheus.yml" --storage.tsdb.path="/prometheus/data"
  ```

After you have the scraper and Prometheus server, then you need a UI to read those metric. Actually, you still can go to localhost:9090 to read and explore more. But that is not enough if you need a custom, beautiful display, alert,...etc. Here i will you Grafana for easy setup.

## Grafana Step

  ```command
  docker run --name grafana -p 3000:3000 --network my-network -d grafana/grafana:7.5.2-ubuntu
  ```

## Add Prometheus to Grafana

* Finally go to Grafana hostname you have set up above, and follow this article to add Prometheus to Grafana: <https://prometheus.io/docs/visualization/grafana/>
* After that, you need to write a query or install an existing dashboard written by someone else
  * You can search a dashboard, for example, I searched a dashboard that shows all information about the nodes as we install "node-exporter" above: <https://grafana.com/grafana/dashboards?search=node>
  * Reference how to import a dashboard: <https://grafana.com/docs/grafana/latest/dashboards/export-import/>

That's all. So Prometheus and Grafana like a couple, they usually go together (not always).
You need to monitor your system just need to set up a node-exporter. If you need anything else like a monitoring container or the network or whatever you want, then you just need to find the exporter suitable for your need and use it.

A few exporters I had found

* Container Advisor (cAdvisor): <https://github.com/google/cadvisor>
* Nginx exporter: <https://github.com/nginxinc/nginx-prometheus-exporter#exported-metrics>
* A list of all exporter: <https://prometheus.io/docs/instrumenting/exporters/>
