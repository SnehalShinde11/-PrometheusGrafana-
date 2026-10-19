# Project 5.1 – Implementing Comprehensive Monitoring with Prometheus and Grafana

## 1. Project Overview

This project implements a comprehensive monitoring solution on Google Cloud Platform (GCP) using Prometheus, Grafana, Node Exporter, and cAdvisor.

The objective is to monitor infrastructure and container-level metrics, visualize system performance, and configure alerts to identify potential resource utilization issues.

The implementation uses two Google Compute Engine virtual machines to separate the monitoring stack from the target system.

## 2. Problem Statement

Modern applications require continuous monitoring to maintain availability, performance, and reliability. Without centralized monitoring and visualization, identifying high CPU utilization, memory consumption, and infrastructure issues can be difficult.

This project addresses these challenges by implementing:

- Centralized metric collection using Prometheus.
- Host-level monitoring using Node Exporter.
- Container-level monitoring using cAdvisor.
- Metrics visualization through Grafana dashboards.
- Alert rules to detect high resource utilization.
- Validation of monitoring targets and metric collection.

## 3. Solution Approach

The solution uses a two-VM architecture on GCP.

### Monitoring VM (`monitoring-vm`)

- **Prometheus:** Collects and stores metrics on port `9090`.
- **Grafana:** Visualizes metrics on port `3000`.
- **cAdvisor:** Collects container metrics on port `8080`.

### Target VM (`target-vm`)

- **Node Exporter:** Exposes host-level metrics on port `9100`.

Prometheus scrapes metrics from its own endpoint, the Node Exporter running on the target VM, and cAdvisor running on the monitoring VM. Grafana connects to Prometheus as its data source.

### Monitoring Targets

| Target | Endpoint | Purpose |
|---|---|---|
| Prometheus | `localhost:9090` | Prometheus self-monitoring |
| Node Exporter | `10.128.0.29:9100` | Target VM host metrics |
| cAdvisor | `cadvisor:8080` | Container resource metrics |

## 4. Prerequisites and Dependencies

Before implementing the project, ensure the following are available:

- A Google Cloud Platform project with billing enabled.
- Permissions to create Compute Engine VMs and firewall rules.
- Two Ubuntu 24.04 virtual machines.
- Docker installed on the monitoring VM.
- Node Exporter installed and running on the target VM.
- Prometheus and Grafana deployed on the monitoring VM.
- Access to the required monitoring ports through appropriately restricted firewall rules.
- Basic familiarity with Linux, Docker, PromQL, and Grafana.

## 5. Repository Structure

Clone the repository and inspect its existing configuration files before making changes.

```text
.
├── prometheus.yml
├── alerts.yml
└── README.md
```

### Configuration Responsibilities

- `prometheus.yml` — Prometheus scrape configuration and alert rule file references.
- `alerts.yml` — Alerting rules, including high CPU utilization.
- `README.md` — Project documentation and deployment instructions.

The README references the existing configuration files rather than duplicating their YAML definitions.

## 6. Step-by-Step Implementation

### Step 1: Prepare the GCP Infrastructure

Create and configure two Google Compute Engine virtual machines.

**Monitoring VM**

- Name: `monitoring-vm`
- Machine type: `e2-standard-2`
- Operating system: Ubuntu 24.04
- Purpose: Host Prometheus, Grafana, and cAdvisor.

**Target VM**

- Name: `target-vm`
- Machine type: `e2-medium`
- Operating system: Ubuntu 24.04
- Purpose: Host Node Exporter and expose system metrics.

Verify that both VMs are running and note their internal and external IP addresses.

Configure firewall rules to allow only the required monitoring traffic. Prefer internal IP communication between the instances.

### Step 2: Clone the Repository

Connect to the monitoring VM using SSH.

Install Git if it is not already available:

```bash
sudo apt update
sudo apt install -y git
```

Clone the project repository:

```bash
git clone https://github.com/SnehalShinde11/-PrometheusGrafana-.git
cd -- -PrometheusGrafana-
```

Verify the repository contents:

```bash
ls -la
```

Confirm that the Prometheus configuration and rule files are present at their expected locations.

### Step 3: Install Docker on the Monitoring VM

Check whether Docker is installed:

```bash
docker --version
```

If Docker is not installed, install it using the official Docker installation instructions for Ubuntu.

Enable and start the Docker service:

```bash
sudo systemctl enable --now docker
```

Verify that Docker is running:

```bash
sudo systemctl status docker
```

### Step 4: Deploy Prometheus

Configure Prometheus using the repository's existing `prometheus.yml` and `alerts.yml` files.

Ensure that the configuration references the correct paths and that all required files are mounted into the Prometheus container.

Start or recreate the Prometheus container using the repository's deployment configuration.

