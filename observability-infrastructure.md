# Observability Infrastructure — DC & Tool Deployment Guide

## How Staging vs Prod Is Determined

Staging and Prod are **not** tied to a physical datacenter. The same DC can host both staging and prod workloads. The environment is determined by the **hostname/FQDN**:

```
if 'staging' in grains['fqdn'] → staging config
else → production config
```

Examples:
- `otelgw-1-ord2-us-prod` → **prod**
- `otelgw-1-ord2-us-staging` → **staging**

Resource labels applied to all telemetry:
- `environment: 'production'` or `environment: 'staging'`
- `datacenter: '<dc-short>'`
- `component: '<service-name>'`

---

## OTEL Gateway Deployment

### Staging OTEL Gateways — 6 DCs, 18 Instances

Each DC runs **3 gateway instances** for HA.

| DC     | Location     | Hostnames                              |
|--------|--------------|----------------------------------------|
| rin1   | Dallas, US   | `otelgw-{1,2,3}-rin1-us-staging`      |
| cjj1   | Newark, US   | `otelgw-{1,2,3}-cjj1-us-staging`      |
| ord2   | Chicago, US  | `otelgw-{1,2,3}-ord2-us-staging`      |
| sea1   | Seattle, US  | `otelgw-{1,2,3}-sea1-us-staging`      |
| atl1   | Atlanta, US  | `otelgw-{1,2,3}-atl1-us-staging`      |
| par3   | Paris, FR    | `otelgw-{1,2,3}-par3-fr-staging`      |

### Production OTEL Gateways — 34 DCs, 102 Instances

Each DC runs **3 gateway instances** for HA.

**US:**
| DC   | Location           |
|------|--------------------|
| rin1 | Dallas             |
| cjj1 | Newark             |
| fnc1 | Fremont            |
| atl1 | Atlanta            |
| iad3 | Northern Virginia  |
| iad5 | Northern Virginia  |
| ord2 | Chicago            |
| sea1 | Seattle            |
| lax3 | Los Angeles        |
| mia3 | Miami              |

**Europe:**
| DC   | Location    |
|------|-------------|
| fra1 | Frankfurt   |
| fra4 | Frankfurt   |
| lon1 | London      |
| lon4 | London      |
| par3 | Paris       |
| par4 | Paris       |
| ams2 | Amsterdam   |
| sto2 | Stockholm   |
| mil1 | Milan       |
| mad2 | Madrid      |
| osl1 | Oslo        |

**Asia-Pacific / Other:**
| DC   | Location    |
|------|-------------|
| sin1 | Singapore   |
| sin2 | Singapore   |
| shg1 | Shanghai    |
| mum1 | Mumbai      |
| bom1 | Mumbai      |
| maa1 | Chennai     |
| syd1 | Sydney      |
| mel1 | Melbourne   |
| osa1 | Osaka       |
| tyo2 | Tokyo       |
| cgk1 | Jakarta     |
| tor1 | Toronto     |
| gru1 | São Paulo   |

### DCs With Both Staging + Prod OTEL GW

| DC   | Staging GW | Prod GW |
|------|:----------:|:-------:|
| rin1 | 3          | 3       |
| cjj1 | 3          | 3       |
| ord2 | 3          | 3       |
| sea1 | 3          | 3       |
| atl1 | 3          | 3       |
| par3 | 3          | 3       |

All other 28 DCs have **prod-only** OTEL gateways.

### OTEL Gateway Specs

- **Instance Type:** g6-dedicated-16 (16GB)
- **Version:** 0.115.1
- **Protocol:** TLS 1.3 on port 443
- **Receivers:** OTLP (gRPC + HTTP), Webhookevent, Syslog (RFC5424/RFC3164)
- **Processors:** Routing, Batch (10,000 msgs), Memory Limiter (75% of RAM)
- **File storage** for durable queuing at `/var/lib/otelcol/`

### OTEL Gateway Endpoints

