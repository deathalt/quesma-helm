# Quesma Helm Chart

Helm chart for deploying **Quesma** (an Elasticsearch API–compatible gateway) on Kubernetes.

This chart can connect Quesma to external datastores or spin up bundled ones via **dependencies**:
- `opensearch` – search cluster (compatible backend)
- `opensearch-dashboards` – OpenSearch web UI

> By default, both dependencies are **disabled**. Enable them in `values.yaml` when needed.

---

## Table of Contents

- [Architecture](#architecture)
- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Dependencies (OpenSearch & Dashboards)](#dependencies-opensearch--dashboards)
- [Quesma Configuration](#quesma-configuration)
- [Ingress](#ingress)
- [Autoscaling](#autoscaling)
- [Resources and Probes](#resources-and-probes)
- [Common Scenarios](#common-scenarios)
- [Upgrade / Uninstall](#upgrade--uninstall)
- [Security Notes](#security-notes)
- [Key Parameters](#key-parameters)
- [Troubleshooting](#troubleshooting)

---

## Architecture

Quesma is deployed as a **Deployment** with a **ClusterIP Service** listening on port `8080`.

Data flow is organized through:

- **frontendConnectors** — emulate Elasticsearch API entry points:
  - `elasticsearch-fe-ingest` (ingest)
  - `elasticsearch-fe-query` (search/query)
- **backendConnectors** — real data sources:
  - OpenSearch/Elasticsearch (indices/metadata, etc.)
  - SQL storages (`clickhouse`, `hydrolix`, `clickhouse-os`)

Request routing and transformations are handled by **processors** and **pipelines**.

---

## Requirements

- **Kubernetes** 1.23+
- **Helm** 3.9+
- Access to container registry hosting `quesma/quesma`
- (Optional) External OpenSearch/Elasticsearch if not using the bundled dependency

---

## Quick Start

1) From the chart directory, pull chart dependencies:

```bash
helm dependency update .
```

2) Create a minimal `values.yaml`:

```yaml
image:
  tag: "v1.0.0" # set your desired image tag

config:
  quesmaConfigurationYaml:
    licenseKey: "PASTE_LICENSE_HERE"
    logging:
      level: "info"
      fileLogging: true
    frontendConnectors:
      - name: elastic-ingest
        type: elasticsearch-fe-ingest
        config:
          listenPort: 8080
          disableAuth: true
      - name: elastic-query
        type: elasticsearch-fe-query
        config:
          listenPort: 8080
          disableAuth: true
    backendConnectors:
      - name: opensearch
        type: elasticsearch
        config:
          url: "https://opensearch-cluster-master:9200"
          user: "admin"
          password: "myStrongPassword@123!"
    processors:
      - name: p-query
        type: quesma-v1-processor-query
        config:
          indexes:
            "*":
              target: [ "opensearch" ]
      - name: p-ingest
        type: quesma-v1-processor-ingest
        config:
          indexes:
            "*":
              target: [ "opensearch" ]
    pipelines:
      - name: pipeline-query
        frontendConnectors: [ "elastic-query" ]
        processors: [ "p-query" ]
        backendConnectors: [ "opensearch" ]
      - name: pipeline-ingest
        frontendConnectors: [ "elastic-ingest" ]
        processors: [ "p-ingest" ]
        backendConnectors: [ "opensearch" ]

opensearch:
  enabled: true
  persistence:
    enabled: false
  extraEnvs:
    - name: OPENSEARCH_INITIAL_ADMIN_PASSWORD
      value: "myStrongPassword@123!"

opensearch-dashboards:
  enabled: true
```

3) Install:

```bash
helm install quesma . -n quesma --create-namespace -f values.yaml
```

4) Verify resources:

```bash
kubectl get pods -n quesma
kubectl get svc -n quesma
```

5) Port-forward locally:

```bash
kubectl -n quesma port-forward svc/quesma 8080:8080
```

---

## Dependencies (OpenSearch & Dashboards)

The chart declares two optional dependencies:

- **OpenSearch** (`opensearch.enabled=true`): deploys a stateful OpenSearch cluster in the same namespace.
  - The master service endpoint is typically `opensearch-cluster-master:9200` (already referenced in defaults).
  - Set the initial admin password via:
    ```yaml
    opensearch:
      extraEnvs:
        - name: OPENSEARCH_INITIAL_ADMIN_PASSWORD
          value: "REPLACE_ME"
    ```
  - In production, enable persistence (`opensearch.persistence.enabled=true`) and configure a StorageClass and volume size.

- **OpenSearch Dashboards** (`opensearch-dashboards.enabled=true`): deploys the UI.
  - Local access:
    ```bash
    kubectl -n quesma port-forward svc/opensearch-dashboards 5601:5601
    # Then open http://localhost:5601
    ```

> After editing dependencies in `Chart.yaml`, run `helm dependency update .` again.

---

## Quesma Configuration

Most settings are defined in `values.yaml`. Key sections include:

### Image

```yaml
image:
  repository: quesma/quesma
  pullPolicy: Always
  tag: ""   # must be set explicitly (avoid 'latest')
imagePullSecrets: []
```

### Service

```yaml
service:
  type: ClusterIP
  port: 8080
```

### Container Environment

```yaml
env:
  - name: QUESMA_CONFIG_FILE
    value: "/quesma-configuration/quesma-config.yaml"
```

### ConfigMap Mount

By default, the chart renders a ConfigMap named `quesma-first-config` and mounts it to `/quesma-configuration/quesma-config.yaml`:

```yaml
volumes:
  - name: quesma-config-volume
    configMap:
      name: quesma-first-config

volumeMounts:
  - name: quesma-config-volume
    mountPath: /quesma-configuration/quesma-config.yaml
    subPath: quesma-config.yaml
```

### Core Quesma Settings

See full README above (truncated for brevity)...
