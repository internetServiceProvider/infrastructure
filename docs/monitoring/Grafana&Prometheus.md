# Prometheus and Grafana with Docker

# Introduction:

One of the milestones of this project is being able to measure the performance of the network and the different services that comprise it. To achieve this, Prometheus with Grafana will be implemented, tools that perfectly fulfill this purpose.

## What is Prometheus?

Prometheus is an open-source monitoring and alerting toolkit originally developed at SoundCloud and now part of the Cloud Native Computing Foundation (CNCF). It is designed for reliability, scalability, and real-time monitoring of systems and applications.

### Key Features of Prometheus

1. **Time-Series Database**
    - Stores metrics as **time-stamped data**, allowing historical analysis.
    - Efficiently handles **multi-dimensional data** (metrics with labels/tags).
2. **Pull-Based Model**
    - Instead of applications pushing data, Prometheus **scrapes** (pulls) metrics from configured targets at defined intervals.
3. **Powerful Query Language (PromQL)**
    - Allows complex queries for filtering, aggregating, and analyzing metrics.
4. **Alerting (Alertmanager)**
    - Sends notifications (email, Slack, PagerDuty) when specified conditions are met.
5. **Service Discovery**
    - Automatically detects and monitors services in dynamic environments (Kubernetes, Docker, etc.).
6. **Integration-Friendly**
    - Works with **Grafana** for visualization.
    - Supports **exporters** (like `node_exporter`) to monitor systems, databases, and apps.

### **What is Prometheus Used For?**

Prometheus is widely used for:

**Infrastructure Monitoring**

- Tracks CPU, memory, disk, and network usage of servers (via `node_exporter`).

**Application Performance Monitoring (APM)**

- Monitors microservices, APIs, and web apps (supports custom metrics).

**Kubernetes & Cloud-Native Monitoring**

- Native integration with **Kubernetes** for monitoring clusters, pods, and containers.

**Business Metrics & Alerting**

- Tracks user activity, request rates, and error rates, triggering alerts when thresholds are breached.

**Distributed Systems Observability**

- Helps debug performance issues in microservices architectures.

---

## What is Grafana?

Grafana is an open-source visualization and analytics platform used for monitoring and observability. It allows users to create interactive dashboards to display metrics, logs, and traces from various data sources in real time.

Developed by Grafana Labs, it is widely used in DevOps, IT operations, and business intelligence to analyze system performance, detect anomalies, and improve decision-making.

### Key Features of Grafana

1. **Multi-Data Source Support**
    - Works with **Prometheus, InfluxDB, Elasticsearch, MySQL, PostgreSQL, Loki (logs), Tempo (traces), and more**.
2. **Interactive Dashboards**
    - Drag-and-drop panels (graphs, tables, gauges, heatmaps).
    - Dynamic variables for filtering data.
3. **Alerting System**
    - Sends notifications (Slack, Email, PagerDuty) when metrics cross thresholds.
4. **Templating & Annotations**
    - Reusable dashboards with variables (e.g., `$server`, `$application`).
    - Mark events (deployments, outages) on graphs.
5. **Plugins & Extensibility**
    - Supports **official and community-built plugins** (AWS CloudWatch, Zabbix, MongoDB, etc.).
6. **User & Team Management**
    - Role-based access control (RBAC) for dashboards and data sources.

### What is Grafana Used For?

Grafana helps in:

**Infrastructure & Server Monitoring**

- Visualize CPU, memory, disk, and network metrics (from Prometheus + `node_exporter`).

**Application Performance Monitoring (APM)**

- Track request latency, error rates, and throughput (e.g., with Jaeger or Elastic APM).

**Log Analysis**

- Correlate logs (using **Loki**) with metrics for debugging.

**Business Analytics**

- Display sales trends, user activity, or IoT sensor data.

**Cloud & Kubernetes Monitoring**

- Monitor AWS, GCP, Azure, and Kubernetes clusters in real time.

**Incident Response & Troubleshooting**

- Combine metrics, logs, and traces in a single dashboard.

---

# Installing Docker

