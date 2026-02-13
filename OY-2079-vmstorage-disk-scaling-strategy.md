# OY-2079: vmstorage Disk Usage Scaling Strategy

**Ticket:** [OY-2079](https://track.akamai.com/jira/browse/OY-2079) | **Epic:** VictoriaMetrics Scaling 2025 | **Priority:** High

---

## The Problem

After the September 2025 ord2 incident (documented in [OP-19](https://bits.linode.com/pages/ops/terraform-observability-team/proposals/victoriametrics_scaling_vmstorage/)), we scaled out vmstorage pods across all production clusters. NA clusters went from 50 to 92 pods, EU/AP clusters from 25 to 48 pods. This resolved the immediate memory and ingestion crises, but introduced a side-effect: **disk usage is heavily uneven across vmstorage PVCs**.

The original 50 pods (NA) carry over a year of historical data and sit at 68-71% disk usage, while the 36 pods added in September 2025 hold only a few months of data and sit at ~5%. The 80% usage alert (`DiskRunsOutOfSpace` in our alert rules) is getting uncomfortably close on the oldest pods.

The ticket asks: **what is the scaling strategy for handling this?** Should we write scripts to rotate pods? Route writes differently? Resize PVCs? Or does something else make more sense?

This document covers what we found, what strategies exist, what we recommend doing now, and what we should have ready if things change.

---

## How We Investigated

### Queries Used

We ran these Prometheus queries against each production cluster to understand the actual state:

**Disk usage percentage per pod:**
```promql
vm_data_size_bytes / (vm_data_size_bytes + vm_free_disk_space_bytes)
```

**Ingestion rate per pod (bytes/sec):**
```promql
rate(vm_rpc_byte_buf_size_bytes_total[1h])
```

**Free disk space with pod names:**
```promql
vm_free_disk_space_bytes
```

We checked four clusters: **ord2**, **lax3**, **mad2**, and **osa1**. (sto2 and cgk1 likely mirror their regional counterparts but should be verified.)

### What the Helm Template Says vs. What's Actually Running

The first thing we found was a significant **config drift** between what the Helm template defines and what's actually deployed.

The template is at:
```
o11y-helm-charts/charts/o11y-helm-charts/templates/_vm-lts-helpers.tpl
```

Lines 55-65 define vmstorage `replicaCount`:

```yaml
{{- if and (eq .Values.environment "staging") (eq .Values.dcShort "sea1") }}
  replicaCount: 28
{{- else if eq .Values.environment "staging" }}
  replicaCount: 12
{{- else if eq .Values.clusterSize "NA" }}
  replicaCount: 56        # <-- template says 56
{{- else if eq .Values.clusterSize "EU" }}
  replicaCount: 28        # <-- template says 28
{{- else if eq .Values.clusterSize "AP" }}
  replicaCount: 28        # <-- template says 28
{{ end }}
```

But live Prometheus data shows:

| Cluster | Region | Template Says | Actually Running | Gap |
|---------|--------|--------------|-----------------|-----|
| ord2 | NA | 56 | **92** | +36 pods |
| lax3 | NA | 56 | **92** | +36 pods |
| mad2 | EU | 28 | **48** | +20 pods |
| osa1 | AP | 28 | **48** | +20 pods |

These extra pods were added during and after the September 2025 scaling (per OP-19), but the Helm template was never updated to reflect it.

**Why this matters:** If ArgoCD syncs with the current template, it could try to scale clusters back down to 56/28 pods. With `replicationFactor=1`, the data on deleted pods (the highest ordinals, which are the newest pods) would be permanently lost. **Updating the template to match reality is the most urgent action item.**

### replicationFactor Verification

The template at line 51 has:
```yaml
replicationFactor: {{ ((((.Values).workloads).vm_k8s_stack).replicationFactor) | default 2 | int }}
```

The default is 2, but **every single cluster overrides this to 1** in its values file. We checked all eight:

| Cluster | Values File | Line |
|---------|------------|------|
| ord2 | `o11y-helm-charts/values/victoriametrics-ord2-us-prod` | 53 |
| lax3 | `o11y-helm-charts/values/victoriametrics-lax3-us-prod` | 48 |
| mad2 | `o11y-helm-charts/values/victoriametrics-mad2-es-prod` | 48 |
| sto2 | `o11y-helm-charts/values/victoriametrics-sto2-se-prod` | 41 |
| osa1 | `o11y-helm-charts/values/victoriametrics-osa1-jp-prod` | 45 |
| cgk1 | `o11y-helm-charts/values/victoriametrics-cgk1-id-prod` | 75 |
| sea1 | `o11y-helm-charts/values/victoriametrics-sea1-us-staging` | 94 |
| iad3 | `o11y-helm-charts/values/victoriametrics-iad3-us-staging` | 54 |

This was made permanent per OP-19 (lines 31-38):
> "we are currently changing replicationFactor from 2 to 1. Initial testing showed this halving memory usage while not making a substantial impact on our data availability."

This is important context for the rest of the analysis because `replicationFactor=1` means every piece of data lives on exactly one pod. There are no replicas. If a pod's disk is lost, that data is gone. (Cross-cluster redundancy still exists since each DC ships to multiple LTS clusters, but within a single cluster, there's no safety net.)

---

## What We Found: Cluster-by-Cluster

### ord2 (North America) - 92 pods, 6 TiB PVCs, 395-day retention

| Pod Generation | Ordinals | Count | Disk Usage | When Added |
|---------------|----------|-------|------------|------------|
| Original deployment | 0-49 | 50 | 68-71% | Feb 2024 (OP-06) |
| Early scale-up | 50-52 | 3 | ~47% | Pre-OP-19 |
| Second scale-up | 53-55 | 3 | ~14% | Pre-OP-19 |
| OP-19 scaling | 56-91 | 36 | ~5% | Sep-Oct 2025 |

Ingestion rate: ~52,500 bytes/sec per pod, **perfectly evenly distributed** across all 92 pods. This confirms vminsert's rendezvous hashing is working as expected — every pod gets an equal share of new writes regardless of how much historical data it holds.

### lax3 (North America) - 92 pods

| Pod Generation | Ordinals | Count | Disk Usage |
|---------------|----------|-------|------------|
| Original deployment | 0-49 | 50 | 68-71% (worst: **71.3%**) |
| Scale-ups | 50-55 | 6 | 30-47% |
| OP-19 scaling | 56-91 | 36 | ~5% |

Nearly identical pattern to ord2. Same ingestion distribution.

### mad2 (Europe) - 48 pods

| Pod Generation | Ordinals | Count | Disk Usage |
|---------------|----------|-------|------------|
| Original deployment | 0-24 | 25 | 45-60% (worst: **59.8%**) |
| Scale-up | 25-27 | 3 | ~30% |
| Post-OP-19 | 28-47 | 20 | ~5% |

Healthier than NA clusters — lower ingestion volume, same PVC size.

### osa1 (Asia Pacific) - 48 pods

| Pod Generation | Ordinals | Count | Disk Usage |
|---------------|----------|-------|------------|
| Original deployment | 0-24 | 25 | 38-52% (worst: **52.4%**) |
| Scale-up | 25-27 | 3 | ~25% |
| Post-OP-19 | 28-47 | 20 | ~5% |

Healthiest cluster. AP region has the lowest ingestion volume.

### Key Takeaway

No cluster is in danger. The worst pod across all clusters is at 71.3% (lax3). The 80% critical alert threshold has not been breached. And critically, the old pods are **declining** — they're losing data faster than they're gaining it (explained in the next section).

---

## Why the Disk Usage Is Uneven (And Why It's Expected)

VictoriaMetrics does not rebalance or migrate data when you add pods. vminsert uses rendezvous hashing to distribute **new** writes evenly across all pods, but existing data stays where it is. This is by design.

This means after scaling from 50 to 92 pods:
- **Old pods (0-49):** Hold ~14 months of data accumulated when they were sharing writes among only 50 pods. They now receive new data at the 92-pod rate (roughly half the old rate). Meanwhile, their oldest data is expiring at the 395-day retention boundary at the old higher rate.
- **New pods (56-91):** Started empty. Accumulate data only from their creation date at the 92-pod rate.

### The Math: Are Old Pods Getting Better or Worse?

Taking ord2 as the example (the numbers are similar for lax3):

- Current ingestion per pod: ~52,500 bytes/sec = ~4.53 GB/day
- Data expiring from the oldest pods: this was ingested ~395 days ago when there were only 50 pods, so it was ~92/50 = 1.84x the current per-pod rate = ~8.34 GB/day expiring
- **Net change per day on old pods: -8.34 + 4.53 = roughly -3.8 GB/day (declining)**

The old pods are shrinking. They will continue shrinking until their data profile fully reflects the 92-pod ingestion rate, which happens when the last data written at the 50-pod rate expires — about 395 days from the September 2025 scale-up, so roughly **October-November 2026**.

### Steady-State Projection

Once all clusters fully equalize:
- Per-pod data at 92 pods: 4.53 GB/day x 395 days = **~1.79 TB**
- On a 6 TiB (6.6 TB) PVC: **~27% usage per pod**

That's 2.5x headroom below the 80% alert threshold. We would only need more pods if total ingestion roughly triples from current levels.

### This Was the Original Design Intent

From OP-06 (`terraform-observability-team/docs/content/proposals/OP-06-victoriametrics-prod-infra.md`, lines 96-100):

> "vminsert distributes metrics to whatever vmstorage pods are accepting them. This means if some were to fill, they'd mark themselves read only and vminsert would simply write elsewhere. This design is so that if we need more storage, we simply scale the StatefulSet out wider."

The system was designed to handle uneven disk distribution through horizontal scaling and read-only mode. We're exactly in that scenario, and it's working.

---

## Strategies: What Was Suggested, What Works, What Doesn't

### Strategy 1: Natural Equalization (Do Nothing, Monitor) — WHAT WE RECOMMEND NOW

As shown above, the old pods are declining at ~3.8 GB/day and will converge to ~27% usage. No manual intervention is needed for the current state.

**Why this is the right call for now:**
- No pod is close to filling up (worst is 71.3%, declining)
- Ingestion is perfectly balanced already
- Zero operational risk, zero code changes
- Matches the original architecture intent (OP-06)

**What to watch for:** If ingestion growth accelerates significantly (new services onboarding, cardinality spikes), the equalization timeline could slow or reverse. The VictoriaMetrics Scaling Dashboard (referenced in the [scaling guide](https://bits.linode.com/pages/ops/terraform-observability-team/services/victoria-metrics/victoriametrics-scaling-guide/)) should be monitored.

---

### Strategy 2: Configure `storage.minFreeDiskSpaceBytes` (Safety Net) — WHAT WE SHOULD SET UP NOW

Even though pods aren't in danger today, we should configure the safety mechanism that VictoriaMetrics provides for exactly this scenario. This is a "costs nothing now, saves us in an emergency" change.

#### What This Flag Does

The `-storage.minFreeDiskSpaceBytes` flag on vmstorage sets a threshold. When free disk space drops below this value, the pod automatically enters **read-only mode**: it stops accepting writes but keeps serving reads. vminsert detects this and reroutes new data to other healthy pods. When free space recovers (old data expires, compaction finishes), the pod automatically switches back.

#### Where We Found This

**VictoriaMetrics cluster documentation** — the upstream docs live at:
```
https://docs.victoriametrics.com/victoriametrics/cluster-victoriametrics/#readonly-mode
```

We also verified this in the Go source code of VictoriaMetrics v1.126.0 (installed locally at `~/go/pkg/mod/github.com/!victoria!metrics/!victoria!metrics@v1.126.0/`). Here are the specific locations:

**Flag definition** — file `app/vmstorage/main.go`, line 63:
```go
minFreeDiskSpaceBytes = flagutil.NewBytes("storage.minFreeDiskSpaceBytes", 10e6,
    "The minimum free disk space at -storageDataPath after which the storage stops accepting new data")
```
The default is **10 MB**. On a 6 TiB volume, 10 MB free means the disk is 99.9999% full — the filesystem would be corrupted long before this kicks in. This default is effectively useless for our setup.

**Read-only mode check** — file `lib/storage/storage.go`, lines 795-817. The storage runs a background goroutine that checks free disk space approximately every 1 second:
```go
func (s *Storage) startFreeDiskSpaceWatcher() {
    f := func() {
        freeSpaceBytes := fs.MustGetFreeSpace(s.path)
        if freeSpaceBytes < freeDiskSpaceLimitBytes {
            // Switch the storage to readonly mode
            if !s.isReadOnly.Load() && s.isReadOnly.CompareAndSwap(false, true) {
                logger.Warnf("switching the storage at %s to read-only mode...")
            }
            return
        }
        // Switch back to read-write if space recovered
        if s.isReadOnly.Load() && s.isReadOnly.CompareAndSwap(true, false) {
            s.notifyReadWriteMode()
            logger.Warnf("switching the storage at %s to read-write mode...")
        }
    }
    ...
}
```
Key points: it checks every ~1 second, it switches automatically in both directions, and it logs the state change.

**Write rejection** — file `app/vmstorage/main.go`, lines 186-197. When a pod is read-only, it returns an error to vminsert:
```go
func AddRows(mrs []storage.MetricRow) error {
    if Storage.IsReadOnly() {
        return errReadOnly
    }
    ...
}
var errReadOnly = errors.New("the storage is in read-only mode; check -storage.minFreeDiskSpaceBytes command-line flag value")
```

**Monitoring metric** — file `app/vmstorage/main.go`, lines 491-495:
```go
isReadOnly := 0
if strg.IsReadOnly() {
    isReadOnly = 1
}
metrics.WriteGaugeUint64(w, fmt.Sprintf(`vm_storage_is_read_only{path=%q}`, *DataPath), uint64(isReadOnly))
```
This exposes `vm_storage_is_read_only` (0 or 1) at the `/metrics` endpoint.

The upstream cluster docs (file `docs/victoriametrics/Cluster-VictoriaMetrics.md`, lines 571-578) summarize it:
> "vmstorage nodes automatically switch to readonly mode when the directory pointed by -storageDataPath contains less than -storage.minFreeDiskSpaceBytes of free space. vminsert nodes stop sending data to such nodes and start re-routing the data to the remaining vmstorage nodes."

#### How vminsert Rerouting Works (Important Detail)

This is where it gets nuanced. vminsert has **two separate** rerouting flags, and they control different things. We verified these in the vminsert flags documentation (file `docs/victoriametrics/vminsert_flags.md`, lines 43-48):

**`-disableRerouting`** (default: **true**)
> "Whether to disable re-routing when some of vmstorage nodes accept incoming data at slower speed compared to other storage nodes."

This controls rerouting for **slow** nodes. The default `true` means vminsert does NOT reroute away from slow-but-healthy nodes. This is intentional — during rolling restarts, a pod that's catching up might be temporarily slow, and rerouting away from it would cause hash instability and create cardinality spikes.

**`-disableReroutingOnUnavailable`** (default: **false**)
> "Whether to disable re-routing when some of vmstorage nodes are unavailable."

This controls rerouting for **unavailable or read-only** nodes. The default `false` means vminsert WILL reroute when a node is down or in read-only mode. **This is the flag that makes the read-only safety net work.** Since we don't override this, the default is active and the safety net functions out of the box.

**`-dropSamplesOnOverload`** (default: **false**)
> "Whether to drop incoming samples if the destination vmstorage node is overloaded and/or unavailable. This prioritizes cluster availability over consistency [...] it's not recommended to use this flag with -replicationFactor enabled."

This tells vminsert to silently drop data instead of applying backpressure. With `replicationFactor=1`, enabling this could cause silent, permanent data loss. **We should never enable this unless replicationFactor is restored to 2.**

We're not setting any of these three flags — the VictoriaMetrics Operator manages vminsert and we use defaults. The defaults happen to be exactly what we want: don't reroute for slow nodes, do reroute for unavailable/read-only nodes, don't drop data.

#### What Threshold Should We Set?

The VictoriaMetrics capacity planning docs (file `docs/victoriametrics/Cluster-VictoriaMetrics.md`, line 789) recommend:
> "At least 20% of free storage space at the directory pointed by -storageDataPath"

The same section (line 791) also warns about monthly compaction:
> "on each vmstorage pod, the monthly final deduplication process will temporarily need as much space as is used for the previous month's data, before it can free up space. For example, if you have a 3 month retention period and you want to keep at least 10% space free at all times, you could pick 35% of your total space as value."

For our setup:
- **Production (6 TiB = 6,597,069,766,656 bytes):** 20% = **1,319,413,953,331 bytes** (~1.2 TiB)
- **Staging (1.8 TiB = 1,979,120,929,997 bytes):** 20% = **395,824,185,999 bytes** (~369 GiB)

This means a pod enters read-only at 80% usage — which lines up with our existing `DiskRunsOutOfSpace` critical alert threshold.

#### The Change

In the Helm template (`o11y-helm-charts/charts/o11y-helm-charts/templates/_vm-lts-helpers.tpl`), the vmstorage `extraArgs` block at lines 67-73 currently has:

```yaml
extraArgs:
  dedup.minScrapeInterval: 1ms
  search.maxUniqueTimeseries: "50000000"
```

We need to add `storage.minFreeDiskSpaceBytes`:

```yaml
extraArgs:
  dedup.minScrapeInterval: 1ms
  search.maxUniqueTimeseries: "50000000"
{{- if eq .Values.environment "staging" }}
  storage.minFreeDiskSpaceBytes: "395824185999"
{{- else if eq .Values.environment "production" }}
  storage.minFreeDiskSpaceBytes: "1319413953331"
{{- end }}
```

This change is purely additive — it doesn't affect current behavior (no pod is at 80% yet) but provides automatic protection if any pod approaches that threshold.

---

### Strategy 3: Add Read-Only Mode Alerts — SHOULD DO ALONGSIDE STRATEGY 2

If we're setting up the read-only safety net, we need to know when it activates.

**Current alerting state:** The alert rules file is at:
```
prometheus_rules/kubernetes/infra-victoriametrics/prometheus/vmcluster.yaml
```

It has six alerts today: `DiskRunsOutOfSpaceIn3Days`, `DiskRunsOutOfSpace` (fires at >80%), `RowsRejectedOnIngestion`, `ProcessNearFDLimits`, `LabelsLimitExceededOnIngestion`, and `VminsertVmstorageConnectionIsSaturated`.

**What's missing:** There is no alert on `vm_storage_is_read_only`. If a pod enters read-only mode, we currently have no direct notification — we'd only find out via the existing `DiskRunsOutOfSpace` alert or by noticing query gaps.

**Proposed new alerts:**

```yaml
- alert: VmstorageReadOnlyMode
  expr: vm_storage_is_read_only == 1
  for: 5m
  labels:
    severity: warning
  annotations:
    grafana: "https://grafana.linode.com/d/oS7Bi_0Wz?viewPanel=113&var-instance={{ $labels.instance }}"
    runbook: 'https://bits.linode.com/pages/ops/prometheus_rules/runbooks/sre-o11y/victoriametricslts/#diskrunsoutofspace--diskrunsoutofspacein3days'
    summary: "{{ $externalLabels.cluster }}: vmstorage {{ $labels.instance }} entered read-only mode"
    description: "vmstorage {{ $labels.instance }} has entered read-only mode due to low disk space. Writes are being rerouted to other pods automatically. The pod will recover when free space rises above the configured threshold."

- alert: TooManyVmstorageReadOnly
  expr: |
    count(vm_storage_is_read_only == 1) /
    count(vm_storage_is_read_only) > 0.15
  for: 10m
  labels:
    severity: critical
  annotations:
    grafana: "https://grafana.linode.com/d/oS7Bi_0Wz"
    runbook: 'https://bits.linode.com/pages/ops/prometheus_rules/runbooks/sre-o11y/victoriametricslts/#diskrunsoutofspace--diskrunsoutofspacein3days'
    summary: "{{ $externalLabels.cluster }}: More than 15% of vmstorage pods are in read-only mode"
    description: "A significant portion of vmstorage pods are read-only. The remaining pods are absorbing all writes, increasing their disk growth rate. Investigate disk usage and consider scaling."
```

The first fires when any single pod enters read-only mode — that's expected to happen occasionally and self-resolve. The second fires if many pods go read-only at once, which would indicate a cluster-wide issue.

---

### Strategy 4: Route Writes to Specific Pods (vminsert `-storageNode` Manipulation) — EVALUATED, NOT APPLICABLE NOW

The VictoriaMetrics docs suggest this approach for handling uneven disk after adding nodes. From the cluster docs (file `docs/victoriametrics/Cluster-VictoriaMetrics.md`, line 703):

> "In order to handle uneven disk space usage distribution after adding new vmstorage node it is possible to update vminsert configuration to route newly ingested metrics only to new storage nodes. Once disk usage will be similar configuration can be updated to include all nodes again. Note that vmselect nodes need to reference all storage nodes for querying."

The idea is: temporarily configure vminsert to only write to the new (empty) pods, while keeping vmselect pointed at all pods so reads work normally. Once disk usage equalizes, restore normal routing.

**Why we can't easily do this:** In a standard VictoriaMetrics deployment, you'd edit the `-storageNode` flag on vminsert directly. But we use the **VictoriaMetrics Operator**, which auto-generates the `-storageNode` list from the VMCluster CRD. You can't manually override `-storageNode` via `extraArgs` — the operator already sets it, and duplicate flags cause a crash.

**Alternative: `maintenanceInsertNodeIDs`:** The operator CRD does provide a field for this. From the CRD definition (`prometheus_rules/kubernetes/definitions/crds/vmrule-crd.yaml`, line 5215):

> "MaintenanceInsertNodeIDs - excludes given node ids from insert requests routing, must contain pod suffixes - for pod-0, id will be 0 and etc. lets say, you have pod-0, pod-1, pod-2, pod-3. to exclude pod-0 and pod-3 from insert routing, define nodeIDs: [0,3]."

So we could use `maintenanceInsertNodeIDs: [0,1,2,...,49]` to exclude the old pods from receiving writes, sending all new data to pods 50-91 only.

**Why we don't need it right now:** Our writes are already evenly distributed across all 92 pods, and the old pods are declining naturally. Excluding old pods from writes would:
- Concentrate all ingestion on 42 pods instead of 92, doubling their per-pod ingestion rate
- Those 42 pods would reach steady-state at ~54% instead of ~27%
- We'd need to carefully manage the re-enablement

This is a useful tool to know about, but it's solving a problem we don't have. The natural approach is already working.

**When this WOULD make sense:** If we had to add pods to a cluster where the old pods were at 90%+ and we needed the new pods to absorb most of the load immediately while old data expired. We're not in that situation.

---

### Strategy 5: Script-Based Pod Rotation — EVALUATED, NOT FEASIBLE

The original Jira ticket mentioned a pseudocode approach of scaling down old pods and scaling up new ones to replace high-usage pods with empty ones.

**Why this doesn't work with our setup:**

1. **StatefulSet mechanics:** vmstorage runs as a Kubernetes StatefulSet. Pods are ordered (pod-0 through pod-91). When you scale down, Kubernetes removes pods from the **highest ordinal** first. So scaling from 92 to 80 removes pods 80-91 — the newest, emptiest pods — not the old full ones. You can't selectively delete pod-5 while keeping pod-91.

2. **Data loss with replicationFactor=1:** Each pod holds unique data. Deleting a pod means permanently losing whatever time series it stored. With replicationFactor=1, there's no replica to fall back on.

3. **Operator control:** The VictoriaMetrics Operator manages the StatefulSet. Manual manipulation of pod ordinals would conflict with the operator's reconciliation loop.

**Verdict:** Not feasible. The approach fundamentally conflicts with how StatefulSets and our operator work.

---

### Strategy 6: Vertically Scale PVCs — EVALUATED, NOT RECOMMENDED

Increase PVC size from 6 TiB to something larger (8 or 10 TiB) for the pods approaching capacity.

This was already discussed and rejected in OP-19 (`terraform-observability-team/docs/content/proposals/OP-19-victoriametrics-scaling-vmstorage.md`, lines 103-105):

> "Vertically Scaling PVCs — Not recommended due to toil and 'VictoriaMetrics suggestion to run many small vmstorage nodes'"

The VictoriaMetrics docs reinforce this (file `docs/victoriametrics/Cluster-VictoriaMetrics.md`, line 796):

> "It is recommended to run a cluster with big number of small vmstorage nodes instead of a cluster with small number of big vmstorage nodes. This increases chances that the cluster remains available and stable when some of vmstorage nodes are temporarily unavailable during maintenance events such as upgrades, configuration changes or migrations."

Additional reasons not to do this:
- No pod is close to filling up (worst is 71.3%, declining)
- PVC resizing on Linode/Akamai block storage has operational overhead
- It treats the symptom, not the cause
- It goes against the horizontal scaling philosophy that the entire system is built around

**When this WOULD make sense:** If ingestion volume grew so large that even with many pods, each pod's steady-state usage exceeded what 6 TiB could hold. At current rates, that's not remotely close (steady-state is ~1.79 TB per pod on 6.6 TB PVCs).

---

## Monthly Compaction: Something to Be Aware Of

VictoriaMetrics runs a deduplication/merge process at the end of each calendar month. During this compaction, the process temporarily needs as much free space as the previous month's data occupies, before it can release the old blocks.

From the VictoriaMetrics docs (file `docs/victoriametrics/Cluster-VictoriaMetrics.md`, line 791):

> "on each vmstorage pod, the monthly final deduplication process will temporarily need as much space as is used for the previous month's data, before it can free up space. For example, if you have a 3 month retention period and you want to keep at least 10% space free at all times, you could pick 35% of your total space as value. When some of your vmstorage pods are in read-only mode, the remaining pods will have a higher share of the total data ingestion, and will therefore need more free space the next month."

For our worst-case pods (71% usage, ~4.7 TB data, 6.6 TB PVC):
- One month of data on an old pod ≈ 4.7 TB / ~13 months of retention ≈ **~360 GB**
- Free space available: ~1.9 TB
- **That's over 5x the compaction requirement** — plenty of headroom

This isn't a problem today, but it's why the `storage.minFreeDiskSpaceBytes` threshold should be set conservatively (20%, not 5%) — you need room for compaction to complete, not just room for new data.

The docs also flag a cascading effect: if pods go read-only, the remaining pods absorb more writes and need more free space for next month's compaction. In our setup, even in the worst case, the newer pods at 5% have massive capacity to absorb this.

---

## Things We Might Be Missing

We want to be transparent about what we verified and what we're assuming:

**Verified:**
- Disk usage percentages across ord2, lax3, mad2, osa1 (live Prometheus queries)
- Ingestion rate distribution (confirmed even across all pods)
- All Helm config values (read every values file, verified every line number)
- replicationFactor = 1 on every cluster (checked all 8 values files)
- VictoriaMetrics source code behavior for read-only mode, rerouting, flag defaults

**Not verified (should be checked by the team):**
- **sto2 and cgk1** pod counts and disk usage — we assume they mirror mad2 and osa1 respectively, but should confirm
- **sea1 and iad3 (staging)** — smaller PVCs (1.8 TiB), shorter retention (180d), should verify independently
- **Actual pod behavior during read-only transitions** — we've read the code, but we haven't tested this in production or staging. Worth doing a controlled test in staging: set a very high `minFreeDiskSpaceBytes` on one pod to force it into read-only and observe vminsert behavior
- **Monthly compaction impact on our specific data** — our calculation is based on uniform distribution over the retention period. Actual monthly data sizes may vary with ingestion spikes
- **ArgoCD sync behavior** — we need to confirm whether the current template drift is because ArgoCD has auto-sync disabled, or because the operator's OnDelete strategy is preventing scale-down. Either way, the template should be fixed
- **Whether the operator will actually pass through `storage.minFreeDiskSpaceBytes`** — we know the operator crashes on `-storageNode` override, but `storage.minFreeDiskSpaceBytes` goes into vmstorage `extraArgs`, not vminsert. This should work but should be tested in staging first

---

## What We Should Do

### Now (as part of this ticket)

1. **Fix the Helm template replicaCounts** to match what's actually running (92 for NA, 48 for EU, 48 for AP). This is the most critical change — it prevents accidental scale-down.

2. **Add `storage.minFreeDiskSpaceBytes` to vmstorage extraArgs** in the Helm template. Set to 20% of PVC size (production: `1319413953331`, staging: `395824185999`). Test in staging first.

3. **Add `VmstorageReadOnlyMode` and `TooManyVmstorageReadOnly` alerts** to the vmcluster alert rules.

### Not Now (no pods are in danger)

4. **Do NOT add more vmstorage pods.** Current pod counts provide ~2.5x headroom at steady-state. Adding pods unnecessarily wastes infrastructure and creates more hash redistribution.

5. **Do NOT implement write routing manipulation, pod rotation, or PVC resizing.** These are either infeasible, unnecessary, or both.

### Have Ready for the Future

6. **If a specific pod is critically full** and the read-only safety net isn't enough, use `maintenanceInsertNodeIDs` in the VMCluster CRD to exclude it from writes while keeping it available for reads.

7. **If total ingestion grows significantly** and steady-state per-pod usage approaches 60-70%, scale out with additional even-numbered pods per the [scaling guide](https://bits.linode.com/pages/ops/terraform-observability-team/services/victoria-metrics/victoriametrics-scaling-guide/). The guide (line 67-71) recommends maintaining a 1:2 vminsert:vmstorage ratio, and (lines 73-76) keeping vmstorage pods at even numbers.

8. **If replicationFactor is ever restored to 2**, re-evaluate all of the above — storage requirements double, and `-dropSamplesOnOverload` becomes safer to consider as an option.

---

## Summary Table

| Strategy | Status | Reason |
|----------|--------|--------|
| Natural equalization (wait) | **Recommended** | Working now, math confirms convergence to ~27% |
| `storage.minFreeDiskSpaceBytes` | **Implement now** | Zero-cost safety net, aligns with VM best practices |
| Read-only mode alerts | **Implement now** | Required if we set the safety net |
| Fix template replicaCounts | **Implement now** | Prevents accidental scale-down (critical) |
| vminsert write routing | Hold for future | Not needed; useful emergency tool via `maintenanceInsertNodeIDs` |
| Pod rotation scripts | Rejected | Incompatible with StatefulSet mechanics and replicationFactor=1 |
| PVC vertical scaling | Rejected | Against VM guidance, unnecessary, operationally expensive |

---

## References

| Document | Path / URL |
|----------|-----------|
| OP-06: VictoriaMetrics Prod Infra | `terraform-observability-team/docs/content/proposals/OP-06-victoriametrics-prod-infra.md` |
| OP-19: VictoriaMetrics Scaling | `terraform-observability-team/docs/content/proposals/OP-19-victoriametrics-scaling-vmstorage.md` |
| Scaling Guide | `terraform-observability-team/docs/content/Services/VictoriaMetrics/victoriametrics-scaling-guide.md` |
| Cluster Upgrade MOP (Jan 2025) | `terraform-observability-team/docs/content/mops/victoriametrics-cluster-upgrade.md` |
| Helm Template | `o11y-helm-charts/charts/o11y-helm-charts/templates/_vm-lts-helpers.tpl` |
| Alert Rules | `prometheus_rules/kubernetes/infra-victoriametrics/prometheus/vmcluster.yaml` |
| VMCluster CRD (maintenanceInsertNodeIDs) | `prometheus_rules/kubernetes/definitions/crds/vmrule-crd.yaml` (line 5215) |
| VM Source: minFreeDiskSpaceBytes flag | `app/vmstorage/main.go:63` (VictoriaMetrics v1.126.0) |
| VM Source: read-only error | `app/vmstorage/main.go:186-197` |
| VM Source: read-only metric | `app/vmstorage/main.go:491-495` |
| VM Source: disk space watcher | `lib/storage/storage.go:795-817` |
| VM Docs: Read-Only Mode | `docs/victoriametrics/Cluster-VictoriaMetrics.md:571-578` / [upstream](https://docs.victoriametrics.com/victoriametrics/cluster-victoriametrics/#readonly-mode) |
| VM Docs: Capacity Planning | `docs/victoriametrics/Cluster-VictoriaMetrics.md:785-791` / [upstream](https://docs.victoriametrics.com/victoriametrics/cluster-victoriametrics/#capacity-planning) |
| VM Docs: vminsert Flags | `docs/victoriametrics/vminsert_flags.md:43-48` / [upstream](https://docs.victoriametrics.com/victoriametrics/cluster-victoriametrics/#list-of-command-line-flags-for-vminsert) |
| VM Docs: Adding vmstorage nodes | `docs/victoriametrics/Cluster-VictoriaMetrics.md:697-703` |