| Direction        | Prod Endpoint                                                     | Staging Endpoint                                                    |
|------------------|-------------------------------------------------------------------|---------------------------------------------------------------------|
| Logs → Loki      | `https://loki-prim-aclp.cloud-observability.akadns.net/loki/api/v1/push` | `https://loki.infra-logging.sea1.us.staging.linode.com/otlp`       |
| Traces → Tempo   | Not deployed                                                      | `https://tempo-otlp.infra-o11y-apps.rin1.us.staging.linode.com/`   |
| ACLP Metrics     | Per-component (`metrics-ingest-vm`, `-fw`, `-nlb`, `-blk`, etc.)  | `https://metrics-ingest-integration.aclp.linode.com`                |
| MOM TSDB         | `https://tsdb-aclp-metrics-ha.cloud-observability.akadns.net/opentelemetry/` | Same                                                       |

### ACLP (CloudPulse) Staging OTEL GW

Separate ACLP managed gateways exist in staging (`aclp-otelgw-*-staging`), configured with dedicated per-service receivers:

| Service | Port  |
|---------|-------|
| VM      | :4318 |
| FW      | :4317 |
| Block   | :4320 |
| NLB     | :4321 |
| NB      | :4323 |
| LKE     | :4324 |
| OBJ     | :4319 |
| DBaaS   | :8086 (InfluxDB) |

---

## Kubernetes Cluster Tool Matrix

### Production Clusters

| Cluster | DC | OTEL Col | Prometheus | Loki | Grafana | Tempo | Pyroscope | Redpanda | VM Stack | OTel-Op | Host-SD |
|---------|-----|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| logging-atl1-us-prod | atl1 | Y | Y | **Y** | - | - | - | - | - | - | - |
| o11y-apps-iad3-us-prod | iad3 | Y | Y (gecko) | - | - | - | - | - | - | - | **Y** |
| o11y-apps-osa1-jp-prod | osa1 | Y | Y (gecko) | - | - | - | - | - | - | - | - |
| o11y-apps-sto2-se-prod | sto2 | Y | Y (gecko) | - | - | - | - | - | - | - | - |
| victoriametrics-ord2-us-prod | ord2 | Y | - | - | - | - | - | Y | **Y** | Y | - |
| victoriametrics-lax3-us-prod | lax3 | Y | - | - | - | - | - | Y | **Y** | Y | - |
| victoriametrics-sto2-se-prod | sto2 | Y | - | - | - | - | - | Y | **Y** | Y | - |
| victoriametrics-osa1-jp-prod | osa1 | Y | - | - | - | - | - | Y | **Y** | Y | - |
| victoriametrics-mad2-es-prod | mad2 | Y | - | - | - | - | - | Y | **Y** | Y | - |
| victoriametrics-cgk1-id-prod | cgk1 | Y | Y | - | - | - | - | - | **Y** | - | - |
| vmselect-ord2-us-prod | ord2 | Y | - | - | - | - | - | - | **Y** | Y | - |

### Staging Clusters

| Cluster | DC | OTEL Col | Prometheus | Loki | Grafana | Tempo | Pyroscope | Redpanda | VM Stack | OTel-Op | Host-SD |
|---------|-----|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| logging-sea1-us-staging | sea1 | Y | Y | **Y** | - | - | - | - | - | - | - |
| o11y-apps-rin1-us-staging | rin1 | Y | Y (gecko) | - | - | **Y** | **Y** | - | - | - | **Y** |
| o11y-apps-sto2-se-staging | sto2 | Y | Y (gecko) | - | - | - | - | - | - | - | **Y** |
| o11y-apps-sea1-us-staging | sea1 | **N** | - | - | - | - | - | - | - | - | - |
| o11y-apps-mia3-us-staging | mia3 | **N** | - | - | **Y** | - | - | - | - | - | - |
| victoriametrics-iad3-us-staging | iad3 | Y | - | - | - | - | - | Y | **Y** | Y | - |
| victoriametrics-sea1-us-staging | sea1 | Y | - | - | - | - | - | Y | **Y** | Y | - |
| vmselect-ord2-us-staging | ord2 | Y | - | - | - | - | - | - | **Y** | Y | - |

