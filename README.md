# GCP VM Monitoring with Prometheus and Grafana

This project demonstrates infrastructure monitoring on Google Cloud using **Prometheus, Grafana, Node Exporter, and cAdvisor**. Follow the steps below to deploy the stack on two Compute Engine VMs. The configuration files are maintained in this repository, so you can clone the repo instead of copying configuration examples from this README.

## What you will build

- **monitoring-vm:** Prometheus, Grafana, and cAdvisor
- **target-vm:** Node Exporter for Linux host metrics
- **Prometheus:** collects metrics from itself, Node Exporter, and cAdvisor
- **Grafana:** visualizes collected metrics and supports dashboards and variables
- **Alert rule:** a sample high-CPU rule is provided in the repository

## Repository files

Use the existing files in this repository:

- [prometheus.yml](./prometheus.yml) — scrape targets and alert rule file reference
- [alerts.yml](./alerts.yml) — Prometheus high-CPU alert rule

If you change a file, keep it in the cloned repository and copy the updated file to the location mounted into the container.

## Prerequisites

- A Google Cloud project with Compute Engine enabled
- Permission to create VM instances and VPC firewall rules
- SSH access to both VMs
- Git and Docker installed on both VMs

## 1. Create the GCP VMs

Create two VMs in **us-central1**, zone **us-central1-a**, using Ubuntu 24.04 LTS.

| Setting | monitoring-vm | target-vm |
|---|---|---|
| Role | Prometheus, Grafana, cAdvisor | Node Exporter |
| Machine type | e2-standard-2 | e2-medium |
| Boot disk | Balanced persistent disk, 30 GB | Balanced persistent disk, 20 GB |
| Network tag | monitoring | target |

### Configure firewall access

Create VPC firewall rules for the required traffic:

| Service | Port | Recommended access |
|---|---:|---|
| Prometheus UI | TCP 9090 | Trusted admin IP/CIDR only |
| Grafana UI | TCP 3000 | Trusted admin IP/CIDR only |
| cAdvisor | TCP 8080 | Keep private; do not expose publicly |
| Node Exporter | TCP 9100 | Monitoring VM's internal IP/CIDR only |

Ensure each rule targets the correct network tag. For Node Exporter, use the current internal IP of monitoring-vm as the source. VM addresses can differ from examples, so use the current addresses shown in Compute Engine.

## 2. Install Docker

Run on **both VMs**:

    sudo apt update
    sudo apt upgrade -y
    sudo apt install -y docker.io git
    sudo systemctl enable --now docker
    sudo systemctl status docker --no-pager
    docker --version
    git --version

## 3. Clone this repository on monitoring-vm

SSH into monitoring-vm, then run:

    git clone https://github.com/SnehalShinde11/-PrometheusGrafana-.git
    cd -- -PrometheusGrafana-
    ls -l

The repository should contain prometheus.yml and alerts.yml.

### Update the target address

Open the existing Prometheus file:

    nano prometheus.yml

Update the Node Exporter target to match the **current internal IP address of target-vm**. Save the file. You do not need to create a new configuration from scratch.

## 4. Prepare the configuration and Docker network

On monitoring-vm, copy the repository's existing configuration files into the paths that Prometheus will use:

    sudo mkdir -p /opt/prometheus
    sudo cp prometheus.yml /opt/prometheus/prometheus.yml
    sudo cp alerts.yml /opt/prometheus/alerts.yml

Create a shared Docker network for Prometheus, Grafana, and cAdvisor:

    sudo docker network create monitoring

If the network already exists, inspect it with:

    sudo docker network inspect monitoring

## 5. Deploy Prometheus

Validate the configuration files copied from the repository:

    sudo docker run --rm \
      --entrypoint /bin/promtool \
      -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
      -v /opt/prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro \
      prom/prometheus \
      check config /etc/prometheus/prometheus.yml

Start Prometheus on the shared Docker network and mount both repository configuration files:

    sudo docker run -d \
      --name prometheus \
      --restart unless-stopped \
      --network monitoring \
      -p 9090:9090 \
      -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
      -v /opt/prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro \
      prom/prometheus

Verify the service:

    curl http://localhost:9090/-/healthy
    sudo docker logs prometheus --tail 30

Open http://<MONITORING_VM_EXTERNAL_IP>:9090 and navigate to **Status → Targets** to check scrape health.

## 6. Deploy Node Exporter on target-vm

SSH into target-vm and run:

    sudo docker run -d \
      --name node-exporter \
      --restart unless-stopped \
      --net="host" \
      --pid="host" \
      -v "/:/host:ro,rslave" \
      prom/node-exporter \
      --path.rootfs=/host

Verify metrics on target-vm:

    curl http://localhost:9100/metrics

From monitoring-vm, test connectivity using the target VM's internal IP:

    curl http://<TARGET_VM_INTERNAL_IP>:9100/metrics | head

