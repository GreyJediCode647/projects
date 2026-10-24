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

## Lets add a webserver to this enviornment

There are two different ways to monitor a webserver, they are Blackbox monitoring and Whitebox monitoring.

**1. Blackbox monitoring "Does it work from outside?"**
The Blackbox Exporter acts like a user. It sends a HTTP request to your web server and reports whether it got a good reponse and how long it took. It does not know or care about what is inside the server.
> Why: This is the closest thing to real health monitoring. A server can be running while the site is broken, and only a external probe catches that. 

**2. Whitebox monitoring "What is haappening inside?"**
The NGINXPrometheus Exporter reads nginx's internal status page and publish metrics like active connections and total request handels. 
> Why: This tells you how busy the server is and helps you spot trends, but it can't tell you wheather a user would see a working page. Together the two approches cover both questions. 

**Step 1: Create a Web server (Nginx)**
Create a nginx.conf file
```bash
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location = /stub_status {
        stub_status;
    }
}
```
> This serves the default nginx welcome page and exposes the internal `stub_status` page that the expoter reads. For the lab it is fine, but in production you would restrict `/stub_status` so the public can't see it. 

**Step 2: Make changes to `docker-compose.yml` (under `services:`):
```bash
  webserver:
    image: nginx:stable
    container_name: webserver
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro,Z
    restart: unless-stopped

  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:latest
    container_name: nginx-exporter
    command:
      - --nginx.scrape-uri=http://webserver:80/stub_status
    depends_on:
      - webserver
    restart: unless-stopped

  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-exporter
    restart: unless-stopped
```
**Step3: Make changes to `prometheus.yml` (under `scrape_configs:`):**
```bash
  - job_name: "nginx"
    static_configs:
      - targets: ["nginx-exporter:9113"]

  - job_name: "blackbox-http"
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://webserver:80
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```
> The rule to remember for YAML is that siblings must line up exactly. Everything at the same level, such as the service names under `services:`, needs the same number of spaces, and children go deeper by a consistent amount, usually 2.
>  When you get this error again, comparing the indentation of neighbouring lines is the first thing to check.