### Stable / Lab

| Cluster | DC | OTEL Col | Prometheus | Loki | Grafana | Tempo | Pyroscope | Redpanda | VM Stack | OTel-Op | Host-SD |
|---------|-----|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| o11y-apps-labkrk2-pl-stable | labkrk2 | - | - | - | **Y** | - | - | - | - | - | **Y** |

---

## Bare-Metal Tool Deployment (Salt-Managed)

| Tool | Prod DCs | Staging DCs | Notes |
|------|----------|-------------|-------|
| **OTEL Gateway** | 34 DCs (3 per DC = 102 instances) | 6 DCs (3 per DC = 18 instances) | Staging only in rin1, cjj1, ord2, sea1, atl1, par3 |
| **Prometheus** | 32+ DCs (2 sharded per DC) | cjj1-us-dev | Heavy sharding (up to 6) in iad3, ord2, par3, sea1, lax3, osa1 |
| **Thanos Query** | ord2 (2), rin1 (2) | None | Distributed query layer |
| **Grafana** | ord2, rin1 (bare-metal) + K8s | mia3, labkrk2 (K8s only) | Central instance via Terraform |
| **Alertmanager (ACLP)** | iad3 (3), ord2 (3), lon4 (3), sin2 (3) | iad3-dev (3), ord2-dev (1) | ACLP alerting |
| **Legacy Monitors** | 11 DCs (dallas, newark, fremont, atlanta, london, singapore, frankfurt, shg1, tor1, mum1, syd1) | None | Legacy system |

---

## Centralized Services

| Service | Prod Location | Staging Location | Endpoint |
|---------|---------------|------------------|----------|
| **Loki (Logs)** | atl1 (Atlanta) | sea1 (Seattle) | `loki.infra-logging.{dc}.{country}.{env}.linode.com` |
| **Tempo (Traces)** | Not deployed | rin1 (beta) | `tempo-otlp.infra-o11y-apps.rin1.us.staging.linode.com` |
| **VM Global Select** | ord2 (Chicago) | ord2 (Chicago) | `global-select.vmselect.ord2.us.{env}.linode.com` |
| **Pyroscope (Profiling)** | Not deployed | rin1 | K8s only |
| **Central Grafana** | Terraform-managed | Terraform-managed | Datasources configured per env |

---

## VictoriaMetrics Clusters

### Production

| Cluster | DC | Role | Retention | Storage |
|---------|----|------|-----------|---------|
| victoriametrics-ord2-us-prod | ord2 | Primary NA + Global hub | 30d | 300Gi PVC |
| victoriametrics-lax3-us-prod | lax3 | NA West | 30d | 1.39 TB |
| victoriametrics-sto2-se-prod | sto2 | Europe | 30d | - |
| victoriametrics-osa1-jp-prod | osa1 | Asia-Pacific | 30d | - |
| victoriametrics-mad2-es-prod | mad2 | EU expansion | 30d | - |
| victoriametrics-cgk1-id-prod | cgk1 | APAC expansion | 30d | - |
| vmselect-ord2-us-prod | ord2 | Global query layer | - | 1-2TB cache |

### Staging

| Cluster | DC | Role | Retention | Storage |
|---------|----|------|-----------|---------|
| victoriametrics-iad3-us-staging | iad3 | Full staging metrics | - | - |
| victoriametrics-sea1-us-staging | sea1 | Staging metrics | 15d | 100Gi PVC |
| vmselect-ord2-us-staging | ord2 | Staging query layer | - | - |

### Prometheus Remote Write Routing

- Prometheus-1 instances → geographically **WEST** VictoriaMetrics cluster
- Prometheus-2 instances → geographically **EAST** VictoriaMetrics cluster
- Deduplication on read, not write
- Replication factor: 2 per cluster

---

## Data Flow

