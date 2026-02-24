# VictoriaMetrics - The Complete Guide

---

## Table of Contents

1. [What is VictoriaMetrics & History](#1-what-is-victoriametrics--history)
2. [Architecture Overview](#2-architecture-overview)
3. [Data Model & Storage Engine Internals](#3-data-model--storage-engine-internals)
4. [All Components Deep Dive](#4-all-components-deep-dive)
5. [Data Ingestion](#5-data-ingestion)
6. [MetricsQL - The Query Language](#6-metricsql---the-query-language)
7. [Cluster Architecture in Detail](#7-cluster-architecture-in-detail)
8. [Multi-Tenancy](#8-multi-tenancy)
9. [Deployment Methods](#9-deployment-methods)
10. [VictoriaMetrics Operator for Kubernetes](#10-victoriametrics-operator-for-kubernetes)
11. [Performance Tuning](#11-performance-tuning)
12. [Backup & Restore](#12-backup--restore)
13. [Security](#13-security)
14. [Monitoring VictoriaMetrics Itself](#14-monitoring-victoriametrics-itself)
15. [Common Pitfalls & Troubleshooting](#15-common-pitfalls--troubleshooting)
16. [VictoriaLogs](#16-victorialogs)
17. [Your Team's Setup - Complete Analysis](#17-your-teams-setup---complete-analysis)

---

## 1. What is VictoriaMetrics & History

VictoriaMetrics is a fast, cost-effective, and scalable open-source time series database (TSDB) and monitoring solution. It's a drop-in replacement for Prometheus with significantly better performance and resource efficiency.

### Who Created It

- Founded in 2018 by **Aliaksandr Valialkin** in Kyiv, Ukraine (now US-headquartered)
- Aliaksandr is also the author of `fasthttp`, `fastcache`, and `quicktemplate` Go libraries (all performance-focused)
- Co-founders: **Roman Khavronenko** (ex-Cloudflare), **Dzmitry Lazerka** (ex-Lyft), **Artem Navoiev** (ex-Google)
- The idea: combine Prometheus-like monitoring with ClickHouse-like storage performance

### Why It Exists

Prometheus has fundamental limitations:

- **No horizontal scaling** - single node only
- **High memory usage** - keeps 2h of data in RAM
- **Limited retention** - not designed for long-term storage
- **No native HA** - requires external solutions (Thanos/Cortex)

VictoriaMetrics solves all of these while remaining 100% PromQL-compatible.

### Comparison Matrix

| Feature | Prometheus | Thanos | Cortex/Mimir | VictoriaMetrics |
|---|---|---|---|---|
| Written from scratch | Yes | No (reuses Prom) | No (reuses Prom) | Yes |
| Storage efficiency | 1x (baseline) | ~1x | ~1x | ~10x better |
| Compression | ~1.3 bytes/point | Same | Same | ~0.4 bytes/point |
| Horizontal scaling | No | Yes | Yes | Yes (cluster mode) |
| Long-term storage | No | Object storage (S3) | Object storage | Block storage (local disk) |
| Operational complexity | Low | High (many components) | Very high | Low |
| Query language | PromQL | PromQL | PromQL | MetricsQL (PromQL superset) |
| Dependencies | None | S3/GCS required | Consul/etcd + S3 | None |
| Single binary option | Yes | No | No | Yes |

---

## 2. Architecture Overview

VictoriaMetrics comes in two flavors:

### Single-Node Mode

One binary that does everything: ingest, store, query.

```
┌─────────────────────────┐
│    VictoriaMetrics      │
│    (single binary)      │
│                         │
│  ┌───────┐  ┌────────┐ │
│  │Ingest │  │ Query  │ │
│  │Engine │  │ Engine │ │
│  └───┬───┘  └───┬────┘ │
│      │          │       │
│  ┌───┴──────────┴────┐  │
│  │   Storage Engine  │  │
│  │   (on-disk TSDB)  │  │
│  └───────────────────┘  │
│                         │
│  HTTP :8428             │
└─────────────────────────┘
```

- **Port 8428** - HTTP API for everything
- Good for up to **~30M active time series** on a single node
- Use this when you can fit your workload on one machine

### Cluster Mode

Three separate components that scale independently:

```
                    ┌──────────────┐
Prometheus ────────►│   vminsert   │──── :8480 (HTTP)
vmagent    ────────►│  (stateless) │──── :8400 (RPC to vmstorage)
Telegraf   ────────►│              │
                    └──────┬───────┘
                           │ consistent hashing
                    ┌──────┴───────┐
                    │  vmstorage   │──── :8482 (HTTP)
                    │  (stateful)  │──── :8400 (RPC from vminsert)
                    │              │──── :8401 (RPC from vmselect)
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐
Grafana ◄──────────│   vmselect   │──── :8481 (HTTP)
API clients ◄──────│  (stateless) │──── :8401 (RPC to vmstorage)
                    └──────────────┘
```

**Key insight:** vminsert and vmselect are **stateless** (easy to scale), vmstorage is **stateful** (needs persistent disks).

### Satellite Components

| Component | Purpose | Port |
|---|---|---|
| vmagent | Scrapes metrics, forwards via remote_write | :8429 |
| vmalert | Evaluates alerting/recording rules | :8880 |
| vmauth | Auth proxy, routing, load balancing | :8427 |
| vmbackup | Creates backups from snapshots | CLI tool |
| vmrestore | Restores from backups | CLI tool |
| vmctl | Migration tool (Prometheus, InfluxDB, etc.) | CLI tool |
| vmui | Built-in web UI | embedded |
| vmanomaly | ML-based anomaly detection (Enterprise) | standalone |

---

## 3. Data Model & Storage Engine Internals

### Data Model

Same as Prometheus - a time series is uniquely identified by its metric name + set of labels:

```
http_requests_total{method="GET", handler="/api/v1/query", status="200"}
```

Each data point is a `(timestamp, value)` pair:
- **Timestamp:** Unix milliseconds (int64)
- **Value:** float64

### Metric Types

VictoriaMetrics' TSDB doesn't actually know about types - it's all just `(name, labels, timestamp, value)`. But conventionally:

- **Counter** - monotonically increasing (e.g., `http_requests_total`)
- **Gauge** - can go up and down (e.g., `temperature_celsius`)
- **Histogram** - set of counters with `le` or `vmrange` labels
- **Summary** - pre-calculated quantiles on the client side

### Storage Engine Internals (The Deep Stuff)

#### Directory Structure

```
{storageDataPath}/
├── data/                    # Time series data
│   ├── small/               # Recently flushed small parts
│   │   ├── 2024_01/         # Monthly partition
│   │   └── 2024_02/
│   └── big/                 # Merged large parts
│       ├── 2024_01/
│       └── 2024_02/
├── indexdb/                 # Inverted index (label→TSID mapping)
│   ├── current/             # Current index
│   └── previous/            # Previous index (for rotation)
├── cache/                   # Persistent cache files
├── metadata/                # Retention, etc.
└── snapshots/               # Snapshots for backup
```

#### How Ingestion Works (Step by Step)

1. Data arrives at VictoriaMetrics (or vminsert in cluster mode)
2. For each incoming time series, the system looks up its **TSID** (Time Series ID) in an in-memory cache
3. If not cached, it checks **IndexDB** on disk
4. If the series is brand new, a new TSID is created and multiple index entries are written:
   - `metricName → metricID`
   - `metricID → TSID`
   - `(date, metricName) → TSID`
   - Each label also gets indexed: `labelName=labelValue → metricID`
5. The data point goes into a **sharded in-memory buffer** (one shard per CPU core)
6. Every ~5 seconds, in-memory parts are flushed to **small** on-disk parts
7. Small parts are asynchronously merged into larger parts (not on a fixed schedule - triggered by the flush)
8. Data is stored in **columnar format**: timestamps, values, and TSIDs in separate files

#### Compression

- Uses custom compression algorithms (inspired by Gorilla but improved)
- Achieves **~0.4 bytes per data point** for typical node_exporter data
- Up to **70x compression** vs raw data
- Float values with >12 significant decimal digits may lose precision

#### Partitioning

- Data is partitioned into **per-month directories**
- Old monthly partitions are deleted when they exceed retention period
- This means actual disk usage = `retentionPeriod + ~1 month`

---

## 4. All Components Deep Dive

### vmagent

Think of vmagent as a **supercharged Prometheus scraper** that doesn't store data locally.

**What it does:**
- Scrapes targets (like Prometheus does)
- Accepts push data (InfluxDB, Graphite, DataDog, OpenTelemetry, etc.)
- Forwards everything via Prometheus `remote_write` to VictoriaMetrics
- Can do **stream aggregation** (pre-aggregate before sending)
- Can do **deduplication** at ingestion time
- Supports all Prometheus service discovery mechanisms

**Key flags:**

```bash
vmagent \
  -promscrape.config=/etc/vmagent/prometheus.yml \    # Scrape config
  -remoteWrite.url=http://vminsert:8480/insert/0/prometheus/api/v1/write \  # Where to send
  -remoteWrite.tmpDataPath=/var/lib/vmagent-remotewrite-data \  # WAL for buffering
  -remoteWrite.maxDiskUsagePerURL=1GB \               # Max buffer size
  -promscrape.maxScrapeSize=64MB \                    # Max scrape response size
  -httpListenAddr=:8429                               # HTTP port
```

**Why vmagent over Prometheus for scraping:**
- Uses **2-3x less memory** than Prometheus
- Supports reading from Kafka and other sources
- Can write to multiple remote storage systems simultaneously
- Has a **persistent buffer** (survives restarts without losing data)
- Supports **stream aggregation** to reduce the number of stored metrics

**Relabeling (3 stages):**

1. **Service Discovery Relabeling** → Before scrape (which targets to scrape) — `relabel_configs`
2. **Scrape Relabeling** → After scrape (modify scraped metrics) — `metric_relabel_configs`
3. **Remote Write Relabeling** → Before sending to remote storage — `-remoteWrite.relabelConfig`

### vmalert

**Purpose:** Evaluates alerting and recording rules, fires alerts to Alertmanager.

**How it works:**

```
┌─────────┐     query      ┌──────────────────┐
│ vmalert │───────────────►│ VictoriaMetrics   │
│         │◄───────────────│ (datasource)      │
│         │    results     └──────────────────┘
│         │
│         │   fire alerts  ┌──────────────┐
│         │───────────────►│ Alertmanager │
│         │                └──────────────┘
│         │
│         │   write rules  ┌──────────────────────┐
│         │───────────────►│ VictoriaMetrics       │
│         │                │ (remote write target) │
└─────────┘                └──────────────────────┘
```

**Key flags:**

```bash
vmalert \
  -rule=/etc/vmalert/rules/*.yml \          # Rule files
  -datasource.url=http://vmselect:8481/select/0/prometheus \  # Where to query
  -notifier.url=http://alertmanager:9093 \  # Where to send alerts
  -remoteWrite.url=http://vminsert:8480/insert/0/prometheus/api/v1/write \  # For recording rules
  -remoteRead.url=http://vmselect:8481/select/0/prometheus \  # Restore state on restart
  -evaluationInterval=30s                   # How often to evaluate rules
```

**Rule format (same as Prometheus):**

```yaml
groups:
  - name: node_alerts
    interval: 30s
    rules:
      - alert: HighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"

      - record: job:node_cpu_usage:avg
        expr: avg by(job) (rate(node_cpu_seconds_total{mode!="idle"}[5m]))
```

### vmauth

**Purpose:** Authentication proxy, router, and load balancer. It's the single entry point for your VM cluster.

**Config example (`auth.yml`):**

```yaml
users:
  # Read-only user
  - username: "grafana"
    password: "secret"
    url_prefix: "http://vmselect:8481/select/0/prometheus/"

  # Write-only user
  - username: "writer"
    password: "writesecret"
    url_map:
      - src_paths: ["/api/v1/write", "/insert/.*"]
        url_prefix: "http://vminsert:8480/"

  # Multi-tenant routing
  - bearer_token: "team-a-token"
    url_map:
      - src_paths: ["/api/v1/query.*"]
        url_prefix: "http://vmselect:8481/select/1/prometheus/"
      - src_paths: ["/api/v1/write"]
        url_prefix: "http://vminsert:8480/insert/1/prometheus/"

  # Load balancing across multiple vmselect instances
  - username: "balanced"
    password: "pass"
    url_prefix:
      - "http://vmselect-1:8481/select/0/prometheus/"
      - "http://vmselect-2:8481/select/0/prometheus/"
      - "http://vmselect-3:8481/select/0/prometheus/"

# Catch-all for unauthenticated requests
unauthorized_user:
  url_prefix: "http://vmselect:8481/select/0/prometheus/"
```

**Load balancing strategies:** least-loaded round-robin (default), round-robin, first-available.

### vmui (Built-in UI)

Accessible at `http://victoriametrics:8428/vmui` (single-node) or `http://vmselect:8481/select/0/vmui/` (cluster).

**Features:**
- Query editor with autocomplete (`Ctrl+Space`)
- Graph, Table, and Heatmap views
- **Cardinality Explorer** - find your highest-cardinality metrics
- **Metric Relabel Debugger** - test relabel rules interactively
- **Active Queries** viewer
- **Trace Analyzer** - visualize query execution traces
- **WITH Expressions Playground** - test reusable query templates

---

## 5. Data Ingestion

### Supported Protocols

| Protocol | Endpoint (single-node) | Endpoint (cluster vminsert) |
|---|---|---|
| Prometheus remote_write | `/api/v1/write` | `/insert/<tenantID>/prometheus/api/v1/write` |
| InfluxDB line protocol | `/write`, `/api/v2/write` | `/insert/<tenantID>/influx/write` |
| Graphite plaintext | TCP `:2003` | TCP `:2003` |
| OpenTSDB put | TCP `:4242` | TCP `:4242` |
| OpenTSDB HTTP | `/api/put` | `/insert/<tenantID>/opentsdb/api/put` |
| DataDog | `/datadog/api/v1/series` | `/insert/<tenantID>/datadog/api/v1/series` |
| CSV | `/api/v1/import/csv` | `/insert/<tenantID>/prometheus/api/v1/import/csv` |
| JSON lines | `/api/v1/import` | `/insert/<tenantID>/prometheus/api/v1/import` |
| Prometheus exposition | `/api/v1/import/prometheus` | `/insert/<tenantID>/prometheus/api/v1/import/prometheus` |
| OpenTelemetry | `/opentelemetry/v1/metrics` | `/insert/<tenantID>/opentelemetry/v1/metrics` |
| NewRelic | `/newrelic/api/v1/...` | `/insert/<tenantID>/newrelic/...` |

### Push vs Pull

- **Pull (scraping):** vmagent or VictoriaMetrics itself scrapes targets at regular intervals (same as Prometheus)
- **Push:** Applications push metrics to VictoriaMetrics directly via any supported protocol

---

## 6. MetricsQL - The Query Language

MetricsQL is a **superset of PromQL**. Every valid PromQL query works in MetricsQL. But MetricsQL adds significant improvements:

### Key Differences from PromQL

**1. Better `rate()` and `increase()`:**

```promql
# PromQL: rate() may return incomplete results because it doesn't look
# at the sample BEFORE the lookbehind window
rate(http_requests_total[5m])

# MetricsQL: automatically considers the last sample before the window
# → returns exact expected results
rate(http_requests_total[5m])
```

**2. Automatic lookbehind window:**

```promql
# PromQL: this is an error (missing [duration])
rate(http_requests_total)

# MetricsQL: automatically picks the right window based on step
rate(http_requests_total)  # Works!
```

**3. Metric name preservation:**

```promql
# PromQL: min_over_time({__name__="foo"}[5m]) → metric name dropped
# MetricsQL: min_over_time(foo[5m]) → keeps "foo" in the result
```

**4. NaN handling:**

MetricsQL removes NaN values from output (cleaner results).

### MetricsQL-Only Functions (Not in PromQL)

**Rollup functions:**
- `rollup(m[d])` - returns min, max, avg for each time series
- `rollup_rate(m[d])` - rollup for rate
- `range_median(m[d])` - median over time window
- `range_trim_outliers(k, m[d])` - trim outliers
- `outlier_iqr_over_time(m[d])` - IQR-based outlier detection
- `mad_over_time(m[d])` - median absolute deviation
- `zscore_over_time(m[d])` - z-score over time

**Transform functions:**
- `keep_last_value(m[d])` - fill gaps with last known value
- `interpolate(m[d])` - linear interpolation
- `range_normalize(q1, q2, ...)` - normalize series to [0,1]
- `running_sum(m)`, `running_avg(m)`, `running_max(m)`, `running_min(m)`
- `smooth_exponential(m, sf)` - exponential smoothing
- `remove_resets(m)` - remove counter resets

**Aggregate functions:**
- `median(m)` - shortcut for `quantile(0.5, m)`
- `mad(m)` - median absolute deviation across series
- `outliersk(N, m)` - returns top N outlier time series
- `limitk(N, m)` - limits results to N series
- `any(m)` - returns any single series matching

**WITH expressions (reusable subqueries):**

```promql
WITH (
  commonFilters = {job="node", instance=~"prod.*"},
  cpuUsage = 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle", commonFilters}[5m])) * 100)
)
cpuUsage > 80
```

---

## 7. Cluster Architecture in Detail

### Sharding

vminsert distributes time series across vmstorage nodes using **consistent hashing** on `metric_name + sorted(labels)`. Each unique series always goes to the same vmstorage node(s).

### Replication

```bash
# On vminsert AND vmselect:
-replicationFactor=2

# On vmselect (to deduplicate replicated data):
-dedup.minScrapeInterval=1ms
```

- vminsert writes each sample to **N distinct vmstorage nodes**
- vmselect deduplicates when querying
- Cluster survives loss of up to **N-1 vmstorage nodes**

### Multi-Level Clustering (Advanced)

```
                    ┌──────────────┐
                    │ vminsert (L1)│  ← receives data
                    └──────┬───────┘
                           │ replicates
              ┌────────────┼────────────┐
              ▼            ▼            ▼
       ┌────────────┐ ┌────────────┐ ┌────────────┐
       │vminsert AZ1│ │vminsert AZ2│ │vminsert AZ3│
       └──────┬─────┘ └──────┬─────┘ └──────┬─────┘
              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌────────────┐
       │vmstorage   │ │vmstorage   │ │vmstorage   │
       │  AZ1       │ │  AZ2       │ │  AZ3       │
       └────────────┘ └────────────┘ └────────────┘
```

### Scaling Each Component

| Component | Scaling For | Notes |
|---|---|---|
| vminsert | Higher ingestion rates | Stateless, put behind load balancer |
| vmselect | More concurrent queries | Stateless, put behind load balancer |
| vmstorage | More storage capacity/series | Stateful, adding nodes requires data rebalancing |

### Port Summary (Cluster)

| Component | HTTP Port | RPC Port |
|---|---|---|
| vminsert | 8480 | sends to vmstorage:8400 |
| vmstorage | 8482 | 8400 (from vminsert), 8401 (from vmselect) |
| vmselect | 8481 | sends to vmstorage:8401 |

---

## 8. Multi-Tenancy

Multi-tenancy is available in **cluster mode only**.

### How It Works

- Tenants are identified by `accountID` or `accountID:projectID` (both are 32-bit integers)
- Tenants are **automatically created** when the first data point arrives
- No cross-tenant queries by default (data isolation)

### URL Pattern

```
# Write
http://vminsert:8480/insert/<accountID>/prometheus/api/v1/write
http://vminsert:8480/insert/<accountID>:<projectID>/prometheus/api/v1/write

# Read
http://vmselect:8481/select/<accountID>/prometheus/api/v1/query
http://vmselect:8481/select/<accountID>:<projectID>/prometheus/api/v1/query_range

# Special: accountID=0 is the default tenant
# Special: accountID=multitenant accepts data with vm_account_id label
```

### In Your Setup

Your team uses tenant ID `0` (the default):

```
https://vminsert.victoriametrics.ord2.us.prod.linode.com/insert/0/prometheus/api/v1/write
https://vmselect.victoriametrics.ord2.us.prod.linode.com/select/0/prometheus/
```

---

## 9. Deployment Methods

### Docker (Simplest)

```bash
# Single-node
docker run -d \
  -p 8428:8428 \
  -v /data/vm:/victoria-metrics-data \
  victoriametrics/victoria-metrics:latest \
  -retentionPeriod=90d \
  -selfScrapeInterval=10s
```

### Bare Metal (systemd)

```ini
# /etc/systemd/system/victoriametrics.service
[Unit]
Description=VictoriaMetrics
After=network.target

[Service]
Type=simple
User=victoriametrics
ExecStart=/usr/local/bin/victoria-metrics-prod \
  -storageDataPath=/var/lib/victoria-metrics \
  -retentionPeriod=90d \
  -selfScrapeInterval=10s
Restart=always
LimitNOFILE=32000

[Install]
WantedBy=multi-user.target
```

### Kubernetes Helm Charts

```bash
# Add repo
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo update

# Option 1: Full stack (VM + Grafana + node-exporter + dashboards)
helm install vmstack vm/victoria-metrics-k8s-stack -n monitoring --create-namespace

# Option 2: Just the cluster
helm install vmcluster vm/victoria-metrics-cluster -n monitoring --create-namespace

# Option 3: Single node
helm install vmsingle vm/victoria-metrics-single -n monitoring --create-namespace

# Option 4: Operator (manages everything via CRDs)
helm install vmoperator vm/victoria-metrics-operator -n monitoring --create-namespace
```

**Your Team Uses:** Salt + Terraform + Helm + ArgoCD (see [section 17](#17-your-teams-setup---complete-analysis))

---

## 10. VictoriaMetrics Operator for Kubernetes

The operator manages VM components declaratively via CRDs.

### All CRDs

| CRD | Purpose | Replaces |
|---|---|---|
| VMSingle | Single-node instance | - |
| VMCluster | Cluster (vminsert+vmselect+vmstorage) | - |
| VMAgent | Metric scraper/forwarder | Prometheus |
| VMAlert | Alerting/recording rules | PrometheusRule evaluator |
| VMAlertmanager | Alert notification manager | Alertmanager |
| VMAuth | Auth proxy | - |
| VMRule | Alert/recording rules definition | PrometheusRule |
| VMServiceScrape | Scrape Kubernetes services | ServiceMonitor |
| VMPodScrape | Scrape pods directly | PodMonitor |
| VMNodeScrape | Scrape Kubernetes nodes | - |
| VMStaticScrape | Scrape static targets | - |
| VMProbe | Blackbox-style probing | Probe |
| VMScrapeConfig | Generic scrape config | ScrapeConfig |
| VMUser | User definition for VMAuth | - |

### Prometheus Operator Compatibility

The operator automatically converts Prometheus Operator CRDs (`ServiceMonitor`, `PodMonitor`, `PrometheusRule`) into VM equivalents. This is enabled by default, so existing Prometheus Operator setups work with **zero changes**.

### Example VMCluster CRD

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMCluster
metadata:
  name: vmcluster
spec:
  retentionPeriod: "12"      # 12 months
  replicationFactor: 2
  vmstorage:
    replicaCount: 3
    storage:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 100Gi
    resources:
      requests:
        memory: 4Gi
        cpu: 2
      limits:
        memory: 4Gi    # requests == limits for vmstorage!
        cpu: 2
  vmselect:
    replicaCount: 2
    resources:
      requests:
        memory: 2Gi
        cpu: 1
  vminsert:
    replicaCount: 2
    resources:
      requests:
        memory: 1Gi
        cpu: 500m
```

---

## 11. Performance Tuning

### The Golden Rule

> **Don't tune it.** VictoriaMetrics auto-tunes based on available resources. Manual tuning (especially cache sizes) often causes OOM crashes.

### Resource Guidelines

| Resource | Recommendation |
|---|---|
| RAM | Keep 50% free (for OS page cache + OOM safety) |
| CPU | Keep 50% spare (for handling spikes) |
| Disk | Keep 20%+ free (compression and reads degrade otherwise) |
| Filesystem | ext4 recommended (no OS tuning needed) |

### Key Flags (Only If You Really Need Them)

```bash
# Memory
-memory.allowedPercent=60     # % of system RAM for caches (default: 60)
-memory.allowedBytes=0        # Absolute limit (overrides percent if non-zero)

# Query limits
-search.maxConcurrentRequests=16    # Max parallel queries (default: 2*CPUs)
-search.maxQueueDuration=10s        # Max time a query waits in queue
-search.maxPointsPerTimeseries=30000  # Max points returned per series
-search.maxUniqueTimeseries=300000    # Max unique series per query

# Ingestion limits
-maxLabelsPerTimeseries=30    # Max labels per series (default: 30)

# Environment variables (alternative to flags)
-envflag.enable=true          # Then use VM_storageDataPath=/data etc.
```

### The "Slow Inserts" Metric

This is the **most important metric to watch**. If the "Slow inserts" graph on your Grafana dashboard exceeds 5% for more than 10 minutes, it means:
- The TSID cache can't hold all your active time series
- **Solution:** Add more RAM (don't try to manually tune cache sizes)

### High Churn Rate (The #1 Performance Killer)

Labels that frequently change values cause "high churn rate" which explodes cardinality:

```promql
# BAD - request_id changes every request!
http_request_duration{request_id="abc-123-def"} 0.5

# GOOD - use stable labels
http_request_duration{method="GET", path="/api/users"} 0.5
```

---

## 12. Backup & Restore

### vmbackup

```bash
# Full backup to S3
vmbackup \
  -storageDataPath=/var/lib/victoria-metrics \
  -snapshot.createURL=http://localhost:8428/snapshot/create \
  -dst=s3://my-bucket/vm-backups/full-$(date +%Y%m%d)

# Incremental backup (automatic if destination exists)
vmbackup \
  -storageDataPath=/var/lib/victoria-metrics \
  -snapshot.createURL=http://localhost:8428/snapshot/create \
  -dst=s3://my-bucket/vm-backups/latest
```

**Supported backends:** S3, GCS, Azure Blob, any S3-compatible (MinIO, Ceph), local filesystem.

**Features:**
- Incremental by default (only uploads changed data)
- Can be interrupted and resumed
- Snapshot is automatically created and deleted

### vmrestore

```bash
# STOP VictoriaMetrics FIRST!
systemctl stop victoriametrics

vmrestore \
  -src=s3://my-bucket/vm-backups/latest \
  -storageDataPath=/var/lib/victoria-metrics

# Then restart
systemctl start victoriametrics
```

### vmbackupmanager (Enterprise)

Automated scheduled backups with configurable intervals and retention.

---

## 13. Security

### TLS

```bash
# On any component:
-tls \
-tlsCertFile=/path/to/cert.pem \
-tlsKeyFile=/path/to/key.pem \
-tlsMinVersion=TLS13
```

### Cluster Internal mTLS (Enterprise)

```bash
# On vminsert, vmselect, vmstorage:
-cluster.tls=true \
-cluster.tlsCertFile=/path/to/cert.pem \
-cluster.tlsKeyFile=/path/to/key.pem \
-cluster.tlsCAFile=/path/to/ca.pem
```

### Authentication Architecture

```
                         ┌─────────┐
Internet ───── TLS ────►│ vmauth  │──── private network ────► vminsert
                        │ (auth)  │──── private network ────► vmselect
                        └─────────┘
```

- vmauth handles **Basic Auth, Bearer Token, JWT**
- Backend services (vminsert, vmselect, vmstorage) should be on an **isolated private network**
- Never send Basic Auth over plaintext

### Per-Component Basic Auth

```bash
# Protect internal HTTP endpoints
-httpAuth.username=admin \
-httpAuth.password=secretpass
```

---

## 14. Monitoring VictoriaMetrics Itself

### Metrics Endpoints

Every component exposes Prometheus metrics at `/metrics`:

| Component | URL |
|---|---|
| Single-node | `http://localhost:8428/metrics` |
| vmagent | `http://localhost:8429/metrics` |
| vmselect | `http://localhost:8481/metrics` |
| vminsert | `http://localhost:8480/metrics` |
| vmstorage | `http://localhost:8482/metrics` |

### Self-Scraping

```bash
-selfScrapeInterval=10s  # Scrape own metrics without external collector
```

### Key Metrics to Watch

| Metric | What It Means | Alert Threshold |
|---|---|---|
| `vm_slow_row_inserts_total / vm_rows_inserted_total` | % of slow inserts (cache misses) | > 5% for 10min |
| `vm_free_disk_bytes` | Free disk space | < 20% of total |
| `process_resident_memory_bytes` | RAM usage | approaching limit |
| `vm_data_size_bytes` | Total data on disk | trending |
| `vm_active_merges` | Ongoing merge operations | unusually high |
| `vm_concurrent_queries_current` | Active queries | near max |
| `vm_rows_inserted_total` | Ingestion rate | baseline comparison |

### Official Grafana Dashboards

- **Single-node:** Dashboard ID 10229
- **Cluster:** Dashboard ID 11176
- **vmagent:** Dashboard ID 12683

### Best Practice

> Monitor your VictoriaMetrics with a **separate monitoring system** (don't use VM to monitor itself in production).

---

## 15. Common Pitfalls & Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| OOM crash | Manual cache tuning or too many active series | Remove manual cache flags; add RAM |
| Slow queries | Overly broad selectors like `{__name__=~".*"}` | Use specific label matchers |
| Slow inserts > 5% | TSID cache too small for active series count | Add more RAM (don't tune caches) |
| Disk full | Didn't account for retention + 1 month overhead | retention uses `retentionPeriod + ~1 month` of disk |
| High cardinality | Labels with frequently changing values | Remove request IDs, timestamps from labels |
| Data gaps | vmagent buffer overflow, network issues | Check `-remoteWrite.maxDiskUsagePerURL` |
| Wrong query results | Stale NaN handling differs from Prometheus | MetricsQL removes NaN; use `keep_last_value()` |
| High memory after restart | Index rebuild on startup | Normal; caches warm up over time |

### Production Best Practices

1. In Kubernetes, set **requests == limits** for vmstorage (prevents OOM kills)
2. Use **default flags** - VM auto-tunes for available resources
3. Avoid labels with >30 unique values per label name
4. Set `-search.maxUniqueTimeseries` to prevent runaway queries
5. Use a **separate monitoring instance** to monitor your primary VM

---

## 16. VictoriaLogs

VictoriaMetrics' newer log management solution (alternative to Elasticsearch, Loki).

### Key Features

- **Automatic indexing** of ALL fields (no schema needed)
- Stores logs in **per-day partitions**
- Scales linearly with resources
- Can run on Raspberry Pi or 100+ CPU servers

### Ingestion

**Supports:** Elasticsearch bulk API, JSON stream, Loki push API, OpenTelemetry, Syslog

**Works with:** Fluent Bit, Fluentd, Logstash, Vector, Filebeat, Promtail

### LogsQL Query Language

```logsql
# Simple full-text search
error

# Field search with time range
_time:5m AND _msg:error AND host.name:web-server-1

# Regex
_msg:~"connection.*refused"

# Aggregation with pipes
_time:1h AND error
  | stats by (service) count() as error_count
  | sort by (error_count) desc
  | limit 10

# Extract and aggregate
_time:5m AND _msg:~"GET /api"
  | extract "time=<response_time>ms"
  | stats avg(response_time), p99(response_time)
```

---

## 17. Your Team's Setup - Complete Analysis

Based on scanning your local repos, your team runs a sophisticated, globally distributed VictoriaMetrics cluster deployment. Here's the full picture:

### Infrastructure Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    PRODUCTION (6 clusters)                    │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ ord2-us-prod │  │ lax3-us-prod │  │ mad2-es-prod │      │
│  │  (Chicago)   │  │ (Los Angeles)│  │  (Madrid)    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ osa1-jp-prod │  │ cgk1-id-prod │  │ sto2-se-prod │      │
│  │  (Osaka)     │  │  (Jakarta)   │  │ (Stockholm)  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                    STAGING (2 clusters)                       │
│                                                              │
│  ┌───────────────────┐  ┌──────────────────┐                │
│  │ iad3-us-staging   │  │ sea1-us-staging  │                │
│  │ (Washington DC)   │  │   (Seattle)      │                │
│  └───────────────────┘  └──────────────────┘                │
└──────────────────────────────────────────────────────────────┘
```

### Each Cluster Runs

```
┌──────────────────────────────────────────────────┐
│              Kubernetes Cluster                    │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │ Traefik (Ingress) + cert-manager (TLS)       │ │
│  └──────────────┬───────────────────────────────┘ │
│                 │                                  │
│  ┌──────────────▼──────┐  ┌──────────────┐       │
│  │      vminsert       │  │  vmselect    │       │
│  │  :8480              │  │  :8481       │       │
│  └──────────┬──────────┘  └──────┬───────┘       │
│             │                    │                │
│  ┌──────────▼────────────────────▼───────┐       │
│  │            vmstorage                   │       │
│  │  :8400 (insert), :8401 (select)       │       │
│  └───────────────────────────────────────┘       │
│                                                    │
│  ┌────────────────────┐  ┌─────────────────────┐ │
│  │  vmselect-envoy    │  │  Cilium (CNI)       │ │
│  │  (client-side LB)  │  │                     │ │
│  └────────────────────┘  └─────────────────────┘ │
└──────────────────────────────────────────────────┘
```

### DNS Pattern

```
{service}.victoriametrics.{dc}.{country}.{env}.linode.com
```

**Examples:**
- `vmselect.victoriametrics.ord2.us.prod.linode.com`
- `vminsert.victoriametrics.sea1.us.staging.linode.com`
- `kubeapi.victoriametrics.iad3.us.staging.linode.com`

### Data Flow in Your Setup

```
Prometheus/vmagent instances (scraping targets)
    │
    │ remote_write
    ▼
vminsert endpoints (per-region)
    │
    │ Also cross-region writes to lax3-us-prod (redundancy)
    ▼
vmstorage (distributed, replicated)
    │
    ▼
vmselect → Grafana datasources (TLS client cert auth)
```

### Infrastructure as Code Stack

| Tool | What It Manages | Location |
|---|---|---|
| Terraform | Linode K8s clusters, DNS (Akamai EdgeDNS), Nodebalancers | `terraform-module-infra/infra/victoriametrics.tf` |
| Helm + ArgoCD | VM components, Cilium, cert-manager per cluster | `o11y-helm-charts/clusters/victoriametrics-*/` |
| Salt | Bare-metal VM instances, vmagent, vmalert, iptables | `victoriametrics-formula/`, `salt-pillar/victoriametrics/` |

### Your vmselect-envoy (Unique to Your Setup)

Your team built a custom Envoy-based TCP proxy for vmselect that provides client-side load balancing across clusters. Located in `o11y-helm-charts/charts/vmselect-envoy/`, it uses different ports (11000+) to route to different regional clusters.

### Alert Rules Your Team Monitors

From `prometheus_rules/kubernetes/infra-victoriametrics/`:

- **DiskRunsOutOfSpace** - Critical at 80% disk usage
- **DiskRunsOutOfSpaceIn3Days** - Warning based on ingestion rate projection
- **RowsRejectedOnIngestion** - Data quality issues
- **VminsertVmstorageConnectionIsSaturated** - Connection health
- **LabelsLimitExceededOnIngestion** - Label cardinality enforcement
- **ProcessNearFDLimits** - System resource limits

### OTLP (OpenTelemetry) Integration

Your setup also supports OpenTelemetry ingestion with dedicated IPv6 endpoints and Traefik routing for OTLP traffic:

```
gecko-otlp-logs.victoriametrics.{dc}.{country}.{env}
```

### Dev Environment

You also have a local dev environment at `devenv/docker/vmagent/prometheus.yml` with vmagent scraping local services.

---

## Quick Reference Card

### Ports

```
Single-node:  8428 (everything)
vminsert:     8480 (HTTP), 8400 (→vmstorage RPC)
vmstorage:    8482 (HTTP), 8400 (←vminsert), 8401 (←vmselect)
vmselect:     8481 (HTTP), 8401 (→vmstorage RPC)
vmagent:      8429
vmalert:      8880
vmauth:       8427
```

### Key Flags

```bash
-storageDataPath=/data      # Where data lives
-retentionPeriod=12         # 12 months retention
-replicationFactor=2        # Write to 2 vmstorage nodes
-dedup.minScrapeInterval=1ms  # Deduplicate replicated data
-promscrape.config=prom.yml # Scrape config path
-selfScrapeInterval=10s     # Self-monitoring
```

### Ingestion URLs (cluster)

```
/insert/0/prometheus/api/v1/write       # Prometheus remote_write
/insert/0/influx/write                  # InfluxDB
/insert/0/opentelemetry/v1/metrics      # OpenTelemetry
```

### Query URLs (cluster)

```
/select/0/prometheus/api/v1/query       # Instant query
/select/0/prometheus/api/v1/query_range # Range query
/select/0/prometheus/api/v1/series      # Series metadata
/select/0/vmui/                         # Web UI
```

### Enterprise Features

Downsampling, per-tenant retention, vmanomaly, cluster mTLS, vmbackupmanager, vmgateway (OIDC/rate limiting)