Verify the container:

```bash
sudo docker ps
```

Validate the configuration:

```bash
sudo docker exec prometheus \
  promtool check config /etc/prometheus/prometheus.yml
```

### Step 5: Deploy Grafana

Start Grafana using the project's existing deployment configuration.

Verify that the container is running:

```bash
sudo docker ps
```

Open Grafana in a browser:

```text
http://<MONITORING_VM_EXTERNAL_IP>:3000
```

Sign in using the configured credentials.

Navigate to **Connections → Data sources**, add Prometheus, configure the reachable Prometheus URL, and select **Save & test**.

### Step 6: Deploy cAdvisor

Start cAdvisor on the monitoring VM using the project's container deployment configuration.

Verify the container:

```bash
sudo docker ps
```

Check whether its metrics endpoint is responding:

```bash
curl http://localhost:8080/metrics
```

Confirm that the Prometheus scrape configuration uses the correct cAdvisor endpoint, such as `cadvisor:8080` when the containers share a Docker network.

### Step 7: Install Node Exporter on the Target VM

Connect to `target-vm` through SSH.

Install and configure Node Exporter using the project's selected installation method.

Start the service and verify that it is running:

```bash
sudo systemctl status node_exporter
```

Check the metrics endpoint locally:

```bash
curl http://localhost:9100/metrics
```

Confirm that metrics are returned successfully.

### Step 8: Configure Prometheus Scrape Targets

Verify the scrape targets in the existing `prometheus.yml`.

The project uses the following endpoints:

| Target | Endpoint | Purpose |
|---|---|---|
| Prometheus | `localhost:9090` | Self-monitoring |
| Node Exporter | `10.128.0.29:9100` | Target VM metrics |
| cAdvisor | `cadvisor:8080` | Container metrics |

Replace example IP addresses or service names if your actual infrastructure differs.

Reload or restart Prometheus after validating any configuration changes.

### Step 9: Verify Monitoring Targets

Open the Prometheus targets page:

```text
http://<MONITORING_VM_EXTERNAL_IP>:9090/targets
```

Verify that all expected targets show `UP`.

If a target shows `DOWN`, check service status, firewall rules, network connectivity, and endpoint configuration.

### Step 10: Configure Grafana Dashboards

Open the Grafana interface and create dashboards using Prometheus as the data source.

Include visualizations for:

- CPU utilization
- Memory utilization
- System load
- Network traffic
- Container CPU and memory usage

Use suitable PromQL queries to confirm that the panels display data.

### Step 11: Configure and Validate Alerting Rules

Review the existing `alerts.yml` file and confirm that the `HighCPUUsage` rule is configured to trigger when CPU utilization exceeds 80% for five minutes.

Validate the rules:

```bash
sudo docker exec prometheus \
  promtool check rules /etc/prometheus/alerts.yml
```

Check the alert status in the Prometheus **Alerts** page.

If Alertmanager is configured, verify its routing and notification settings separately.

### Step 12: Verify the Complete Monitoring Setup

Perform a final end-to-end check:

1. Confirm that all monitoring containers are running.
2. Verify that Node Exporter is running on the target VM.
3. Confirm that Prometheus targets are `UP`.
4. Confirm that Grafana can query Prometheus.
5. Verify that dashboards display the collected metrics.
6. Confirm that alert rules are loaded and evaluated.
7. Review logs if any component is not functioning correctly.

## 7. Security Considerations

- Restrict access to Prometheus and Grafana using firewall rules.
- Allow exporter access only from the required monitoring sources.
- Avoid exposing internal monitoring endpoints publicly without appropriate controls.
- Use strong credentials for Grafana and other administrative services.
- Do not commit passwords, service account keys, access tokens, or other secrets to GitHub.
- Prefer internal VM communication for metric scraping.

## 8. Troubleshooting

| Issue | Recommended Action |
|---|---|
| Prometheus target is `DOWN` | Check the target service, endpoint, firewall rules, and connectivity. |
| Grafana data source fails | Verify the Prometheus URL and network reachability from Grafana. |
| Node Exporter metrics are missing | Confirm that Node Exporter is running and port `9100` is reachable. |
| cAdvisor metrics are missing | Check the container status, port `8080`, and Prometheus scrape configuration. |
| Prometheus configuration fails | Run `promtool check config` and inspect the configuration and rule-file paths. |
| Alert does not fire | Check the expression, available metrics, threshold, evaluation interval, and pending duration. |
| Dashboard panels show no data | Verify the selected data source, PromQL query, time range, and metric availability. |

## Assignment Details

**Assignment:** Project 5.1 – Implementing Comprehensive Monitoring with Prometheus and Grafana

**Submitted by:** Snehal Shinde