The following commands will be executed within the cj server. To access this server, follow the procedure below:
Connect with the PC in the i2t lab via ssh:

```bash
ssh [osm@10.244.45.201](mailto:osm@10.244.45.201)
```

Then connect to the cj server via ssh:

```bash
ssh [cj@192.168.88.167](mailto:cj@192.168.88.167)
```

**System update:** This ensures that the system has the latest versions of packages and prevents dependency errors later.

```bash
sudo apt update && sudo apt upgrade -y
```

**Installing necessary dependencies:** Secure HTTPS connections are allowed here and new repositories can be added with add-apt-repository.

```bash
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg
```

**Added the official Docker GPG key:** This adds Docker's digital signature. Ubuntu uses this key to verify that downloaded packages actually come from Docker and are not modified.

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
```

**Add the official Docker repository:** This adds the Docker "catalog" to the system so that apt knows where to look for the official installation.

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/trusted.gpg.d/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**The Docker package list is reloaded:**

```bash
sudo apt update
```

**Install Docker and its dependencies:** Docker (engine), CLI (commands), containerd (which runs the containers), and docker compose (to run several services at once) are installed.

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Verifying that Docker is running:**

```bash
sudo systemctl status docker
```

If it is not active, it is activated with:

```bash
sudo systemctl start docker
```

**Docker initialization always with the system:**

```bash
sudo systemctl enable docker
```

**Using Docker without sudo:** A Docker user is created so that there is no need to always use sudo to run Docker commands.

```bash
sudo usermod -aG docker $USER
```

However, for this to take effect, you must first log out and log back in:

```bash
exit # For local terminals
logout # For ssh connections
```

---

# Grafana Docker Configuration

This part covers the deployment of Grafana using Docker Compose, including the configuration, deployment process, and access methods.

## Docker Compose Configuration

### docker-compose.yml

```yaml
version: "3.8"
services:
  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: unless-stopped
    ports:
     - '3000:3000'
    volumes:
      - grafana-storage:/var/lib/grafana
volumes:
  grafana-storage: {}

```

### Configuration Details

- **Image**: Uses the official `grafana/grafana` Docker image
- **Ports**: Maps host port 3000 to container port 3000 (Grafana web interface)
- **Volumes**:
    - Persistent volume `grafana-storage` mounted at `/var/lib/grafana`
- **Restart Policy**: `unless-stopped` for automatic recovery

### Deployment Process

1. Save the configuration in a `docker-compose.yml` file
2. Run `docker compose up -d` to start Grafana in detached mode

### Access via SSH Tunneling

When deployed on a server behind a gateway PC, use this double tunnel method:

```bash
ssh -L 3000:192.168.88.167:3000 osm@10.244.45.201
```

This command creates a tunnel from your local machine through an intermediate PC to the server hosting Grafana:

- Local port 3000 forwards to 192.168.88.167:3000 (Grafana server)
- Access is routed through 10.244.45.201 (intermediate PC)
- Username: osm

### Post-Installation

- Access Grafana at `http://localhost:3000`
- Default credentials (admin/admin) should be changed or skipped on first login

---

# Prometheus Docker Configuration

This part covers the deployment of Prometheus using Docker Compose, including the configuration, deployment process, and access methods for monitoring servers.

## Docker Compose Configuration

### docker-compose.yml

```yaml
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - 9090:9090
    restart: unless-stopped
    volumes:
      - /home/cj/docker/prometheus/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prom_data:/prometheus
volumes:
  prom_data:
  
```

## Configuration Components

### Service: Prometheus

- **Image**: prom/prometheus - The official Docker image for Prometheus
- **Container Name**: prometheus - Defines the container name for easy reference
- **Command**:
    - -config.file=/etc/prometheus/prometheus.yml - Specifies the configuration file location
- **Ports**:
    - 9090:9090 - Maps host port 9090 to container port 9090, exposing Prometheus's web interface
- **Volumes**:
    - /home/cj/docker/prometheus/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml - Mounts local config file to the container
    - prom_data:/prometheus - Provides persistent storage for Prometheus data
