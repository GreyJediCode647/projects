# What is Prometheus?
Prometheus is an open-source monitoring and alerting system. It answers questions like “Is my server up?”, “How much CPU is it using?” and “Is disk space running out?”, and it keeps the history so you can see trends over time.

## The key ideas:

* **Pull mode** : Prometheus goes out and scrapes metrics from your systems every 15 seconds. Your system does not push data to it. 
* **Exporters** : Most software does not expose metrics by default. An exporter is a small program that sit next to the system, coillects its stats, and publish them as a plain text on a web page(usually /metrics). **Node Exporter** usually does this for Linux machines: CPU, memory, disk and network.
* **Time-series database** : Every scrapped value is stored with a timestamp and labels, such as `instance="server1"`, so you can query how it changed over time.
* **PromQL** : This is the query language for asking questions of that data, for example "average CPU usage over the last 5 minutes".
* **Alerting** : You can define rules like "CUP above 90% in last 5 minute". Prometheus raises the alert and Alert manager sends and email, Slack and so on.
* **Grafana** : Prometheus has a basic UI, but Grafana is the standard tool for dashboards built on top of it.

```bash
Node Exporter (exposes /metrics) ←── scraped every 15s ── Prometheus (stores + queries) ──→ Grafana (dashboards)
```

## Lets create a enviornment for Prometheus

We will be using Docker for this Lab

**Setp 1: Confirm that Docker is healthy**
```bash
docker --version
docker compose version
docker run --rm hello-world
```
> Why do this: The `hello-world` run proves that Docker can pull images and start container, and also the commnad works

**Step 2: Create the project folder**
```bash
mkdir -p ~/linux-lab/prometheus-lab
cd ~/linux-lab/prometheus-lab
```
> Why do this: Keeping each lab in its own folder means the config, compose file and notes live together.

**Step 3: Write the Prometheus config**
Create `prometheus.yml` 

```bash
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
```
> Why, line by line:
> 1. `scrape_interval: 15s` tells Prometheus to pull fresh data every 15 seconds. This is the pull model in action.
> 2. A `job` is named group of target that do the same thing. We have two: Prometheus montoring itself and the Node Exporter
> 3. `prometheus:9090` and `node-exporter:9100` are container names, not IP addresses. Docker Compose pulls all services on one network with build in DNS, so they find each other by name. 9090 and 9100 are the default ports for each program. 

** Step 4: Write compose file**
Create `docker-compose.yml`

```bash
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro,Z
      - prometheus-data:/prometheus
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped

volumes:
  prometheus-data:
```

> Why piece by piece:
> 1. `ports: "9090:9090"` makes the Prometheus Web UI reachable from your browser at `localhost:9090`.
> 2. The `./prometheus.yml:/etc/prometheus/prometheus.yml:ro,Z` line injects your config into the container `:ro` make it read-only, because Prometheus never needs to change it.`Z` is the SELinux label that lets the container read it.
> 3. `prometheus-data:/prometheus` is a named volume. Prometheus stores its time-series database in `/promerheus`. Without a volume all your history disappears when the container is removed.
> 4. Publishing port 9100 at Node Exporter isn't strictly required, since Prometheus reaches it over the Docker Network. We do it so you can open `localhost:9100/metrics` yourself and see the raw data.
> 5. `restart:unless-stopped` brings the container back after reboot or crash.

**Step 5: Start the stack and verify**
```bash
docker compose up -d
docker compose ps
```
Then open http://localhost:9090 and go to **Status->Targets.**

>Why did we do this: `-d` runs the container in the background. The Target page is your health check: both`prometheus` and `node` should show **UP in green**.
