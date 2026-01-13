# OpenTelemetry Implementation Guide - Your Company's Infrastructure

This document provides a comprehensive guide to how OpenTelemetry is implemented across your organization's infrastructure, including data flows, configurations, and deployment patterns.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Key Components](#2-key-components)
3. [Data Flow - How Telemetry Moves](#3-data-flow---how-telemetry-moves)
4. [OpenTelemetry Gateway (otelgw)](#4-opentelemetry-gateway-otelgw)
5. [Agent Mode Configuration](#5-agent-mode-configuration)
6. [Loki Integration - Centralized Logging](#6-loki-integration---centralized-logging)
7. [Metrics Pipeline](#7-metrics-pipeline)
8. [Kubernetes (Helm/ArgoCD) Deployments](#8-kubernetes-helmargocd-deployments)
9. [Salt Formula Deployment](#9-salt-formula-deployment)
10. [Configuration Validation](#10-configuration-validation)
11. [Security and TLS](#11-security-and-tls)
12. [Key Endpoints Reference](#12-key-endpoints-reference)
13. [Repository Reference](#13-repository-reference)
14. [Troubleshooting Guide](#14-troubleshooting-guide)

---

## 1. Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  Bare Metal Hosts    │    Kubernetes Pods    │    Network Devices           │
│  (Salt-managed)      │    (Helm-deployed)    │    (SmartWall, Junos)        │
└──────────┬───────────┴──────────┬────────────┴──────────┬────────────────────┘
           │                      │                       │
           ▼                      ▼                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OTEL COLLECTOR AGENTS (Per Host/Pod)                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  filelog    │  │  journald   │  │    otlp     │  │  webhook    │         │
│  │  receiver   │  │  receiver   │  │  receiver   │  │  receiver   │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         └────────────────┴────────────────┴────────────────┘                 │
│                                   │                                          │
│                    ┌──────────────┴──────────────┐                           │
│                    │      PROCESSORS            │                           │
│                    │  • batch (10000 msgs)      │                           │
│                    │  • memory_limiter          │                           │
│                    │  • filter (stale logs)     │                           │
│                    │  • resource (labels)       │                           │
│                    └──────────────┬──────────────┘                           │
│                                   │                                          │
│                    ┌──────────────┴──────────────┐                           │
│                    │      EXPORTERS              │                           │
│                    │  • otlp (to gateway)        │                           │
│                    │  • loadbalancing            │                           │
│                    └──────────────┬──────────────┘                           │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OTEL GATEWAYS (Per Datacenter)                            │
│                                                                              │
│  otelgw.{dc}.{country}.prod.linode.com:443                                  │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Receivers: otlp (TLS 1.3), webhookevent, syslog                    │    │
│  │  Processors: routing, batch, memory_limiter                         │    │
│  │  Exporters: loki/prod, loki/staging, prometheusremotewrite         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
           ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│      LOKI       │    │ VICTORIAMETRICS │    │     TEMPO       │
│  (Logs)         │    │  (Metrics)      │    │   (Traces)      │
│                 │    │                 │    │                 │
│  atl1 (prod)    │    │  vminsert       │    │  rin1 (staging) │
│  sea1 (staging) │    │  via RedPanda   │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
           │                       │                       │
           └───────────────────────┴───────────────────────┘
                                   │
                                   ▼
                        ┌─────────────────┐
                        │    GRAFANA      │
                        │  (Visualization)│
                        └─────────────────┘
```

### Two Deployment Patterns

| Pattern | Where | Managed By | Use Case |
|---------|-------|------------|----------|
| **Salt Formula** | Bare metal hosts | Salt pillar/states | Traditional infrastructure |
| **Helm Charts** | Kubernetes | ArgoCD + Helm | Cloud-native workloads |

---

## 2. Key Components

### OpenTelemetry Collector Distributions

| Distribution | Version | Use Case |
|--------------|---------|----------|
| `otelcol-contrib` | 0.115.1 | Standard (most services) |
| `otelcol-contrib` | 0.96.0 | Billing, Coldfusion |
| `otelcol-contrib` | 0.86.0 | Object storage proxy |

### Component Roles

| Component | Function | Port |
|-----------|----------|------|
| **Agent** | Collects from local sources (files, journald) | N/A |
| **Gateway** | Aggregates from agents, routes to backends | 443 (OTLP gRPC) |
| **OTEL Operator** | Manages collectors in Kubernetes | N/A |

---

## 3. Data Flow - How Telemetry Moves

### Logs Flow

```
Source → Agent Collector → Gateway Collector → Loki
```

1. **Source Generation**
   - Application writes to `/var/log/`
   - Systemd journal entries
   - Docker container logs

2. **Agent Collection** (per host)
   - `filelog` receiver reads log files
   - `journald` receiver reads systemd journal
   - Adds resource attributes: `instance`, `datacenter`, `component`

3. **Gateway Aggregation** (per datacenter)
   - DNS load-balanced: `otelgw.{dc}.{country}.prod.linode.com`
   - Routes based on content (auth.log → specific Loki instance)
   - Batches for efficiency

4. **Loki Storage**
   - Production: `https://loki.infra-logging.atl1.us.prod.linode.com/loki/api/v1/push`
   - Staging: `https://loki.infra-logging.sea1.us.staging.linode.com/loki/api/v1/push`

### Metrics Flow (Kubernetes)

```
OTLP Source → Producer Collector → RedPanda → Consumer Collector → VictoriaMetrics
```

1. **Producer** receives metrics via OTLP gRPC (port 4317)
2. **RedPanda** (Kafka) queues for reliability
3. **Consumer** exports via Prometheus Remote Write

### Traces Flow

```
Application SDK → OTLP Receiver → Tempo
```

- Staging endpoint: `tempo.infra-o11y-apps.rin1.us.staging.linode.com`
- Correlates with logs via TraceID

---

## 4. OpenTelemetry Gateway (otelgw)

### Location
- Salt pillar: `/Users/sgour/salt-pillar/otelgw/`
- Configuration template: `/Users/sgour/salt-pillar/otelgw/lib/config.j2`

### Gateway Configuration Structure

```yaml
# Full gateway configuration example
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:443
        tls:
          cert_file: /etc/ssl/certs/${hostname}.crt
          key_file: /etc/ssl/private/${hostname}.key
          min_version: "1.3"

processors:
  routing:
    from_attribute: filename
    table:
      - value: auth.log
        exporters: [loki/2]
      - value: syslog
        exporters: [loki]
  batch:
    send_batch_size: 20000
    timeout: 10s
  memory_limiter:
    check_interval: 5s
    limit_percentage: 75
    spike_limit_percentage: 10

exporters:
  loki/prod:
    endpoint: https://loki.infra-logging.atl1.us.prod.linode.com/loki/api/v1/push
    tls:
      ca_file: /etc/ssl/certs/linode_internal_cacert.pem
      cert_file: /etc/ssl/certs/${hostname}.crt
      key_file: /etc/ssl/private/${hostname}.key
    retry_on_failure:
      initial_interval: 5s
      max_interval: 15s
      max_elapsed_time: 30s
    sending_queue:
      num_consumers: 50
      queue_size: 5000

extensions:
  health_check:
    endpoint: :13133
    path: /health/status
  file_storage:
    directory: /var/lib/otelcol
    compaction:
      on_start: true

service:
  extensions: [health_check, file_storage]
  pipelines:
    logs:
      receivers: [otlp]
      processors: [routing, batch]
      exporters: [loki/prod]
```

### Accessing Gateways

```bash
# SSH to a gateway (via teleport/hss)
hss otelgw1iad3

# Check service status
systemctl status otelcol-contrib

# View logs
journalctl -u otelcol-contrib -f

# Test health
curl -k https://localhost:13133/health/status
```

### Gateway Endpoints by Datacenter

| DC Code | DNS Endpoint |
|---------|--------------|
| iad3 | `otelgw.iad3.us.prod.linode.com:443` |
| sea1 | `otelgw.sea1.us.staging.linode.com:443` |
| atl1 | `otelgw.atl1.us.prod.linode.com:443` |
| cjj1 | `otelgw.cjj1.us.staging.linode.com:443` |
| sto2 | `otelgw.sto2.se.prod.linode.com:443` |
| osa1 | `otelgw.osa1.jp.prod.linode.com:443` |

---

## 5. Agent Mode Configuration

### Location
- Salt formula: `/Users/sgour/otelcol-formula/`
- Example pillar: `/Users/sgour/otelcol-formula/agent-pillar-custom.sls`

### Agent Configuration Structure

```yaml
# Agent configuration (per host)
otelcol:
  enabled: true
  pkg_distribution: otelcol-contrib
  pkg_version: 0.115.1

  systemd_environment_file:
    env:
      GOMEMLIMIT: "200MiB"

  config:
    receivers:
      filelog:
        include:
          - /var/log/**/{syslog,auth.log}
        start_at: end
        storage: file_storage
        resource:
          instance: "{{ grains.fqdn }}"
          datacenter: "{{ datacenter }}"
        operators:
          - type: move
            from: attributes["log.file.name"]
            to: resource["filename"]

      journald:
        start_at: end
        priority: err..debug
        units:
          - ssh
          - prometheus
          - nginx

    processors:
      batch:
        send_batch_size: 10000
        timeout: 200ms
      memory_limiter:
        check_interval: 5s
        limit_percentage: 75

    exporters:
      otlp:
        endpoint: loadbal-grpc-endpoint:443
        tls:
          insecure: true
        retry_on_failure:
          max_elapsed_time: 2m
        sending_queue:
          queue_size: 5000
          storage: file_storage

    extensions:
      file_storage:
        directory: /var/lib/otelcol
        compaction:
          on_start: true
          on_rebound: true

    service:
      extensions: [file_storage]
      pipelines:
        logs:
          receivers: [filelog, journald]
          processors: [batch]
          exporters: [otlp]
```

### Common Receivers in Use

| Receiver | Purpose | Key Config |
|----------|---------|------------|
| `filelog` | Log files | `include: [/var/log/**/*.log]` |
| `journald` | Systemd journal | `units: [ssh, nginx]` |
| `otlp` | OTLP protocol | `endpoint: 0.0.0.0:4317` |
| `prometheus` | Prometheus scrape | `scrape_configs: [...]` |
| `hostmetrics` | System metrics | `scrapers: [cpu, memory, disk]` |
| `webhookevent` | HTTP webhooks | `endpoint: 0.0.0.0:7777` |
| `syslog` | RFC5424 syslog | `protocol: rfc5424` |

---

## 6. Loki Integration - Centralized Logging

### Loki Endpoints

| Environment | Endpoint | Use |
|-------------|----------|-----|
| **Production** | `https://loki.infra-logging.atl1.us.prod.linode.com/loki/api/v1/push` | All prod logs |
| **Staging** | `https://loki.infra-logging.sea1.us.staging.linode.com/loki/api/v1/push` | Staging/dev logs |
| **LKE Alpha** | `https://infra1-cjj1-alpha-loki.lkemetrics.linode.com` | K8s alpha |
| **LKE Prod** | `https://infra1-rin1-loki.lkemetrics.linode.com` | K8s production |

### Loki Labels (Resource Attributes)

All logs sent to Loki include these labels for querying:

```yaml
resource:
  instance: "hostname.dc.prod.linode.com"     # Source host
  datacenter: "iad3"                           # DC short code
  environment: "production"                    # prod/staging/dev
  component: "nodebalancer"                    # Service type
  filename: "auth.log"                         # Log file name
  path: "/var/log/auth.log"                    # Full path
  unit: "ssh"                                  # Systemd unit
  cluster: "main"                              # Cluster name
  service_name: "nginx"                        # Service identifier
```

### Querying Logs in Grafana

```logql
# Find all SSH auth failures in production
{environment="production", filename="auth.log"} |= "Failed password"

# Filter by datacenter
{datacenter="iad3", component="nodebalancer"} | json

# Search by instance
{instance=~"web.*"} |= "error"
```

### Loki Exporter Configuration

```yaml
exporters:
  loki/prod:
    endpoint: https://loki.infra-logging.atl1.us.prod.linode.com/loki/api/v1/push
    tls:
      ca_file: /etc/ssl/certs/linode_internal_cacert.pem
      cert_file: /etc/ssl/certs/host.crt
      key_file: /etc/ssl/private/host.key
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 15s
      max_elapsed_time: 30s
    sending_queue:
      enabled: true
      num_consumers: 50
      queue_size: 5000
      storage: file_storage    # Persistent queue
```

---

## 7. Metrics Pipeline

### VictoriaMetrics Integration

Metrics flow through RedPanda (Kafka) for reliability:

```
Producer (OTLP) → RedPanda Topics → Consumer → VictoriaMetrics
```

### Prometheus Remote Write Exporter

```yaml
exporters:
  prometheusremotewrite:
    endpoint: http://vminsert-victoriametrics.victoriametrics.svc.cluster.local:8480/insert/0/prometheus/api/v1/write
    resource_to_telemetry_conversion:
      enabled: true
```

### ACLP/CloudPulse Metrics

For cloud pulse metrics, separate exporters per service type:

```yaml
exporters:
  otlphttp/aclp_vm:
    endpoint: https://metrics-ingest-vm.aclp.linode.com:4318
  otlphttp/aclp_fw:
    endpoint: https://metrics-ingest-fw.aclp.linode.com:4318
  otlphttp/aclp_blk:
    endpoint: https://metrics-ingest-blk.aclp.linode.com:4318
  # ... more per service type
```

---

## 8. Kubernetes (Helm/ArgoCD) Deployments

### Repository
- Helm charts: `/Users/sgour/o11y-helm-charts/`
- OTEL resources chart: `/Users/sgour/o11y-helm-charts/charts/otel-redpanda-resources/`

### OpenTelemetry Operator

```yaml
# Deployed via ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: otel-operator
spec:
  source:
    repoURL: https://open-telemetry.github.io/opentelemetry-helm-charts
    chart: opentelemetry-operator
    targetRevision: 0.65.1
    helm:
      values: |
        manager:
          collectorImage:
            repository: otel/opentelemetry-collector-contrib
          featureGates: operator.observability.prometheus
```

### OpenTelemetryCollector Custom Resource

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-producer
spec:
  mode: statefulset
  replicas: 10
  resources:
    requests:
      memory: 4Gi
    limits:
      memory: 6Gi
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
            max_recv_msg_size_mib: 10
    processors:
      memory_limiter:
        limit_percentage: 75
      batch:
        send_batch_size: 10000
    exporters:
      kafka:
        brokers:
          - redpanda.redpanda.svc.cluster.local:9093
        topic: o11y-logs
        encoding: otlp_proto
    service:
      pipelines:
        logs:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [kafka]
```

### Values Files by Cluster

Located in `/Users/sgour/o11y-helm-charts/values/`:

| Cluster | File | Purpose |
|---------|------|---------|
| victoriametrics-sea1-us-staging | `victoriametrics-sea1-us-staging` | Staging metrics |
| victoriametrics-ord2-us-prod | `victoriametrics-ord2-us-prod` | Production metrics |
| o11y-apps-iad3-us-prod | `o11y-apps-iad3-us-prod` | Production o11y apps |
| logging-atl1-us-prod | `logging-atl1-us-prod` | Production logging |

---

## 9. Salt Formula Deployment

### Repository
- Formula: `/Users/sgour/otelcol-formula/`
- Pillar data: `/Users/sgour/salt-pillar/`
- States: `/Users/sgour/salt-states/`

### Deploying OTEL Collector via Salt

1. **Enable in pillar** (`salt-pillar/{service}/otelcol.sls`):

```yaml
otelcol:
  enabled: true
  pkg_version: 0.115.1
  config:
    # ... configuration
```

2. **Include in top.sls** (`salt-states/top.sls`):

```yaml
base:
  'web*':
    - otelcol
```

3. **Apply state**:

```bash
salt 'web01.iad3' state.apply otelcol
```

### Using the OTEL Library

For standardized configurations, use the library:

```yaml
# salt-pillar/{service}/otelcol.sls
include:
  - otelgw.lib.default:
      defaults:
        component: "myservice"
        otel_units:
          - myservice
          - nginx
        otel_files:
          - "/var/log/myservice/*.log"
        docker_logs: false
        use_infra_ipv6: false
```

### File Locations After Deployment

| File | Purpose |
|------|---------|
| `/etc/otelcol-contrib/config.yaml` | Main configuration |
| `/etc/default/otelcol-contrib` | Environment variables |
| `/etc/systemd/system/otelcol-contrib.service` | Systemd service |
| `/var/lib/otelcol/` | Persistent queue storage |
| `/var/run/otelcol-contrib/` | Runtime sockets |

---

## 10. Configuration Validation

### otelconf-validator Tool

Repository: `/Users/sgour/otelconf-validator/`

**Purpose**: Validates and detects conflicts when merging multiple OTEL configs.

### Usage

```bash
# Validate single config
./otelconf-validator --validate config.yaml

# Check multiple configs for conflicts
./otelconf-validator config1.yaml config2.yaml

# Output files generated:
# - otel_mergegen_merged.yaml    (merged config)
# - otel_mergegen_conflicts.json (conflict report)
```

### Conflict Report Format

```json
{
  "overwritten_fields": [
    {
      "key": "exporters.loki.endpoint",
      "overwritten_by": "service-config.yaml",
      "previous_value": "http://loki:3100",
      "new_value": "http://loki-prod:3100"
    }
  ],
  "duplicate_keys": [
    {
      "key": "receivers.otlp.protocols.grpc.endpoint",
      "files": ["base.yaml", "override.yaml"]
    }
  ]
}
```

### Naming Conventions (Enforced)

```yaml
# GOOD - properly namespaced
exporters:
  otlphttp/goblin_aclp:      # {type}/{service}_{team}
  loki/sre_o11y:             # {type}/{team}

# BAD - will generate warnings
exporters:
  loadbalancing:             # Missing identifier
  prometheus:                # Missing identifier
```

### Integration with CI/CD

```yaml
# GitHub Actions example
- name: Validate OTEL Config
  run: |
    ./otelconf-validator --validate config/*.yaml
    if [ -f otel_mergegen_conflicts.json ]; then
      conflicts=$(jq '.overwritten_fields | length' otel_mergegen_conflicts.json)
      if [ "$conflicts" -gt 0 ]; then
        echo "Configuration conflicts detected!"
        exit 1
      fi
    fi
```

---

## 11. Security and TLS

### Certificate Requirements

| Endpoint Type | TLS Version | Certificates Required |
|---------------|-------------|----------------------|
| Gateway OTLP (443) | TLS 1.3 | Server cert + key |
| Loki Push | TLS 1.2+ | mTLS (client + server) |
| Health Check (13133) | TLS 1.3 | Server cert |
| SmartWall Webhook (7777) | TLS 1.2 | Server cert |

### Certificate Locations

```
/etc/ssl/certs/{hostname}.crt        # Server certificate
/etc/ssl/private/{hostname}.key      # Private key
/etc/ssl/certs/linode_internal_cacert.pem  # Internal CA
```

### TLS Configuration Example

```yaml
# Server TLS (receiving connections)
receivers:
  otlp:
    protocols:
      grpc:
        tls:
          cert_file: /etc/ssl/certs/server.crt
          key_file: /etc/ssl/private/server.key
          min_version: "1.3"
          reload_interval: 12h

# Client TLS (making connections)
exporters:
  loki:
    tls:
      ca_file: /etc/ssl/certs/ca.pem
      cert_file: /etc/ssl/certs/client.crt
      key_file: /etc/ssl/private/client.key
      min_version: "1.2"
```

### Kubernetes Certificate Management

Via cert-manager:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: otel-producer-tls
spec:
  issuerRef:
    name: vault-production
    kind: ClusterIssuer
  dnsNames:
    - otel.example.linode.com
```

---

## 12. Key Endpoints Reference

### Unified OTLP Endpoints

| Environment | Endpoint | Port |
|-------------|----------|------|
| Staging | `otlp.staging.o11y.infra.linode.com` | 30001 |
| Production | `otlp.prod.o11y.infra.linode.com` | 30001 |

### Backend Endpoints

| Service | Staging | Production |
|---------|---------|------------|
| **Loki** | `loki.infra-logging.sea1.us.staging.linode.com` | `loki.infra-logging.atl1.us.prod.linode.com` |
| **Tempo** | `tempo.infra-o11y-apps.rin1.us.staging.linode.com` | N/A |
| **Pyroscope** | `pyroscope.infra-o11y-apps.rin1.us.staging.linode.com` | N/A |
| **VictoriaMetrics** | `vmselect.sea1.victoriametrics.staging.o11y.linode.com` | `vmselect.ord2.victoriametrics.prod.o11y.linode.com` |

### Grafana Datasources

All configured in `/Users/sgour/terraform-grafana-config/main.tf`

---

## 13. Repository Reference

| Repository | Path | Purpose |
|------------|------|---------|
| **otelcol-formula** | `/Users/sgour/otelcol-formula/` | Salt formula for OTEL Collector |
| **otelconf-validator** | `/Users/sgour/otelconf-validator/` | Config validation tool |
| **o11y-helm-charts** | `/Users/sgour/o11y-helm-charts/` | Kubernetes Helm charts |
| **salt-pillar** | `/Users/sgour/salt-pillar/` | Salt pillar data (109+ services) |
| **salt-states** | `/Users/sgour/salt-states/` | Salt state files |
| **terraform-observability-team** | `/Users/sgour/terraform-observability-team/` | Infrastructure as code |
| **terraform-grafana-config** | `/Users/sgour/terraform-grafana-config/` | Grafana datasource config |

### Key Files to Know

| File | What It Contains |
|------|------------------|
| `otelcol-formula/otelcol/defaults.yml` | Default collector settings |
| `otelcol-formula/agent-pillar-custom.sls` | Agent mode example |
| `otelcol-formula/gateway-pillar-custom.sls` | Gateway mode example |
| `salt-pillar/otelgw/lib/default.sls` | Reusable OTEL library |
| `salt-pillar/otelgw/config.sls` | Gateway configuration |
| `o11y-helm-charts/charts/otel-redpanda-resources/templates/producers.yaml` | K8s producer template |
| `o11y-helm-charts/charts/otel-redpanda-resources/templates/consumers.yaml` | K8s consumer template |

---

## 14. Troubleshooting Guide

### Check Collector Status

```bash
# Salt-managed hosts
systemctl status otelcol-contrib
journalctl -u otelcol-contrib -f

# Kubernetes
kubectl get pods -n observability -l app.kubernetes.io/name=otel
kubectl logs -n observability otel-producer-0 -f
```

### Common Issues

#### 1. Collector Not Starting

```bash
# Check config syntax
otelcol-contrib validate --config=/etc/otelcol-contrib/config.yaml

# Check file permissions
ls -la /etc/ssl/certs/*.crt
ls -la /etc/ssl/private/*.key
```

#### 2. Logs Not Reaching Loki

```bash
# Check gateway connectivity
curl -k https://otelgw.iad3.us.prod.linode.com:13133/health/status

# Check persistent queue
ls -la /var/lib/otelcol/

# Check exporter metrics
curl http://localhost:8888/metrics | grep otelcol_exporter
```

#### 3. Memory Issues

```bash
# Check current memory usage
curl http://localhost:8888/metrics | grep go_memstats

# Adjust GOMEMLIMIT in /etc/default/otelcol-contrib
GOMEMLIMIT=500MiB
```

#### 4. Certificate Errors

```bash
# Verify certificate validity
openssl x509 -in /etc/ssl/certs/host.crt -text -noout

# Test TLS connection
openssl s_client -connect otelgw.iad3.us.prod.linode.com:443
```

### Useful Metrics

| Metric | Meaning |
|--------|---------|
| `otelcol_exporter_queue_size` | Items waiting to be exported |
| `otelcol_exporter_send_failed_requests` | Failed export attempts |
| `otelcol_receiver_accepted_log_records` | Logs received |
| `otelcol_processor_batch_batch_send_size` | Batch sizes |

### Debug Exporter

Add temporarily for debugging:

```yaml
exporters:
  debug:
    verbosity: detailed

service:
  pipelines:
    logs:
      exporters: [loki, debug]
```

---

## Quick Start: Adding OTEL to a New Service

### For Salt-Managed Services

1. Create pillar file: `salt-pillar/{service}/otelcol.sls`

```yaml
include:
  - otelgw.lib.default:
      defaults:
        component: "myservice"
        otel_units:
          - myservice
        otel_files:
          - "/var/log/myservice/*.log"
```

2. Add to service init: `salt-pillar/{service}/init.sls`

```yaml
include:
  - myservice.otelcol
```

3. Apply: `salt '{service}*' state.apply`

### For Kubernetes Services

1. Add to `serviceTelemetryTuples` in values file:

```yaml
serviceTelemetryTuples:
  - name: myservice
    telemetryTypes:
      logs:
        partitions: 6
        otlp:
          lokiAddr: "https://loki.infra-logging.sea1.us.staging.linode.com/loki/api/v1/push"
```

2. Deploy via ArgoCD (automatic sync)

---

## Additional Resources

- OpenTelemetry Docs: https://opentelemetry.io/docs/
- Collector Contrib Repo: https://github.com/open-telemetry/opentelemetry-collector-contrib
- Loki Documentation: https://grafana.com/docs/loki/latest/
- Internal Docs: `/Users/sgour/terraform-observability-team/docs/`

---

*Document generated from analysis of local repositories on 2026-01-12*