- **Restart Policy**: unless-stopped - Ensures automatic restart unless explicitly stopped

## Prometheus Configuration

### prometheus.yml

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: []
      scheme: http
      timeout: 10s
      api_version: v2

scrape_configs:
  - job_name: prometheus
    honor_timestamps: true
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /metrics
    scheme: http
    static_configs:
      - targets:
          - localhost:9090

  - job_name: servidores
    static_configs:
      - targets:
          - 192.168.88.165:9100
          - 192.168.88.167:9100
```

### Configuration Structure

- **Global Settings**: Defines scrape interval, timeout, and evaluation settings
- **Alerting**: Configuration for alert managers integration
- **Scrape Configs**: Specifies jobs and targets for metrics collection:
    - prometheus job: Self-monitoring on localhost:9090
    - servidores job: Monitoring external servers (Server Faraday and Server CJ)

### Deployment Process

1. Save the configuration in a `docker-compose.yml` file, and `prometheus.yml` file
2. Run `docker compose up -d` to start Grafana in detached mode

### Access Configuration

Since the Prometheus instance is deployed on a remote server with restricted access, a double SSH tunnel is required:

```bash
ssh -L 9090:192.168.88.167:9090 osm@10.244.45.201
```

This command creates a tunnel from your local machine through an intermediate PC to the server hosting Prometheus:

- Local port 9090 forwards to 192.168.88.167:9090 (Prometheus server)
- Access is routed through 10.244.45.201 (intermediate PC)
- **Username**: osm

### Post-Installation

- Access Grafana at `http://localhost:9090`
- Use the interface to view metrics, query data, and manage alerts

---

# Node Exporter:

Node Exporter will be installed on each server and virtual machine where you want to perform measurements. Run the following commands in the root directory of each server:

**Begin by downloading Node Exporter using the wget command:**

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
```

**Extract the contents:**

```bash
tar xvf node_exporter-1.9.1.linux-amd64.tar.gz
```

**Move the Node Exporter Binary:**

```bash
cd node_exporter-1.9.1.linux-amd64
sudo cp node_exporter /usr/local/bin
```

**Then, clean up by removing the downloaded tar file and its directory:**

```bash
cd ..
rm -r ./node_exporter-1.9.1.linux-amd64
rm node_exporter-1.9.1.linux-amd64.tar.gz
```

**Create a Node Exporter User and Assign ownership permissions of the node_exporter binary to this user:**

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

**To ensure that node exporter starts upon server restart, the following is configured:**

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```bash
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Save and exit the editor.

**Allow and start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

**Confirm that the service is running correctly:**

```bash
sudo systemctl status node_exporter.service
```

After successfully completing all of this configuration, the host where the configuration was performed should appear in Prometheus.

---

# Grafana Dashboard Configuration

## Setting Up Prometheus Data Source

### Navigate to Data Sources:

1. Log in to Grafana
2. Go to **Configuration** → **Data Sources**
3. Click **"Add data source"**

### Configure Prometheus Connection:

1. Select **"Prometheus"** as the type
2. Set the URL to `http://localhost:9090` (or your Prometheus server address)
3. Set **"Scrape interval"** to match your Prometheus configuration
4. Click **"Save & Test"** to verify the connection

## Importing Node Exporter Dashboard

### Import Dashboard:

1. Go to **Create** → **Import**
2. Enter dashboard ID **`1860`** (Node Exporter Full)
3. Select your Prometheus data source
4. Click **"Import"**

### Dashboard Configuration:

- The dashboard will automatically detect `node_exporter` metrics
- All panels will populate with system metrics

## Working with Node Metrics

### Viewing Node Data:

- Use the **"Job"** dropdown to select your Prometheus scrape job
- The **"Node"** dropdown will show all discovered hosts
- Metrics will appear in the pre-configured panels

### Key Dashboard Sections:

- **System Overview**: CPU, memory, and load averages
- **Disk Performance**: IOPS, throughput, and space usage
- **Network Traffic**: Bandwidth and error rates
- **Hardware Monitoring**: Temperatures and fan speeds (if available)