If the request fails, check the firewall source range, target network tag, internal IP, and Node Exporter container status.

## 7. Deploy cAdvisor on monitoring-vm

Run cAdvisor on the same Docker network as Prometheus:

    sudo docker run -d \
      --name cadvisor \
      --restart unless-stopped \
      --network monitoring \
      -p 8080:8080 \
      -v /:/rootfs:ro \
      -v /var/run:/var/run:ro \
      -v /sys:/sys:ro \
      -v /var/lib/docker/:/var/lib/docker:ro \
      gcr.io/cadvisor/cadvisor:latest

Verify that cAdvisor exposes metrics:

    curl http://localhost:8080/metrics | head
    sudo docker exec prometheus wget -qO- http://cadvisor:8080/metrics | head

## 8. Deploy Grafana

On monitoring-vm, create persistent storage and start Grafana on the shared network:

    sudo docker volume create grafana-storage

    sudo docker run -d \
      --name grafana \
      --restart unless-stopped \
      --network monitoring \
      -p 3000:3000 \
      -v grafana-storage:/var/lib/grafana \
      grafana/grafana

Check its health:

    curl http://localhost:3000/api/health
    sudo docker ps

Open http://<MONITORING_VM_EXTERNAL_IP>:3000. Initial Grafana credentials are commonly admin/admin; change the password when prompted and never keep the default password.

### Add Prometheus as the Grafana data source

In Grafana:

1. Open **Connections → Data sources**.
2. Select **Add data source → Prometheus**.
3. Set the URL to **http://prometheus:9090**.
4. Click **Save & test**.

Use the container name because Grafana and Prometheus are separate containers. The shared Docker network allows Grafana to resolve Prometheus by name.

## 9. Verify monitoring and alerts

### Verify Prometheus targets

Open Prometheus at http://<MONITORING_VM_EXTERNAL_IP>:9090, then select **Status → Targets**. Confirm that the Prometheus, Node Exporter, and cAdvisor targets are **UP**.

Run this PromQL query in Prometheus or Grafana Explore:

    up

A value of 1 means a target is up; 0 means its scrape is failing.

### Verify the alert rule

The sample HighCPUUsage rule is defined in the repository's alerts.yml file and mounted into Prometheus during deployment. Open:

http://<MONITORING_VM_EXTERNAL_IP>:9090/alerts

Prometheus evaluates the rule and shows its state. Notification delivery requires a separately configured receiver such as Alertmanager.

## 10. Create Grafana dashboards

In Grafana, go to **Dashboards → New → New dashboard → Add visualization**, then select Prometheus as the data source.

Example queries for panels:

| Metric | PromQL |
|---|---|
| CPU usage (%) | 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) |
| Memory usage (%) | 100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) |
| Disk read throughput | rate(node_disk_read_bytes_total[5m]) |
| Network receive throughput | rate(node_network_receive_bytes_total[5m]) |

Set the visualization to **Time series**, provide a descriptive title, and save the dashboard. You can also import the community **Node Exporter Full** dashboard using ID **1860**.

### Add a host dropdown variable

To filter panels by host:

1. Open dashboard settings and select **Variables → Add variable**.
2. Name the variable instance and label it Host.
3. Choose **Query** as the type and select your Prometheus data source.
4. Configure the query to return instance label values.
5. Update panel queries to filter on instance=~"$instance".

## Troubleshooting

- **A target is DOWN:** Check the internal IP, exporter status, VPC firewall source range, and network tags.
- **Grafana cannot connect to Prometheus:** Confirm both containers are attached to the monitoring Docker network and the data-source URL is http://prometheus:9090.
- **cAdvisor target is DOWN:** Confirm cAdvisor is running on the shared network and the repository's prometheus.yml uses cadvisor:8080.
- **Configuration validation fails:** Validate the files copied from the repository with promtool check config and check file paths and YAML indentation.
- **Updated configuration is not active:** Copy the updated file from the clone to /opt/prometheus and restart or recreate the Prometheus container as appropriate.

## Security recommendations

- Restrict web UIs with firewall rules and trusted source CIDRs.
- Keep Node Exporter and cAdvisor private; expose only what you need.
- Change the initial Grafana password.
- Never commit credentials, private keys, or service-account keys to the repository.
- Keep the operating system and container images updated.

## Useful links

- [Repository](https://github.com/SnehalShinde11/-PrometheusGrafana-)
- [Prometheus configuration file](./prometheus.yml)
- [Prometheus alert rules](./alerts.yml)
- [Prometheus documentation](https://prometheus.io/docs/)
- [Grafana documentation](https://grafana.com/docs/)
- [Node Exporter](https://github.com/prometheus/node_exporter)
- [cAdvisor](https://github.com/google/cadvisor)

---
**Project:** GCP VM Monitoring with Prometheus, Grafana, Node Exporter, and cAdvisor