```
                              BARE METAL / VM
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  Host/App → OTEL Agent (per host)                            │
    │                  │                                           │
    │                  ▼                                           │
    │  OTEL Gateway (3 per DC, port 443/TLS 1.3)                  │
    │       ├── Logs ────────► Loki (atl1=prod / sea1=staging)     │
    │       ├── Metrics ─────► VictoriaMetrics (via Prom RW)       │
    │       └── Traces ──────► Tempo (rin1 staging only)           │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘

                              KUBERNETES
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  Pod → OTEL Collector DaemonSet                              │
    │             │                                                │
    │             ▼                                                │
    │  Redpanda (Kafka-compatible queue)                           │
    │       ├── Logs ────────► Loki (via OTLP)                     │
    │       └── Metrics ─────► VictoriaMetrics (via Remote Write)  │
    │                                                              │
    │  Prometheus Gecko ──────► VictoriaMetrics (Remote Write)     │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘

                           VISUALIZATION
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  Grafana ◄── VictoriaMetrics (metrics)                       │
    │          ◄── Loki (logs)                                     │
    │          ◄── Tempo (traces, staging only)                    │
    │          ◄── Pyroscope (profiling, staging only)             │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

### Where Is Staging?

| Role | DC(s) |
|------|-------|
| K8s management | **rin1** (primary), **sto2** (secondary) |
| Logging (Loki) | **sea1** |
| Metrics (VictoriaMetrics) | **iad3**, **sea1** |
| Query (vmselect) | **ord2** |
| OTEL Gateways | **rin1, cjj1, ord2, sea1, atl1, par3** |
| Traces (Tempo) | **rin1** (beta) |
| Profiling (Pyroscope) | **rin1** |

### Where Is Prod?

| Role | DC(s) |
|------|-------|
| K8s management | **iad3** (primary), **osa1**, **sto2** |
| Logging (Loki) | **atl1** |
| Metrics (VictoriaMetrics) | **ord2, lax3, sto2, osa1, mad2, cgk1** |
| Query (vmselect) | **ord2** |
| OTEL Gateways | **All 34 DCs** |
| Thanos Query | **ord2, rin1** |

### What's NOT in Staging?

- No Thanos Query
- No Tempo in prod (only rin1 staging, beta)
- No Pyroscope in prod (only rin1 staging)
- OTEL Collector disabled on `o11y-apps-sea1-us-staging` and `o11y-apps-mia3-us-staging`
- Only 6 of 34 DCs have staging OTEL gateways

### What's NOT in Prod?

- No Tempo (tracing) — staging/beta only
- No Pyroscope (profiling) — staging only
- No Grafana on o11y-apps K8s clusters (uses central Grafana)

---

## Key Configuration Sources

| What | Where |
|------|-------|
| OTEL GW IPs | `salt-kvdata/ips/otelgw.json` |
| OTEL GW config (prod) | `salt-pillar/otelgw/config.sls` |
| OTEL GW config (staging) | `salt-pillar/otelgw/staging/config.sls` |
| OTEL GW DC list | `salt-pillar/otelgw/lib/datacenters.sls` |
| OTEL formula | `otelcol-formula/` |
| OTEL config validator | `otelconf-validator/` |
| K8s cluster values | `o11y-helm-charts/values/<cluster-name>` |
| Prometheus IPs | `salt-kvdata/ips/prometheus.json` |
| Grafana datasources | `terraform-grafana-config/grafana.auto.tfvars` |
| Prometheus rules per DC | `prometheus_rules/infra/datacenter/<dc>/` |
| Salt pillar (master config) | `salt-pillar/top.sls` |
| All DC definitions | `salt-kvdata/datacenters/all.json` |
| ACLP OTEL GW IPs | `salt-kvdata/ips/aclp-otelgw.json` |
| Pillar overrides | `salt-pillar-overrides/override/` |
| Terraform infra per region | `terraform-module-infra/infra/vars/` |
| Team & PagerDuty config | `terraform-observability-team/` |
