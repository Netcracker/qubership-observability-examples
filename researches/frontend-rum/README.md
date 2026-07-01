# Frontend RUM

## Grafana Faro + Qubership APIHUB

### Overview

To PoC I took Qubership APIHUB UI and integrate Grafana Faro in it UI. No other changes made.

Qubership APIHUB UI changes:

- Branch [https://github.com/Netcracker/qubership-apihub-ui/tree/feature/integrate-grafana-faro](https://github.com/Netcracker/qubership-apihub-ui/tree/feature/integrate-grafana-faro)
- PR [https://github.com/Netcracker/qubership-apihub-ui/pull/278](https://github.com/Netcracker/qubership-apihub-ui/pull/278)

Used Grafana Faro library:

- [https://github.com/grafana/faro-web-sdk](https://github.com/grafana/faro-web-sdk)

For backends to save data used:

- VictoriaLogs - for logs and events
- Grafana Tempo - for traces

### Working Flow

```mermaid
graph TB
    subgraph "Kubernetes"
        subgraph "Browser"
            direction TB
            APIHUB_FE[APIHUB Frontend]
        end
        subgraph "APIHUB UI"
            direction TB
            APIHUB_NGINX[Nginx]
        end

        subgraph "OTEL Backend"
            direction TB
            OTEC[OpenTelemetry Collector]
            VL[VictoriaLogs]
            TEMPO[Tempo]
        end

        subgraph "Visualization"
            direction TB
            GRAFANA[Grafana<br/>Dashboards & Visualization]
        end
    end

    %% Application to OTEC backend communication
    APIHUB_FE -->|send logs and traces| APIHUB_NGINX
    APIHUB_NGINX -->|proxy logs and traces| OTEC
    OTEC -->|Logs and events| VL
    OTEC -->|Traces| TEMPO

    %% Pyroscope to external systems
    VL -->|Query Data| GRAFANA
    TEMPO -->|Query Data| GRAFANA

    %% Styling
    classDef app fill:#e1f5fe
    classDef otec fill:#fff3e0
    classDef backends fill:#f3e5f5
    classDef visualization fill:#e8f5e8

    class APIHUB_FE,APIHUB_NGINX app
    class OTEC otec
    class VL,TEMPO backends
    class GRAFANA visualization
```

### How to display this information?

Logs and events store in VictoriaLogs as logs and can be selected from VictoriaLogs directly:

![victorialogs-logs-events](images/victorialogs-logs-events.png)

Traces (currently only page calls) store in Grafana Tempo and also can be selected directly from Tempo:

![Tempo](images/tempo-traces.png)

Also, in Grafana we can make a dashboard that will show data, graphs:

![APIHUB Portal](images/grafana-dashboard-apihub-portal.png)

![APIHUB Agents](images/grafana-dashboard-apihub-agents.png)

JSON for this dashboard already added in PoC and available in [Grafana Faro VictoriaLogs](apihub-faro-docker-compose/config/grafana/provisioning/dashboards/grafana-faro-victorialogs.json).

### Run PoC

#### Requirements

- docker or podman
- docker compose or podman compose

#### How to run PoC

1. Clone this repository

    ```bash
    git clone git@github.com:Netcracker/qubership-observability-examples.git
    ```

2. Navigate to `researches/frontend-rum/apihub-faro-docker-compose`

    ```bash
    cd researches/frontend-rum/apihub-faro-docker-compose
    ```

3. Run script to generate all ENVs and JWT keys:

    ```bash
    ./generate.sh
    ```

4. Print credentials for Qubership APIHUB UI using the command:

    ```bash
    cat qubership-apihub-backend-config.yaml
    ```

    in the section `zeroDayConfiguration` you can find email and password to login:

    ```yaml
    zeroDayConfiguration:
      adminEmail: <email>
      adminPassword: <password>
    ```

5. Run docker compose

    ```bash
    docker compose up
    ```

    > Note: All env and api keys already filled

6. Login in APIHUB and Grafana to see events

Available UIs:

- APIHUB UI - [http://localhost:8081](http://localhost:8081)
- Grafana - [http://localhost:3000](http://localhost:3000)
- VictoriaLogs - [http://localhost:9428](http://localhost:9428)
- Tempo - [http://localhost:3000](http://localhost:3000) (inside the Grafana, Explore -> Tempo)

Credentials:

- APIHUB - see above how to find
- Grafana - admin / admin
