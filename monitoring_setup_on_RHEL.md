## Install Node Exporter on Bare Metal

Run these commands directly on your RHEL host machine terminal to install it securely from the enterprise repositories:

```bash
# 1. Install Node Exporter natively
sudo dnf install -y node-exporter

# 2. Enable it to start automatically on boot and launch it now
sudo systemctl enable --now node-exporter

# 3. Verify it is running cleanly
sudo systemctl status node-exporter

```

---

## Configure the Persistent Network & Podman Pod

Before creating files, we need to manually instantiate the network bridge and define the shared Pod workspace where all containers live on the same loopback namespace.

```bash
# 1. Create the dedicated network
podman network create --subnet=172.18.0.0/16 monitoring-testing

# 2. Create the core Pod and map the external accessibility ports
podman pod create --name monitoring-pod --network monitoring-testing \
  -p 127.0.0.1:9090:9090 \
  -p 127.0.0.1:3001:3000 \
  -p 127.0.0.1:9093:9093 \
  -p 127.0.0.1:9115:9115

```

---

## Create the Orchestration File (`compose.monitoring.yml`)

Save the following content as `compose.monitoring.yml`. All host volume definitions contain the `:Z` suffix so that **SELinux** grants file stream access to rootless containers.

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:Z
      - ./node_alerts.yml:/etc/prometheus/node_alerts.yml:Z
      - prometheus_data:/prometheus:Z
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'
      - '--web.external-url=http://localhost:9090/'
      - '--web.enable-remote-write-receiver'
      # Memory diet adjustments for 2GB limit
      - '--storage.tsdb.max-block-duration=2h'
      - '--storage.tsdb.min-block-duration=2h'
      - '--storage.tsdb.retention.time=7d'
    networks:                         
      - monitoring-testing

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana_data:/var/lib/grafana:Z
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASS}
    networks:                         
      - monitoring-testing

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    configs:
      - source: alertmanager_config
        target: /etc/alertmanager/alertmanager.yml
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--web.external-url=http://localhost:9093'
    networks:                         
      - monitoring-testing

  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-exporter
    restart: unless-stopped
    volumes:
      - ./blackbox.yml:/etc/blackbox_exporter/config.yml:Z
    networks:
      - monitoring-testing

networks:
  monitoring-testing:
    external: true

volumes:
  prometheus_data:
  grafana_data:

configs:
  alertmanager_config:
    content: |
      global:
        resolve_timeout: 5m
      route:
        group_by: ['alertname', 'instance']
        group_wait: 30s
        group_interval: 5m
        repeat_interval: 4h
        receiver: 'gmail-notifications'
      receivers:
      - name: 'gmail-notifications'
        email_configs:
        - to: '${GMAIL}'
          from: '${GMAIL}'
          smarthost: 'smtp.gmail.com:587'
          auth_username: '${GMAIL}'
          auth_password: '${GMAIL_PASS}'
          auth_identity: '${GMAIL}'
          send_resolved: true

```

---

## Create the Global Router Engine Config (`prometheus.yml`)

Save this configuration as `prometheus.yml`. Because your monitoring core is deployed inside a Pod namespace, applications reach each other inside the network via `localhost`. To exit the container namespace and scrape the RHEL host machine, it targets the bridge gateway `172.18.0.1`.

```yaml
global:
  scrape_interval: 30s 
  evaluation_interval: 30s

rule_files:
  - "/etc/prometheus/node_alerts.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'

scrape_configs:
  # 1. Prometheus self-auditing
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # 2. Bare-Metal Server Node Metrics (Hits Host OS directly)
  - job_name: 'node'
    static_configs:
      - targets: ['172.18.0.1:9100'] 

  # 3. Target Java Applications outside Podman
  - job_name: 'spring-boot-demo-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['172.18.0.1:8084']

  # 4. Blackbox Active Endpoint Synthetics
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://localhost:3000            # Probes Grafana locally inside the Pod
        - https://ibrahimmoalim.dev        # External Production Target
        - https://api.ibrahimmoalim.dev    # External API Target
        - http://172.18.0.1:8081/api/users   # Internal App Endpoint on Host
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115        # Redirects payload traffic into the Pod's prober engine

```

---

## Create your Rules and Blackbox Alerts (`node_alerts.yml`)

Save this as `node_alerts.yml`. This contains the exact warning limits you had before, along with the automated notification triggers for when Blackbox endpoints fail a connection ping or return bad HTTP statuses.

```yaml
groups:
  - name: node_exporter_alerts
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "The Node Exporter target has been offline for more than 1 minute."

      - alert: HighCpuLoad
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 90
        for: 30s
        labels:
          severity: warning
        annotations:
          summary: "High CPU load on {{ $labels.instance }}"
          description: "CPU usage is at {{ printf \"%.2f\" $value }}% for the last 5 minutes."

      - alert: LowMemory
        expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 10
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Low Memory on {{ $labels.instance }}"
          description: "Available memory has dropped below 10% (Current: {{ printf \"%.2f\" $value }}%)."

  - name: spring_boot_app_alerts
    rules:
      - alert: HikariPoolNearExhaustion
        expr: hikaricp_connections_active{pool="WalletApp-HikariPool"} > 16
        for: 10s
        labels:
          severity: warning
        annotations:
          summary: "HikariCP pool utilizing > 80% capacity"
          description: "Active connections are at {{ $value }} out of a max pool size of 20."

      - alert: HikariPoolPendingThreads
        expr: hikaricp_connections_pending{pool="WalletApp-HikariPool"} > 0
        for: 0s
        labels:
          severity: critical
        annotations:
          summary: "Threads are waiting for database connections!"
          description: "There are currently {{ $value }} threads blocked waiting to get a database connection."

      - alert: HighHttpErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.*"}[1m]) > 1
        for: 15s
        labels:
          severity: critical
        annotations:
          summary: "Spike in Application Errors (5xx status codes)"
          description: "The application is throwing internal server exceptions at a rate of {{ $value }} per second."

  - name: blackbox_exporter_alerts
    rules:
      - alert: BlackboxProbeFailed
        expr: probe_success == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Endpoint Down: {{ $labels.instance }}"
          description: "The Blackbox prober failed to hit target endpoint {{ $labels.instance }} for more than 30 seconds."

      - alert: SSLCertExpiringSoon
        expr: (probe_ssl_earliest_cert_expiry - time()) / 3600 / 24 < 7
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "SSL Certificate Expiring Soon on {{ $labels.instance }}"
          description: "The SSL certificate for {{ $labels.instance }} expires in {{ printf \"%.1f\" $value }} days."

```

---

## Firewall

Since Node Exporter is running on the bare-metal host, RHEL's firewalld will block traffic
coming from Podman's internal network interfaces to host port 9100 unless explicitly
permitted.
Run these commands:
```bash
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --reload
```
check with:
```bash
sudo firewall-cmd --list-all
```

## Deploy Everything

Once these files sit together in your directory, bring the stack online inside your target pod:

```bash
podman-compose -f compose.monitoring.yml up -d

```

### Hot-Reloading Reminder

Whenever you tweak your alert properties inside `node_alerts.yml` or add a new HTTP url route to `prometheus.yml`, remember to execute a reload payload without stopping any application:

```bash
curl -X POST http://localhost:9090/-/reload

```
