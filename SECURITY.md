# Security Monitoring

This repository contains the security monitoring and auditing module designed for the SA-FLaaS platform.

This project implements a **multi-layered security auditing system** to address the critical threats of **Tampering (Model Poisoning)** and **Repudiation** in Federated Learning environments. It aligns with the traceability requirements of the **EU AI Act** for high-risk AI systems.

The system utilizes a **Sidecar Architecture** to provide two independent layers of verification:
1.  **Internal Ledger (Security Module):** A cryptographically verifiable, HMAC-chained log of client-reported actions (e.g., `training_started`, `training_ended`).
2.  **External Monitor (Sidecar):** An independent filesystem monitor using a hybrid engine (Watchdog + Polling) to reliably detect file modifications, deletions, and log truncation attempts in real-time.

These two streams are correlated in a centralized dashboard to detect anomalies (e.g., a client claiming to train without accessing data).

---

## Project Structure

* `MVD/src/monitoring.py`: The Python security module (Internal Ledger) integrated into the Client Application.
* `filesystem-monitor/`: Contains the Docker build context and `security_monitor.py` script for the independent monitor.
* `monitoring-stack/`: Contains the Docker Compose configuration for the visualization stack (Loki, Promtail, Grafana).

---


## System Architecture

The system utilizes a **Sidecar Architecture** to provide independent verification of client activities. 

![alt text](diag-1.png)


## Prerequisites 

### 1.Install Docker Desktop

Docker allows us to run the monitoring tools without installing complex software directly on your computer.

Windows / WSL2: Download and install Docker Desktop.

Important for WSL Users: Do not install Docker inside your Ubuntu terminal. Install Docker Desktop on Windows, and ensure "Use the WSL 2 based engine" is checked in the Settings > General menu.

Mac: Download and install Docker Desktop.

Linux (Native): Install Docker Engine and the Docker Compose Plugin using your package manager (e.g., apt or yum).


## Installation of new python dependencies

```bash
pip install watchdog

```

## Integration Guide

### Part 1: Client Application Integration

To enable the internal cryptographic ledger, the client application uses a wrapper function.

1.  **Import the Wrapper:**
    In your main client script (e.g., `clientApp.py`), import the logging helper:
    ```python
    from MVD.dashboard.clientLogging import send_client_monitoring_events
    ```

2.  **Log Critical Events:**
    Add logging calls at the start and end of core functions (`train`, `evaluation`).

    ```python
    @app.train(action='simulation')
    def train(msg: Message, ctx: Context):
        # ... initialization ...
    
        # SECURITY Log Start
        send_client_monitoring_events(
            event_name="training_started", 
            client_id=str(app.node_id), 
            extra_data={
                "message_type": msg.metadata.message_type,
                "run_id": str(ctx.run_id) 
            }
        )
    
        # ... Run training (app._run_training) ...
    
        # SECURITY Log End
        send_client_monitoring_events(
            event_name="training_ended", 
            client_id=str(app.node_id), 
            extra_data={"status": "success", "run_id": str(ctx.run_id)}
        )
        
        return ...
    
    @app.evaluate(action='simulation')
    def evaluate(msg: Message, ctx: Context) -> Message:
        # SECURITY Log Start
        send_client_monitoring_events(
            event_name="evaluation_started", 
            client_id=str(app.node_id),
            extra_data={"message_type": msg.message_type}
        )
        
        # ... evaluation logic ...
    
        # SECURITY Log End
        send_client_monitoring_events(
            event_name="evaluation_ended", 
            client_id=str(app.node_id), 
            extra_data={"status": "completed"}
        )
    
        return ...
    ```


### Part 2: Monitoring Stack Configuration

The Security Monitor is deployed as part of the `monitoring-stack` to provide global oversight of the project files.

**File Location:** `monitoring-stack/docker-compose.yml`

Ensure your configuration includes the `security` service and correctly mounts the project root:

