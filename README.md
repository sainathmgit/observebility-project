# observebility-project

## Overview

This project automates the installation and setup of a monitoring stack with Prometheus, Grafana, Loki, Alloy, Node Exporter, and a web server running a Flask app.

It is designed for a 4-instance deployment:
- `grafana` instance
- `prometheus` instance
- `loki` instance
- `web server` instance

The provided shell scripts perform the installation and configuration for each service.

## What each script does

- `prometheus-setup.sh` — installs Prometheus v3.5.0, creates a Prometheus user, configures systemd, and starts the Prometheus service.
- `grafana-setup.sh` — installs Grafana Enterprise v12.2.1 from the Grafana Debian package and starts the Grafana service.
- `lokisetup.sh` — installs Loki v3.5.7, creates configuration and systemd service, opens port `3100`, and verifies the Loki endpoint.
- `webnode_setup.sh` — configures the web server instance with Node Exporter, installs the Titan app, configures Alloy, and sets up load/log generation.
- `load.sh` — continuously generates light CPU/memory/disk load for Prometheus metrics.
- `generate_multi_logs.sh` — writes random INFO/WARNING/ERROR log lines into `/var/log/titan/*.log` for Loki testing.

## Instance topology

The intended topology is:

1. `prometheus` instance
   - Prometheus service on port `9090`
2. `grafana` instance
   - Grafana UI on port `3000`
3. `loki` instance
   - Loki API on port `3100`
4. `web server` instance
   - Flask web app + Node Exporter + Alloy collector
   - Node Exporter on port `9100`
   - Alloy listener on port `12345`
   - Local app metrics endpoints expected on `5000` and `8080`

## Setup and installation steps

### 1. Create the instances

Create four Linux instances and name them clearly:
- `grafana`
- `prometheus`
- `loki`
- `webserver`

### 2. Open security groups / firewall rules

Allow communication between the instances on these ports:

- `9090` — Prometheus UI/API
- `3000` — Grafana UI
- `3100` — Loki API
- `9100` — Node Exporter metrics
- `12345` — Alloy service (web node collector)
- `22` — SSH access

If the web server exposes app metrics on `5000` or `8080`, allow those ports too if Prometheus/Alloy need direct access.

### 3. Run the setup scripts

#### Prometheus instance

On the `prometheus` instance:
```bash
sudo bash prometheus-setup.sh
```

This does:
- set hostname to `prometheus`
- download Prometheus v3.5.0
- install Prometheus system user
- create `/etc/prometheus` and `/var/lib/prometheus`
- configure systemd service
- start Prometheus

#### Grafana instance

On the `grafana` instance:
```bash
sudo bash grafana-setup.sh
```

This does:
- update the package list
- install Grafana Enterprise v12.2.1
- enable and start `grafana-server`

After installation, access Grafana at:

```text
http://<GrafanaIP>:3000
```

Default login is usually `admin` / `admin` unless changed in Grafana itself.

#### Loki instance

On the `loki` instance:
```bash
sudo bash lokisetup.sh
```

This does:
- install Loki v3.5.7
- create `/etc/loki/config.yml`
- create `/var/lib/loki` storage directories
- set up Loki as a systemd service
- open port `3100`
- verify the Loki ready endpoint

#### Web server instance

On the `web server` instance:
```bash
sudo bash webnode_setup.sh
```

This does:
- set hostname to `web01`
- install utilities and `stress-ng`
- install Prometheus Node Exporter and configure it as `node.service`
- clone the Titan app repository and install Python dependencies
- configure and start the Titan Flask app as `titan.service`
- install Alloy and configure it to remote-write metrics to Prometheus and logs to Loki
- download and run load/log generation scripts
- configure firewall with `ufw`

> Note: `webnode_setup.sh` writes Alloy config with placeholder addresses `http://PrometheusIP:9090` and `http://LokiIP:3100`. Replace those placeholders with the actual private IPs of the `prometheus` and `loki` instances before starting Alloy.

### 4. Verify service connectivity

To confirm the stack is reachable:

- Prometheus: `http://<PrometheusIP>:9090`
- Grafana: `http://<GrafanaIP>:3000`
- Loki: `http://<LokiIP>:3100`
- Node Exporter: `http://<WebServerIP>:9100`
- Alloy: `http://<WebServerIP>:12345`
- Flask app: `http://<WebServerIP>` (app runs as a systemd service)

## Configure data sources in Grafana

1. Log in to Grafana.
2. Add a Prometheus data source with URL:
   - `http://<PrometheusIP>:9090`
3. Add a Loki data source with URL:
   - `http://<LokiIP>:3100`
4. Create dashboards using Prometheus metrics and Loki logs.

## Web server monitoring details

The `webnode_setup.sh` script installs:
- Node Exporter at `/var/lib/node/node_exporter`
- Titan Flask app under `/opt/titan`
- Alloy config at `/etc/alloy/config.alloy`
- load generator scripts in `/usr/local/bin/load.sh` and `/usr/local/bin/generate_multi_logs.sh`

Log files generated for Loki are written to:
- `/var/log/titan/app1.log`
- `/var/log/titan/app2.log`
- `/var/log/titan/app3.log`

## Important notes

- The scripts assume an Ubuntu/Debian environment.
- The `webnode_setup.sh` script installs `alloy` from Grafana APT repository and configures it for remote write.
- If you use AWS security groups, ensure each instance can talk to the others on the required ports.
- If the web server app listens on a different port than expected, update the Alloy scrape target and firewall rules accordingly.

## Useful commands

- Check Prometheus status:
  ```bash
  sudo systemctl status prometheus
  ```
- Check Grafana status:
  ```bash
  sudo systemctl status grafana-server
  ```
- Check Loki status:
  ```bash
  sudo systemctl status loki
  ```
- Check Node Exporter status:
  ```bash
  sudo systemctl status node
  ```
- Check Titan app status:
  ```bash
  sudo systemctl status titan
  ```

## Summary

This repository includes all setup scripts needed to install and configure a simple monitoring solution across four instances. Use the shell scripts in the order above and verify network access between the instances for the stack to work correctly.

## About the tools used

- **Prometheus**: an open-source monitoring and alerting system that scrapes metrics from targets, stores them in a time series database, and provides a query language for analysis.
- **Grafana**: a visualization and dashboard platform used to display Prometheus metrics, Loki logs, and other observability data through reusable panels.
- **Loki**: a log aggregation system optimized for storing and querying logs with labels, designed to work closely with Grafana for log search and dashboards.
- **Alloy**: a Grafana data collector agent used here to scrape local metrics and log files on the web server and forward them to Prometheus and Loki.
- **Node Exporter**: a Prometheus exporter that exposes host-level metrics such as CPU, memory, disk, and network usage from the web server.
- **Flask/Titan app**: a small web application deployed on the web server to generate app metrics and serve as a sample service for monitoring.
- **stress-ng / load scripts**: lightweight load generators (`load.sh` and `generate_multi_logs.sh`) used to produce CPU/disk load and log events for Prometheus and Loki testing.