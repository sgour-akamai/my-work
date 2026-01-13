# OpenTelemetry Implementation Guide - Linode Infrastructure

This document provides a comprehensive guide to how OpenTelemetry is implemented across your organization's infrastructure, including data flows, configurations, and deployment patterns.

---

## Table of Contents

1. [Component Glossary - Quick Reference](#1-component-glossary---quick-reference)
2. [Architecture Overview](#2-architecture-overview)
3. [How Data is Exported - Push vs Pull](#3-how-data-is-exported---push-vs-pull)
4. [Exporters Deep Dive](#4-exporters-deep-dive)
5. [Receivers Deep Dive](#5-receivers-deep-dive)
6. [Processors Deep Dive](#6-processors-deep-dive)
7. [Data Flow - Metrics, Traces, Logs](#7-data-flow---metrics-traces-logs)
8. [OpenTelemetry Gateway (otelgw)](#8-opentelemetry-gateway-otelgw)
9. [Agent Mode Configuration](#9-agent-mode-configuration)
10. [Loki Integration - Centralized Logging](#10-loki-integration---centralized-logging)
11. [Tempo Integration - Distributed Tracing](#11-tempo-integration---distributed-tracing)
12. [Metrics Pipeline - VictoriaMetrics](#12-metrics-pipeline---victoriametrics)
13. [Kubernetes (Helm/ArgoCD) Deployments](#13-kubernetes-helmargocd-deployments)
14. [Salt Formula Deployment](#14-salt-formula-deployment)
15. [Application Instrumentation](#15-application-instrumentation)
16. [Configuration Validation](#16-configuration-validation)
17. [Security and TLS](#17-security-and-tls)
18. [Key Endpoints Reference](#18-key-endpoints-reference)
19. [Repository Reference](#19-repository-reference)
20. [Troubleshooting Guide](#20-troubleshooting-guide)

---

## 1. Component Glossary - Quick Reference

### Core OpenTelemetry Components

| Component | What It Is | What It Does |
|-----------|------------|--------------|
| **OpenTelemetry (OTel)** | Observability framework | A vendor-neutral standard for collecting telemetry data (metrics, traces, logs) from applications and infrastructure |
| **OTel Collector** | Data pipeline | Receives, processes, and exports telemetry data. Acts as a proxy between your apps and backends |
| **OTel SDK** | Code library | Library you add to your application code to generate traces, metrics, and logs |
| **OTel Operator** | Kubernetes controller | Manages OTel Collectors in Kubernetes, auto-injects instrumentation into pods |

### Collector Pipeline Components

| Component | What It Is | What It Does |
|-----------|------------|--------------|
| **Receiver** | Data ingestion point | Listens for or scrapes telemetry data from sources (apps, files, APIs) |
| **Processor** | Data transformer | Modifies, filters, batches, or enriches telemetry data as it flows through |
| **Exporter** | Data output | Sends processed telemetry to backend storage/visualization systems |
| **Extension** | Helper service | Provides auxiliary functionality like health checks, authentication, file storage |
| **Connector** | Pipeline bridge | Routes data between pipelines or converts data types (e.g., traces to metrics) |

### Telemetry Types

| Type | What It Is | Example | Backend Used |
|------|------------|---------|--------------|
| **Metrics** | Numeric measurements over time | CPU usage: 45%, Request count: 1000 | VictoriaMetrics, Prometheus |
| **Traces** | Request journey across services | User request → API → DB → Cache → Response | Tempo |
| **Logs** | Timestamped text events | "2024-01-01 ERROR: Connection failed" | Loki |
| **Spans** | Single unit of a trace | One function call or HTTP request within a trace | Tempo |

### Backend Systems

| System | What It Is | What It Does | Protocol Used |
|--------|------------|--------------|---------------|
| **Loki** | Log aggregation system | Stores and queries logs using labels, like Prometheus for logs | HTTP Push (Loki API) |
| **Tempo** | Distributed tracing backend | Stores traces, correlates spans, enables trace visualization | OTLP, Jaeger, Zipkin |
| **VictoriaMetrics** | Time-series database | Stores metrics, compatible with Prometheus, highly scalable | Prometheus Remote Write |
| **Prometheus** | Metrics system | Scrapes and stores metrics, provides query language (PromQL) | HTTP Pull (scrape) |
| **Grafana** | Visualization platform | Dashboards for metrics, logs, traces from all backends | Query APIs |
| **RedPanda** | Streaming platform | Kafka-compatible message queue for reliable data delivery | Kafka protocol |

### Deployment Patterns

| Pattern | What It Is | Where Used |
|---------|------------|------------|
| **Agent Mode** | Collector runs on each host | Bare metal servers, collects local logs/metrics |
| **Gateway Mode** | Collector aggregates from multiple agents | Per-datacenter, receives from all hosts in DC |
| **Sidecar Mode** | Collector runs alongside each pod | Kubernetes, per-pod telemetry collection |
| **DaemonSet Mode** | Collector runs on each Kubernetes node | Kubernetes, node-level collection |

### Protocols

| Protocol | What It Is | Push/Pull | Port |
|----------|------------|-----------|------|
| **OTLP gRPC** | OpenTelemetry's native protocol over gRPC | Push | 4317 |
| **OTLP HTTP** | OpenTelemetry's native protocol over HTTP | Push | 4318 |
| **Prometheus Scrape** | Prometheus pull-based collection | Pull | 9090 |
| **Prometheus Remote Write** | Push metrics to Prometheus-compatible backends | Push | varies |
| **Loki Push** | Push logs to Loki | Push | 3100 |
| **Jaeger** | Jaeger tracing protocol | Push | 14250 (gRPC) |
| **Zipkin** | Zipkin tracing protocol | Push | 9411 |

---

## 2. Architecture Overview

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

## 3. How Data is Exported - Push vs Pull

### Understanding Push vs Pull

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         PUSH MODEL                                       │
│  ┌──────────┐         ──────────>         ┌──────────────┐              │
│  │   App    │   App sends data TO backend │   Backend    │              │
│  └──────────┘                             └──────────────┘              │
│                                                                          │
│  Examples: OTLP, Loki Push, Prometheus Remote Write                     │
│  Used for: Logs, Traces, High-volume metrics                            │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         PULL MODEL                                       │
│  ┌──────────┐         <──────────         ┌──────────────┐              │
│  │   App    │   Backend fetches FROM app  │   Backend    │              │
│  └──────────┘                             └──────────────┘              │
│                                                                          │
│  Examples: Prometheus scrape                                             │
│  Used for: Metrics from long-running services                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Your Implementation: What Uses What

| Telemetry Type | Method | Protocol/Exporter | Destination |
|----------------|--------|-------------------|-------------|
| **Logs** | PUSH | OTLP gRPC → Loki Exporter | Loki |
| **Traces** | PUSH | OTLP HTTP | Tempo |
| **Metrics (K8s)** | PUSH | Prometheus Remote Write | VictoriaMetrics |
| **Metrics (Host)** | PULL | Prometheus Scrape | Prometheus/VictoriaMetrics |
| **ACLP Metrics** | PUSH | OTLP HTTP | CloudPulse endpoints |

### Detailed Flow: Logs (PUSH)

```
1. Application writes to /var/log/app.log
                    │
                    ▼
2. Filelog Receiver reads new lines (tail -f style)
                    │
                    ▼
3. Batch Processor groups 10000 records
                    │
                    ▼
4. OTLP Exporter PUSHES to Gateway (gRPC, port 443)
                    │
                    ▼
5. Gateway receives via OTLP Receiver
                    │
                    ▼
6. Routing Processor routes based on filename attribute
                    │
                    ▼
7. Loki Exporter PUSHES to Loki HTTP API
                    │
                    ▼
8. Loki stores logs, queryable in Grafana
```

### Detailed Flow: Metrics (PUSH via Remote Write)

```
1. Application exposes /metrics endpoint (Prometheus format)
                    │
                    ▼
2. Prometheus Receiver SCRAPES metrics (PULL from app)
                    │
                    ▼
3. Batch Processor groups metrics
                    │
                    ▼
4. Prometheus Remote Write Exporter PUSHES to VictoriaMetrics
                    │
                    ▼
5. VictoriaMetrics stores, queryable via PromQL in Grafana
```

### Detailed Flow: Traces (PUSH)

```
1. Application SDK creates span: otel.Tracer("myapp").Start(ctx, "operation")
                    │
                    ▼
2. SDK batches spans internally
                    │
                    ▼
3. OTLP HTTP Exporter PUSHES to Collector (port 4318)
                    │
                    ▼
4. Collector Batch Processor groups spans
                    │
                    ▼
5. OTLP HTTP Exporter PUSHES to Tempo
                    │
                    ▼
6. Tempo stores traces, viewable in Grafana
```

---

## 4. Exporters Deep Dive

### What is an Exporter?

> **Exporter**: A component that sends telemetry data OUT of the collector to external systems. Exporters are the "output" side of the pipeline - they take processed data and deliver it to storage backends, visualization tools, or other collectors.

### All Exporters Used in Your Infrastructure

#### 1. OTLP Exporter (Primary)

**What it does**: Sends data using OpenTelemetry's native protocol. Most efficient and feature-complete.

```yaml
# Configuration: /Users/sgour/salt-pillar/infra/otelcol.sls
exporters:
  otlp:
    endpoint: otelgw.linode.com:443
    tls:
      insecure: false
      min_version: "1.3"
    retry_on_failure:
      enabled: true
      max_elapsed_time: 2m
    sending_queue:
      enabled: true
      queue_size: 5000
      storage: file_storage  # Persist queue to disk
```

| Property | Value | Purpose |
|----------|-------|---------|
| Protocol | gRPC | Fast, bidirectional streaming |
| Port | 4317 (gRPC), 443 (with TLS) | Standard OTLP ports |
| Use Case | Agent → Gateway communication | Primary data transport |
| Compression | gzip | Reduces bandwidth |

#### 2. OTLP HTTP Exporter

**What it does**: Same as OTLP but over HTTP. Better for firewalls and proxies.

```yaml
# Configuration: /Users/sgour/host_sd/otel-collector-config.yaml
exporters:
  otlphttp:
    endpoint: http://tempo:4318
    tls:
      insecure: true
```

| Property | Value | Purpose |
|----------|-------|---------|
| Protocol | HTTP/1.1 or HTTP/2 | Firewall-friendly |
| Port | 4318 | Standard OTLP HTTP port |
| Use Case | Traces to Tempo, ACLP metrics | When gRPC is blocked |

#### 3. Loki Exporter

**What it does**: Pushes logs to Grafana Loki. Converts OTel log format to Loki's label-based format.

```yaml
# Configuration: /Users/sgour/salt-pillar/otelgw/lib/config.j2
exporters:
  loki/prod:
    endpoint: https://loki.infra-logging.atl1.us.prod.linode.com/loki/api/v1/push
    tls:
      ca_file: /etc/ssl/certs/linode_internal_cacert.pem
      cert_file: /etc/ssl/certs/${hostname}.crt
      key_file: /etc/ssl/private/${hostname}.key
    labels:
      resource:
        instance: ""
        datacenter: ""
        environment: ""
        component: ""
        filename: ""
        path: ""
        unit: ""
    retry_on_failure:
      initial_interval: 5s
      max_interval: 15s
      max_elapsed_time: 30s
    sending_queue:
      num_consumers: 50
      queue_size: 5000
```

| Property | Value | Purpose |
|----------|-------|---------|
| Protocol | HTTP POST | Loki's push API |
| Endpoint | /loki/api/v1/push | Standard Loki ingest path |
| Labels | Resource attributes → Loki labels | Enables LogQL queries |
| Queue Size | 5000 | Buffer during Loki outages |

**Loki Endpoints Used:**

| Environment | Endpoint |
|-------------|----------|
| Production | `https://loki.infra-logging.atl1.us.prod.linode.com/loki/api/v1/push` |
| Staging | `https://loki.infra-logging.sea1.us.staging.linode.com/loki/api/v1/push` |
| LKE Alpha | `https://infra1-cjj1-alpha-loki.lkemetrics.linode.com` |
| LKE Prod | `https://infra1-rin1-loki.lkemetrics.linode.com` |

#### 4. Prometheus Remote Write Exporter

**What it does**: Pushes metrics to Prometheus-compatible backends (VictoriaMetrics) using the remote write protocol.

```yaml
# Configuration: Used for VictoriaMetrics
exporters:
  prometheusremotewrite:
    endpoint: http://vminsert-victoriametrics.victoriametrics.svc.cluster.local:8480/insert/0/prometheus/api/v1/write
    resource_to_telemetry_conversion:
      enabled: true  # Convert resource attributes to metric labels
    tls:
      insecure: true  # Internal K8s network
```

| Property | Value | Purpose |
|----------|-------|---------|
| Protocol | HTTP POST | Prometheus remote write spec |
| Format | Snappy-compressed protobuf | Efficient metric encoding |
| Use Case | Metrics to VictoriaMetrics | Time-series storage |

#### 5. Prometheus Exporter

**What it does**: Exposes metrics on HTTP endpoint for Prometheus to SCRAPE. This is PULL-based.

```yaml
# Configuration: /Users/sgour/host_sd/pkg/otelsetup/otelsetup.go
exporters:
  prometheus:
    endpoint: 0.0.0.0:8888
    namespace: otelcol
    const_labels:
      service: host_sd
```

| Property | Value | Purpose |
|----------|-------|---------|
| Protocol | HTTP GET | Prometheus scrape target |
| Port | 8888, 8889 | Metrics endpoint |
| Use Case | Collector self-monitoring | Prometheus pulls metrics |

#### 6. Loadbalancing Exporter

**What it does**: Distributes data across multiple backend collectors for high availability.

```yaml
# Configuration: /Users/sgour/salt-pillar/infra/otelcol.sls
exporters:
  loadbalancing:
    protocol:
      otlp:
        endpoint: backend:4317
    resolver:
      dns:
        hostname: otelgw.iad3.us.prod.linode.com
        port: 443
```

| Property | Value | Purpose |
|----------|-------|---------|
| Protocol | OTLP (wraps) | Underlying transport |
| Resolver | DNS | Discovers backends via DNS |
| Use Case | Agent → multiple Gateways | High availability |

#### 7. ACLP/CloudPulse Exporters

**What it does**: Sends metrics to Akamai CloudPulse service-specific endpoints.

```yaml
# Configuration: /Users/sgour/otelcol-formula/agent-pillar-custom.sls
exporters:
  otlphttp/aclp_vm:
    endpoint: https://metrics-ingest-vm.aclp.linode.com:4318
  otlphttp/aclp_fw:
    endpoint: https://metrics-ingest-fw.aclp.linode.com:4318
  otlphttp/aclp_blk:
    endpoint: https://metrics-ingest-blk.aclp.linode.com:4318
  otlphttp/aclp_nb:
    endpoint: https://metrics-ingest-nb.aclp.linode.com:4318
```

| Exporter Name | Service Type |
|---------------|--------------|
| `otlphttp/aclp_vm` | Virtual Machine metrics |
| `otlphttp/aclp_fw` | Firewall metrics |
| `otlphttp/aclp_blk` | Block storage metrics |
| `otlphttp/aclp_nb` | NodeBalancer metrics |

#### 8. Kafka Exporter

**What it does**: Pushes telemetry to Kafka/RedPanda topics for reliable queuing.

```yaml
# Configuration: /Users/sgour/o11y-helm-charts K8s deployments
exporters:
  kafka:
    brokers:
      - redpanda.redpanda.svc.cluster.local:9093
    topic: o11y-logs
    encoding: otlp_proto
    producer:
      max_message_bytes: 10000000
```

| Property | Value | Purpose |
|----------|-------|---------|
| Protocol | Kafka | Message queue |
| Encoding | otlp_proto | Native OTel format |
| Use Case | Producer → RedPanda → Consumer | Reliable delivery |

#### 9. Debug Exporter

**What it does**: Prints telemetry to stdout/logs. Only for development/debugging.

```yaml
# Configuration: /Users/sgour/otelcol-formula/agent-pillar-custom.sls
exporters:
  debug:
    verbosity: detailed  # basic, normal, detailed
    sampling_initial: 5
    sampling_thereafter: 200
```

| Property | Value | Purpose |
|----------|-------|---------|
| Output | stdout | Console logging |
| Use Case | Debugging pipelines | NEVER use in production |

### Exporter Summary Table

| Exporter | Push/Pull | Telemetry Types | Destination |
|----------|-----------|-----------------|-------------|
| `otlp` | Push | Logs, Metrics, Traces | OTel Collector/Gateway |
| `otlphttp` | Push | Logs, Metrics, Traces | Tempo, ACLP |
| `loki` | Push | Logs | Grafana Loki |
| `prometheusremotewrite` | Push | Metrics | VictoriaMetrics |
| `prometheus` | Pull (exposes) | Metrics | Prometheus scrapes |
| `loadbalancing` | Push | All | Multiple collectors |
| `kafka` | Push | All | RedPanda/Kafka |
| `debug` | N/A | All | stdout (debugging) |

---

## 5. Receivers Deep Dive

### What is a Receiver?

> **Receiver**: A component that brings data INTO the collector. Receivers can either listen for incoming data (push) or actively fetch data from sources (pull/scrape).

### All Receivers Used in Your Infrastructure

#### 1. OTLP Receiver

**What it does**: Receives telemetry via OpenTelemetry Protocol. The primary receiver for application-generated telemetry.

```yaml
# Configuration: Gateway mode
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:443  # Gateway uses 443 with TLS
        max_recv_msg_size_mib: 10
        tls:
          cert_file: /etc/ssl/certs/${hostname}.crt
          key_file: /etc/ssl/private/${hostname}.key
          min_version: "1.3"
      http:
        endpoint: 0.0.0.0:4318
```

| Property | Value | Purpose |
|----------|-------|---------|
| Ports | 4317 (gRPC), 4318 (HTTP) | Standard OTLP ports |
| Push/Pull | Push (listens) | Apps send data to it |
| Telemetry | Metrics, Traces, Logs | All three signals |

#### 2. Filelog Receiver

**What it does**: Reads log files from disk, similar to `tail -f`. Tracks file offsets to avoid duplicates.

```yaml
# Configuration: /Users/sgour/otelcol-formula/agent-pillar-custom.sls
receivers:
  filelog:
    include:
      - /var/log/**/{syslog,auth.log,daemon.log,kern.log,minion,frr}
    exclude:
      - /var/log/**/*.gz
      - /var/log/**/*.1
    start_at: end  # Only new logs, not historical
    storage: file_storage  # Track offsets persistently
    resource:
      instance: "{{ grains.fqdn }}"
      datacenter: "{{ datacenter }}"
      environment: "{{ environment }}"
      component: "{{ component }}"
    operators:
      - type: move
        from: attributes["log.file.name"]
        to: resource["filename"]
      - type: move
        from: attributes["log.file.path"]
        to: resource["path"]
```

| Property | Value | Purpose |
|----------|-------|---------|
| Push/Pull | Pull (reads files) | Actively reads from disk |
| Tracking | file_storage extension | Remember last position |
| Operators | move, regex, json | Parse and transform logs |

#### 3. Journald Receiver

**What it does**: Reads logs from systemd journal (Linux system logs).

```yaml
# Configuration: /Users/sgour/otelcol-formula/agent-pillar-custom.sls
receivers:
  journald:
    start_at: end
    priority: err..debug  # Error to debug level
    units:
      - ssh
      - prometheus
      - nginx
      - node_exporter
      - otelcol-contrib
    resource:
      instance: "{{ grains.fqdn }}"
```

| Property | Value | Purpose |
|----------|-------|---------|
| Push/Pull | Pull (reads journal) | Reads from journald |
| Priority | err..debug | Log level filter |
| Units | List of services | Which systemd units to read |

#### 4. Prometheus Receiver

**What it does**: Scrapes Prometheus-format metrics from targets. Implements Prometheus service discovery.

```yaml
# Configuration: /Users/sgour/devenv/docker/aclp-gw/aclp-gateway-config.yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'node_exporter'
          scrape_interval: 30s
          static_configs:
            - targets: ['localhost:9100']
        - job_name: 'haproxy'
          scrape_interval: 10s
          static_configs:
            - targets: ['localhost:9101']
```

| Property | Value | Purpose |
|----------|-------|---------|
| Push/Pull | Pull (scrapes targets) | Fetches /metrics endpoints |
| Discovery | static, kubernetes_sd, dns_sd | Find scrape targets |
| Interval | 10s-30s typical | How often to scrape |

#### 5. Host Metrics Receiver

**What it does**: Collects system-level metrics (CPU, memory, disk, network) from the host.

```yaml
receivers:
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu:
        metrics:
          system.cpu.utilization:
            enabled: true
      memory:
      disk:
      filesystem:
      network:
      load:
      process:
        mute_process_name_error: true
```

| Property | Value | Purpose |
|----------|-------|---------|
| Push/Pull | Pull (reads /proc, /sys) | System metrics |
| Scrapers | cpu, memory, disk, etc. | What to collect |
| Interval | 30s | Collection frequency |

#### 6. Webhook Event Receiver

**What it does**: Receives HTTP webhooks and converts them to OTel logs/events.

```yaml
# Configuration: SmartWall integration
receivers:
  webhookevent:
    endpoint: 0.0.0.0:7777
    path: /events
    read_timeout: 5s
```

| Property | Value | Purpose |
|----------|-------|---------|
| Push/Pull | Push (HTTP listener) | Receives webhooks |
| Port | 7777 | SmartWall events |
| Use Case | Network device events | External system integration |

#### 7. Syslog Receiver

**What it does**: Receives syslog messages (RFC3164 or RFC5424) over UDP/TCP.

```yaml
receivers:
  syslog:
    tcp:
      listen_address: 0.0.0.0:514
    protocol: rfc5424
```

| Property | Value | Purpose |
|----------|-------|---------|
| Push/Pull | Push (listens) | Network devices send logs |
| Protocol | RFC5424, RFC3164 | Syslog format |
| Port | 514 (standard) | Syslog port |

#### 8. Docker Stats Receiver

**What it does**: Collects container metrics from Docker daemon.

```yaml
# Configuration: /Users/sgour/salt-pillar/otelgw/lib/config.j2
receivers:
  filelog/docker:
    include:
      - /var/lib/docker/containers/*/*.log
    operators:
      - type: json_parser
        timestamp:
          parse_from: attributes.time
          layout: '%Y-%m-%dT%H:%M:%S.%fZ'
```

### Receiver Summary Table

| Receiver | Push/Pull | Data Type | Source |
|----------|-----------|-----------|--------|
| `otlp` | Push | All | Applications, other collectors |
| `filelog` | Pull | Logs | Log files on disk |
| `journald` | Pull | Logs | Systemd journal |
| `prometheus` | Pull | Metrics | Prometheus /metrics endpoints |
| `hostmetrics` | Pull | Metrics | System /proc, /sys |
| `webhookevent` | Push | Logs | HTTP webhooks |
| `syslog` | Push | Logs | Syslog UDP/TCP |
| `filelog/docker` | Pull | Logs | Docker container logs |

---

## 6. Processors Deep Dive

### What is a Processor?

> **Processor**: A component that transforms telemetry data between receiving and exporting. Processors can filter, modify, batch, sample, or enrich data as it flows through the pipeline.

### All Processors Used in Your Infrastructure

#### 1. Batch Processor

**What it does**: Groups telemetry into batches for more efficient export. Reduces network calls and backend load.

```yaml
# Configuration
processors:
  batch:
    send_batch_size: 10000    # Records per batch
    timeout: 200ms            # Max wait time before sending
    send_batch_max_size: 20000  # Upper limit
```

| Property | Agent Value | Gateway Value | Purpose |
|----------|-------------|---------------|---------|
| send_batch_size | 10000 | 20000 | Target batch size |
| timeout | 200ms | 10s | Max wait time |
| Use Case | All pipelines | Efficiency | Fewer network calls |

#### 2. Memory Limiter Processor

**What it does**: Prevents collector from using too much memory. Drops data if limits exceeded (to avoid OOM).

```yaml
# Configuration
processors:
  memory_limiter:
    check_interval: 5s
    limit_percentage: 75      # Of total system memory
    spike_limit_percentage: 10
    # OR absolute limits:
    limit_mib: 250
    spike_limit_mib: 50
```

| Property | Value | Purpose |
|----------|-------|---------|
| limit_percentage | 75% | Max memory usage |
| spike_limit | 10% | Buffer for spikes |
| check_interval | 5s | How often to check |

#### 3. Resource Processor

**What it does**: Adds, modifies, or deletes resource attributes on telemetry data.

```yaml
# Configuration: Add labels to all data
processors:
  resource:
    attributes:
      - key: datacenter
        value: "iad3"
        action: upsert
      - key: environment
        value: "production"
        action: upsert
      - key: service.name
        value: "myapp"
        action: insert  # Only if not exists
```

| Action | Purpose |
|--------|---------|
| insert | Add if doesn't exist |
| update | Modify if exists |
| upsert | Add or update |
| delete | Remove attribute |

#### 4. Attributes Processor

**What it does**: Modifies attributes on individual telemetry records (not resource-level).

```yaml
processors:
  attributes:
    actions:
      - key: password
        action: delete  # Remove sensitive data
      - key: http.url
        action: hash    # Hash sensitive values
```

#### 5. Filter Processor

**What it does**: Drops telemetry that matches certain conditions. Use for reducing noise.

```yaml
# Configuration: Drop old logs
processors:
  filter:
    logs:
      log_record:
        - 'resource.attributes["filename"] == "debug.log"'
        - 'IsMatch(body, ".*TRACE.*")'
```

#### 6. Routing Processor

**What it does**: Routes telemetry to different exporters based on attributes.

```yaml
# Configuration: /Users/sgour/salt-pillar/otelgw/lib/config.j2
processors:
  routing:
    from_attribute: filename
    default_exporters: [loki/prod]
    table:
      - value: auth.log
        exporters: [loki/security]
      - value: syslog
        exporters: [loki/prod]
      - value: nginx
        exporters: [loki/web]
```

| Property | Value | Purpose |
|----------|-------|---------|
| from_attribute | filename | Attribute to route on |
| default_exporters | fallback | If no match |
| table | routing rules | Value → exporter mapping |

#### 7. Transform Processor

**What it does**: Uses OTTL (OpenTelemetry Transformation Language) to modify data.

```yaml
processors:
  transform:
    log_statements:
      - context: log
        statements:
          - set(severity_text, "ERROR") where body contains "error"
          - set(attributes["parsed"], ParseJSON(body)) where IsMatch(body, "^{")
```

### Processor Order Matters!

Processors run in the order listed in the pipeline. Recommended order:

```yaml
service:
  pipelines:
    logs:
      processors:
        - memory_limiter  # 1. First: prevent OOM
        - filter          # 2. Drop unwanted data early
        - resource        # 3. Add labels
        - batch           # 4. Last: batch for export
```

### Processor Summary Table

| Processor | Purpose | When to Use |
|-----------|---------|-------------|
| `batch` | Group for efficiency | Always |
| `memory_limiter` | Prevent OOM | Always |
| `resource` | Add labels | Add DC, env, component |
| `attributes` | Modify record attrs | Sensitive data handling |
| `filter` | Drop data | Reduce noise |
| `routing` | Send to different exporters | Multi-destination |
| `transform` | Complex modifications | OTTL transformations |

---

## 7. Data Flow - Metrics, Traces, Logs

### Logs Flow (PUSH to Loki)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          LOGS PIPELINE                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. SOURCE                                                               │
│     ├── Application writes to /var/log/app.log                          │
│     ├── Systemd journal entries                                          │
│     └── Docker container logs (/var/lib/docker/containers/)             │
│                              │                                           │
│                              ▼                                           │
│  2. AGENT COLLECTOR (per host)                                          │
│     ├── Receivers:                                                       │
│     │   ├── filelog: /var/log/**/{syslog,auth.log}                      │
│     │   ├── journald: ssh, nginx, prometheus units                      │
│     │   └── filelog/docker: container logs                              │
│     ├── Processors:                                                      │
│     │   ├── resource: add instance, datacenter, component               │
│     │   ├── memory_limiter: 250 MiB limit                               │
│     │   └── batch: 10000 records, 200ms timeout                         │
│     └── Exporters:                                                       │
│         └── otlp: PUSH to otelgw.{dc}.prod.linode.com:443              │
│                              │                                           │
│                              ▼                                           │
│  3. GATEWAY COLLECTOR (per datacenter)                                  │
│     ├── Receivers:                                                       │
│     │   └── otlp: TLS 1.3, port 443                                     │
│     ├── Processors:                                                      │
│     │   ├── routing: filename → specific Loki instance                  │
│     │   ├── memory_limiter: 75% of system memory                        │
│     │   └── batch: 20000 records, 10s timeout                           │
│     └── Exporters:                                                       │
│         ├── loki/prod: https://loki.infra-logging.atl1...               │
│         └── loki/staging: https://loki.infra-logging.sea1...            │
│                              │                                           │
│                              ▼                                           │
│  4. LOKI (log storage)                                                  │
│     ├── Labels: instance, datacenter, environment, component, filename  │
│     ├── Storage: S3-compatible object storage                           │
│     └── Query: LogQL in Grafana                                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Metrics Flow (PUSH to VictoriaMetrics)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         METRICS PIPELINE (Kubernetes)                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. SOURCE                                                               │
│     ├── Application exposes /metrics (Prometheus format)                │
│     ├── Node Exporter: port 9100                                        │
│     └── HAProxy: port 9101                                              │
│                              │                                           │
│                              ▼                                           │
│  2. PRODUCER COLLECTOR (StatefulSet, 10 replicas)                       │
│     ├── Receivers:                                                       │
│     │   ├── otlp: port 4317 (gRPC)                                      │
│     │   └── prometheus: scrape_interval 30s                             │
│     ├── Processors:                                                      │
│     │   ├── memory_limiter: 75%                                         │
│     │   └── batch: 10000 metrics                                        │
│     └── Exporters:                                                       │
│         └── kafka: PUSH to RedPanda topic "o11y-metrics"                │
│                              │                                           │
│                              ▼                                           │
│  3. REDPANDA (Kafka-compatible queue)                                   │
│     ├── Topics: o11y-metrics, o11y-logs, o11y-traces                    │
│     ├── Partitions: 6-12 per topic                                       │
│     └── Retention: Configurable                                          │
│                              │                                           │
│                              ▼                                           │
│  4. CONSUMER COLLECTOR (StatefulSet)                                    │
│     ├── Receivers:                                                       │
│     │   └── kafka: consume from RedPanda                                │
│     ├── Processors:                                                      │
│     │   └── batch                                                        │
│     └── Exporters:                                                       │
│         └── prometheusremotewrite: PUSH to VictoriaMetrics              │
│                              │                                           │
│                              ▼                                           │
│  5. VICTORIAMETRICS                                                     │
│     ├── vminsert: receives remote write                                 │
│     ├── vmstorage: stores time-series data                              │
│     ├── vmselect: PromQL queries                                        │
│     └── Query: PromQL in Grafana                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Traces Flow (PUSH to Tempo)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         TRACES PIPELINE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. SOURCE (Application SDK)                                            │
│     ├── Go: go.opentelemetry.io/otel                                    │
│     ├── Python: opentelemetry-sdk                                       │
│     └── Java: io.opentelemetry                                          │
│                                                                          │
│     Code example (Go):                                                   │
│     tracer := otel.Tracer("myservice")                                  │
│     ctx, span := tracer.Start(ctx, "operation-name")                    │
│     defer span.End()                                                     │
│                              │                                           │
│                              ▼                                           │
│  2. SDK BATCHING                                                        │
│     └── SDK batches spans internally before export                      │
│                              │                                           │
│                              ▼                                           │
│  3. OTEL COLLECTOR                                                      │
│     ├── Receivers:                                                       │
│     │   ├── otlp: port 4317 (gRPC), 4318 (HTTP)                        │
│     │   ├── jaeger: port 14250 (gRPC), 6831 (thrift)                   │
│     │   └── zipkin: port 9411                                           │
│     ├── Processors:                                                      │
│     │   └── batch: 10s timeout                                          │
│     └── Exporters:                                                       │
│         └── otlphttp: PUSH to Tempo                                     │
│                              │                                           │
│                              ▼                                           │
│  4. TEMPO (Trace Backend)                                               │
│     ├── Endpoint: tempo.infra-o11y-apps.rin1.us.staging.linode.com     │
│     ├── Retention: 48 hours                                              │
│     ├── Receivers: OTLP, Jaeger, Zipkin, OpenCensus                     │
│     └── Query: TraceQL in Grafana                                        │
│                                                                          │
│  TRACE CORRELATION:                                                      │
│  └── Trace ID in logs → click to see full trace in Tempo               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### ACLP/CloudPulse Metrics Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ACLP METRICS PIPELINE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. SOURCE                                                               │
│     ├── HAProxy stats (port 9101)                                       │
│     ├── Node Exporter (port 9100)                                       │
│     └── Application metrics                                              │
│                              │                                           │
│                              ▼                                           │
│  2. AGENT COLLECTOR                                                     │
│     ├── Receivers:                                                       │
│     │   └── prometheus: scrape_interval 30s                             │
│     ├── Processors:                                                      │
│     │   └── batch, filter                                                │
│     └── Exporters (by service type):                                    │
│         ├── otlphttp/aclp_vm: VM metrics                                │
│         ├── otlphttp/aclp_fw: Firewall metrics                          │
│         ├── otlphttp/aclp_blk: Block storage metrics                    │
│         └── otlphttp/aclp_nb: NodeBalancer metrics                      │
│                              │                                           │
│                              ▼                                           │
│  3. CLOUDPULSE ENDPOINTS                                                │
│     ├── https://metrics-ingest-vm.aclp.linode.com:4318                 │
│     ├── https://metrics-ingest-fw.aclp.linode.com:4318                 │
│     └── https://metrics-ingest-blk.aclp.linode.com:4318                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. OpenTelemetry Gateway (otelgw)

### What is the Gateway?

> **Gateway**: A centralized OTel Collector that aggregates telemetry from multiple agents in a datacenter. It provides a single point for routing, buffering, and exporting to backends.

### Location
- Salt pillar: `/Users/sgour/salt-pillar/otelgw/`
- Configuration template: `/Users/sgour/salt-pillar/otelgw/lib/config.j2`
- Terraform: `/Users/sgour/terraform-module-infra/infra/otel.tf`

### Gateway Infrastructure

| Property | Value |
|----------|-------|
| Instance Type | g6-standard-2 |
| OS Image | linode/debian11 |
| Network | Infrastructure IPv6 |
| DNS | Linode + Akamai |

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

## 9. Agent Mode Configuration

### What is Agent Mode?

> **Agent Mode**: The OTel Collector runs on each individual host, collecting local telemetry (files, journal, system metrics) and forwarding to the gateway.

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

### Common Receivers Reference

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

## 10. Loki Integration - Centralized Logging

### What is Loki?

> **Loki**: A horizontally-scalable, highly-available log aggregation system inspired by Prometheus. Unlike traditional log systems, Loki indexes only metadata (labels), not the log content, making it very efficient.

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

### Querying Logs in Grafana (LogQL)

```logql
# Find all SSH auth failures in production
{environment="production", filename="auth.log"} |= "Failed password"

# Filter by datacenter
{datacenter="iad3", component="nodebalancer"} | json

# Search by instance pattern
{instance=~"web.*"} |= "error"

# Rate of errors per minute
rate({environment="production"} |= "error" [1m])

# Top 10 hosts by log volume
topk(10, sum by (instance) (rate({environment="production"}[5m])))
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
    labels:
      resource:
        instance: ""
        datacenter: ""
        environment: ""
        component: ""
        filename: ""
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

## 11. Tempo Integration - Distributed Tracing

### What is Tempo?

> **Tempo**: Grafana's open-source, high-scale distributed tracing backend. It only requires object storage to operate and is deeply integrated with Grafana, Loki, and Prometheus.

### What are Traces?

> **Trace**: A record of a request's journey through a distributed system. Made up of **spans**, where each span represents one operation (e.g., HTTP request, database query, function call).

### Tempo Configuration

```yaml
# Location: /Users/sgour/host_sd/tempo.yaml
# Helm: /Users/sgour/o11y-helm-charts/charts/o11y-helm-charts/templates/o11y-apps-applications/tempo-application.yaml

server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        http:
          endpoint: 0.0.0.0:4318
        grpc:
          endpoint: 0.0.0.0:4317
    jaeger:
      protocols:
        grpc:
          endpoint: 0.0.0.0:14250
        thrift_binary:
          endpoint: 0.0.0.0:6832
        thrift_compact:
          endpoint: 0.0.0.0:6831
        thrift_http:
          endpoint: 0.0.0.0:14268
    zipkin:
      endpoint: 0.0.0.0:9411

storage:
  trace:
    backend: s3
    s3:
      bucket: tempo-traces
      endpoint: s3.amazonaws.com
```

### Tempo Endpoints

| Environment | Endpoint | Version |
|-------------|----------|---------|
| Staging | `tempo.infra-o11y-apps.rin1.us.staging.linode.com` | 1.7.1 |
| Production | TBD | - |

### Tempo Receivers (What Protocols It Accepts)

| Protocol | Port | Use Case |
|----------|------|----------|
| OTLP gRPC | 4317 | Native OTel apps |
| OTLP HTTP | 4318 | Native OTel apps |
| Jaeger gRPC | 14250 | Legacy Jaeger apps |
| Jaeger Thrift (binary) | 6832 | Legacy Jaeger UDP |
| Jaeger Thrift (compact) | 6831 | Legacy Jaeger UDP |
| Jaeger Thrift HTTP | 14268 | Legacy Jaeger HTTP |
| Zipkin | 9411 | Zipkin-instrumented apps |

### Trace Retention

| Property | Value |
|----------|-------|
| Retention | 48 hours |
| Storage | S3-compatible object storage |

### Sending Traces to Tempo

```yaml
# OTel Collector config to export to Tempo
exporters:
  otlphttp/tempo:
    endpoint: http://tempo:4318
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/tempo]
```

### Querying Traces in Grafana (TraceQL)

```traceql
# Find traces with errors
{ status = error }

# Find slow traces (> 1 second)
{ duration > 1s }

# Find traces for specific service
{ resource.service.name = "api-gateway" }

# Find traces with specific span name
{ name = "HTTP GET /users" }
```

### Trace-Log Correlation

Traces can be correlated with logs using the Trace ID:

1. Application logs include `trace_id` field
2. In Grafana Loki, click on trace ID to jump to Tempo
3. In Tempo, click "Logs for this span" to see related logs

---

## 12. Metrics Pipeline - VictoriaMetrics

### What is VictoriaMetrics?

> **VictoriaMetrics**: A fast, cost-effective, and scalable time-series database. Fully compatible with Prometheus, supports PromQL, and designed for high-cardinality data.

### Metrics Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VICTORIAMETRICS CLUSTER                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │   vminsert   │────▶│  vmstorage   │◀────│   vmselect   │        │
│  │  (writes)    │     │  (storage)   │     │  (queries)   │        │
│  └──────────────┘     └──────────────┘     └──────────────┘        │
│        ▲                                          │                 │
│        │                                          ▼                 │
│   Prometheus                                  Grafana               │
│   Remote Write                               (PromQL)               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### VictoriaMetrics Endpoints

| Environment | Endpoint | Component |
|-------------|----------|-----------|
| Staging | `vmselect.sea1.victoriametrics.staging.o11y.linode.com` | Query |
| Production | `vmselect.ord2.victoriametrics.prod.o11y.linode.com` | Query |
| Internal | `vminsert-victoriametrics.victoriametrics.svc.cluster.local:8480` | Write |

### Prometheus Remote Write Configuration

```yaml
exporters:
  prometheusremotewrite:
    endpoint: http://vminsert-victoriametrics.victoriametrics.svc.cluster.local:8480/insert/0/prometheus/api/v1/write
    resource_to_telemetry_conversion:
      enabled: true  # Convert resource attrs to labels
    tls:
      insecure: true  # Internal K8s network
    retry_on_failure:
      enabled: true
      max_elapsed_time: 120s
```

### Metrics via RedPanda (Kafka)

For reliability, metrics flow through RedPanda:

```
Producer (OTLP) → RedPanda Topics → Consumer → VictoriaMetrics
```

| Component | Role |
|-----------|------|
| Producer Collector | Receives OTLP, writes to Kafka |
| RedPanda | Kafka-compatible message queue |
| Consumer Collector | Reads from Kafka, writes to VM |

### ACLP/CloudPulse Metrics

Separate exporters for each service type:

```yaml
exporters:
  otlphttp/aclp_vm:
    endpoint: https://metrics-ingest-vm.aclp.linode.com:4318
  otlphttp/aclp_fw:
    endpoint: https://metrics-ingest-fw.aclp.linode.com:4318
  otlphttp/aclp_blk:
    endpoint: https://metrics-ingest-blk.aclp.linode.com:4318
```

---

## 13. Kubernetes (Helm/ArgoCD) Deployments

### What is the OTel Operator?

> **OpenTelemetry Operator**: A Kubernetes operator that manages OTel Collectors. It can auto-inject instrumentation into pods and manage collector deployments.

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

## 14. Salt Formula Deployment

### What is Salt?

> **Salt (SaltStack)**: A configuration management and remote execution tool. Used to deploy and configure OTel Collectors on bare metal hosts.

### Repository
- Formula: `/Users/sgour/otelcol-formula/`
- Pillar data: `/Users/sgour/salt-pillar/`
- States: `/Users/sgour/salt-states/`

### Services with OTel Configuration (100+)

Example services in `/Users/sgour/salt-pillar/`:

| Category | Services |
|----------|----------|
| Infrastructure | api, billing, cloud, prometheus, consul, vault, grafana |
| CloudPulse | akamaicloudpulse*, cloudpulse-managed-gw |
| Storage | blockstorage, objectstorage, storagebox, storagecanary |
| Networking | nlb-controllers, loadbals, nodebalancer, dns |

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

## 15. Application Instrumentation

### What is Instrumentation?

> **Instrumentation**: Adding code to your application to generate telemetry (traces, metrics, logs). Can be automatic (agent-based) or manual (code changes).

### Go Application Instrumentation

Location: `/Users/sgour/host_sd/pkg/otelsetup/otelsetup.go`

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/otel/sdk/metric"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/exporters/prometheus"
    "go.opentelemetry.io/contrib/instrumentation/github.com/gorilla/mux/otelmux"
)

func InitOtel(ctx context.Context) error {
    // Resource with service info
    res := resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceName("host_sd"),
        semconv.ServiceNamespace("host_sd"),
        semconv.ServiceVersion(version),
    )

    // Trace exporter (OTLP HTTP to Tempo)
    traceExporter, _ := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint("tempo:4318"),
        otlptracehttp.WithInsecure(),
    )

    // Trace provider with batching
    tracerProvider := trace.NewTracerProvider(
        trace.WithBatcher(traceExporter),
        trace.WithResource(res),
    )
    otel.SetTracerProvider(tracerProvider)

    // Metrics exporter (Prometheus)
    promExporter, _ := prometheus.New()
    meterProvider := metric.NewMeterProvider(
        metric.WithReader(promExporter),
        metric.WithResource(res),
    )
    otel.SetMeterProvider(meterProvider)

    return nil
}

// Usage in code
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    ctx, span := otel.Tracer("host_sd").Start(r.Context(), "HandleRequest")
    defer span.End()

    // Your code here...
    span.SetAttributes(attribute.String("user.id", userID))
}

// HTTP middleware instrumentation
router := mux.NewRouter()
router.Use(otelmux.Middleware("host_sd"))
```

### Instrumentation Libraries Used

| Library | Purpose |
|---------|---------|
| `go.opentelemetry.io/otel` | Core OTel SDK |
| `go.opentelemetry.io/otel/sdk/trace` | Trace SDK |
| `go.opentelemetry.io/otel/sdk/metric` | Metric SDK |
| `go.opentelemetry.io/contrib/instrumentation/github.com/gorilla/mux/otelmux` | Gorilla Mux HTTP instrumentation |
| `go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp` | OTLP trace exporter |
| `go.opentelemetry.io/otel/exporters/prometheus` | Prometheus metric exporter |

### Resource Attributes (Semantic Conventions)

```go
// Semantic Conventions v1.20.0
resource.NewWithAttributes(
    semconv.SchemaURL,
    semconv.ServiceName("host_sd"),
    semconv.ServiceNamespace("host_sd"),
    semconv.ServiceVersion(version),
    semconv.DeploymentEnvironment("production"),
)
```

---

## 16. Configuration Validation

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

---

## 17. Security and TLS

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
/etc/pki/linode_test_ca/certs/       # Test CA certs
```

### TLS Configuration Examples

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

### Memory Management

```yaml
# Environment variable
GOMEMLIMIT: "200MiB"

# Processor configuration
processors:
  memory_limiter:
    check_interval: 5s
    limit_mib: 250
    spike_limit_mib: 50
```

---

## 18. Key Endpoints Reference

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

### Collector Ports Reference

| Port | Protocol | Use |
|------|----------|-----|
| 443 | OTLP gRPC (TLS) | Gateway receives from agents |
| 4317 | OTLP gRPC | Standard OTLP gRPC |
| 4318 | OTLP HTTP | Standard OTLP HTTP |
| 8888 | HTTP | Prometheus metrics (collector self-monitoring) |
| 8889 | HTTP | Prometheus metrics (alternate) |
| 9100 | HTTP | Node Exporter metrics |
| 9101 | HTTP | HAProxy metrics |
| 13133 | HTTP | Health check endpoint |
| 7777 | HTTP | Webhook events (SmartWall) |

### Grafana Datasources

All configured in `/Users/sgour/terraform-grafana-config/main.tf`

---

## 19. Repository Reference

| Repository | Path | Purpose |
|------------|------|---------|
| **otelcol-formula** | `/Users/sgour/otelcol-formula/` | Salt formula for OTEL Collector |
| **otelconf-validator** | `/Users/sgour/otelconf-validator/` | Config validation tool |
| **o11y-helm-charts** | `/Users/sgour/o11y-helm-charts/` | Kubernetes Helm charts |
| **salt-pillar** | `/Users/sgour/salt-pillar/` | Salt pillar data (109+ services) |
| **salt-states** | `/Users/sgour/salt-states/` | Salt state files |
| **host_sd** | `/Users/sgour/host_sd/` | Host service discovery (Go app with OTel) |
| **terraform-observability-team** | `/Users/sgour/terraform-observability-team/` | Infrastructure as code |
| **terraform-grafana-config** | `/Users/sgour/terraform-grafana-config/` | Grafana datasource config |
| **terraform-module-infra** | `/Users/sgour/terraform-module-infra/` | Gateway infrastructure |
| **prometheus_rules** | `/Users/sgour/prometheus_rules/` | Alert rules (includes otelgw alerts) |
| **devenv** | `/Users/sgour/devenv/` | Development environment configs |

### Key Files to Know

| File | What It Contains |
|------|------------------|
| `otelcol-formula/otelcol/defaults.yml` | Default collector settings |
| `otelcol-formula/agent-pillar-custom.sls` | Agent mode example |
| `otelcol-formula/gateway-pillar-custom.sls` | Gateway mode example |
| `salt-pillar/otelgw/lib/default.sls` | Reusable OTEL library |
| `salt-pillar/otelgw/lib/config.j2` | Gateway config template |
| `salt-pillar/otelgw/config.sls` | Gateway configuration |
| `o11y-helm-charts/charts/otel-redpanda-resources/templates/producers.yaml` | K8s producer template |
| `o11y-helm-charts/charts/otel-redpanda-resources/templates/consumers.yaml` | K8s consumer template |
| `host_sd/pkg/otelsetup/otelsetup.go` | Go instrumentation example |
| `host_sd/otel-collector-config.yaml` | Dev collector config |
| `host_sd/tempo.yaml` | Dev Tempo config |
| `host_sd/docker-compose.yml` | Dev environment |

### Collector Versions

| Distribution | Version | Use Case |
|--------------|---------|----------|
| `otelcol-contrib` | 0.115.1 | Standard (most services) |
| `otelcol-contrib` | 0.96.0 | Billing, Coldfusion |
| `otelcol-contrib` | 0.86.0 | Object storage proxy |
| OTEL Operator | 0.65.1 | Kubernetes orchestration |
| Tempo | 1.7.1 | Trace backend |

---

## 20. Troubleshooting Guide

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
| `otelcol_exporter_sent_log_records` | Successfully sent logs |
| `otelcol_receiver_refused_log_records` | Rejected logs |

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
- Tempo Documentation: https://grafana.com/docs/tempo/latest/
- VictoriaMetrics Documentation: https://docs.victoriametrics.com/
- Internal Docs: `/Users/sgour/terraform-observability-team/docs/`

---

*Document updated with comprehensive details from local repository analysis on 2026-01-13*
