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