```yaml
version: '3.8'

services:
  loki:
    image: grafana/loki:2.9.2
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - monitoring-net

  promtail:
    image: grafana/promtail:2.9.2
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yml
      - ../logs:/var/log/sa-flaas
    command: -config.file=/etc/promtail/config.yml
    networks:
      - monitoring-net

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - ./datasources.yaml:/etc/grafana/provisioning/datasources/ds.yaml
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_AUTH_ANONYMOUS_ENABLED=true
    depends_on:
      - loki
    networks:
      - monitoring-net

  # SECURITY MONITOR
  security:
    build:
      context: ../filesystem-monitor
      dockerfile: Dockerfile
    # Uses host User ID for secure permission handling
    user: "${UID}:${GID}"
    # Arguments passed explicitly to the script
    command: >
      python -u security_monitor.py 
      --watch-dir /project_root 
      --log-file /var/log/sa-flaas/filesystem.log
    volumes:
      - ../:/project_root:ro
      - ../logs:/var/log/sa-flaas
    networks:
      - monitoring-net

volumes:
  grafana_data:

networks:
  monitoring-net:
 ```


## 2.  **Configure Shared Volumes:**
    Ensure your main client service also mounts the `sa-flaas-logs` volume so it can write the `app.log`.

### Note on Linux vs. Non-Linux Environments (The "Poller" Logic)

The Hybrid Monitoring architecture (Watchdog + Poller) was designed to address cross-platform compatibility issues inherent in Docker containers:

* **Non-Linux Hosts (Windows/macOS):** The Docker virtualization layer for file systems (bind-mounts) often fails to propagate `IN_DELETE` events from the host to the container. In these environments, the **Poller is critical** for detecting file deletions.
* **Native Linux Hosts:** On native Linux deployments (e.g., production servers), the standard kernel `inotify` events propagate correctly through bind-mounts.
    * *Optimization:* In strict Linux-only environments, the Poller frequency can be reduced or the feature disabled to save resources.
    * *Recommendation:* We recommend keeping the Poller enabled even on Linux as a **Defense in Depth** mechanism (Redundancy), ensuring audit integrity even if the event listener is flooded or dropped.

---

### Part 3: Visualization Stack

To view the security dashboard, deploy the monitoring backend.

1.  Navigate to the `monitoring-stack` directory:
    ```bash
    cd monitoring-stack
    ```
2. Configure Permissions (Required for Rootless Mode): Generate a .env file to map your host User ID to the container. This prevents "Permission Denied" errors in the log files.
    ```bash
    echo "UID=$(id -u)" > .env
    echo "GID=$(id -g)" >> .env
    ```
3.  Start the stack:
    ```bash
    docker-compose up -d
    ```
4.  Access Grafana at **http://localhost:3000** (Default login: `admin` / `admin`).
    Import the Security Dashboard

    In Grafana, go to Dashboards (icon on the left) -> New -> Import.

    Click "Upload dashboard JSON file".

    Select the file monitoring-stack/saf-dashboard.json from this repository.

    Click import

---

##  Dashboard Features

The Grafana dashboard provides real-time visibility into the system's integrity:

* **Security Correlation Timeline:** Visualizes the temporal alignment between client claims (Blue bars) and sidecar observations (Orange bars). Perfect overlap confirms valid training behavior.
* **Session Information:** Identifies the specific User, PID, and Script responsible for the current simulation run.
* **Client Audit Trail:** Displays the parsed, HMAC-verified log entries from the internal ledger.
* **Sidecar Observations:** Displays the raw file access events captured by the sidecar.

---

## Configuration

The system uses **Command Line Arguments** for configuration, ensuring flexibility across different environments. These are passed to the `security_monitor.py` script via the `command` field in `docker-compose.yml`.

**Available Arguments:**
* `--watch-dir`: The directory to monitor (e.g., `/project_root`).
* `--log-file`: The path to the output log file (e.g., `/var/log/sa-flaas/filesystem.log`).

## Appendix: Multi-Machine Deployment

Note: Follow these steps ONLY if you are deploying the Server and Clients on different physical computers.

    Server Machine: Run docker compose up normally to start Loki and Grafana.

    Client Machines:

        Edit monitoring-stack/promtail-config.yaml.

        Change the URL from localhost to the IP address of your Server Machine: url: http://<SERVER_IP>:3100/loki/api/v1/push

        Run the security container as a standalone sidecar on this machine.

