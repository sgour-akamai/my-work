# O11y Kubernetes Security Controls Documentation

## Overview

This document describes the defense-in-depth security architecture for the Observability (O11y) Kubernetes clusters. Our infrastructure employs multiple layers of security controls to protect Victoria Metrics endpoints and ensure only authorized clients can access our monitoring infrastructure.

## Security Control Layers

Our security architecture implements three primary control layers:

1. **Cloud Firewall (CFW)** - Network-level IP allowlisting on NodeBalancers/NILs
2. **Cilium Network Policies** - Host-based firewall on Kubernetes nodes
3. **Mutual TLS (mTLS)** - Certificate-based authentication and encryption

Additionally, we use **Enterprise Application Access (EAA)** for certain management endpoints.

---

## Architecture Diagrams

### High-Level Security Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         External Clients                             │
│  (linmon, grafana, jumpboxes, EAA connectors, appbattery, etc.)    │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         │ 1. Source IP Check
                         ▼
        ┌────────────────────────────────────────┐
        │   Linode Cloud Firewall (CFW)          │
        │   - Managed by linode-ccm              │
        │   - IP Allowlist (IPv4 + IPv6)         │
        │   - Applied to NodeBalancer/NIL        │
        └────────────────┬───────────────────────┘
                         │ 2. Layer 4 Load Balancing
                         ▼
        ┌────────────────────────────────────────┐
        │   Linode NodeBalancer (NIL)            │
        │   - Premium tier in production         │
        │   - Proxy Protocol v2                  │
        │   - ExternalTrafficPolicy: Local       │
        └────────────────┬───────────────────────┘
                         │
                         │ 3. TLS Termination + Client Cert Check
                         ▼
        ┌────────────────────────────────────────┐
        │   Traefik Ingress Controller           │
        │   - mTLS with RequireAndVerifyClientCert│
        │   - CA certificate validation          │
        │   - Multiple CA trust anchors          │
        │   - 3 replicas (HA)                    │
        └────────────────┬───────────────────────┘
                         │
                         │ 4. Network Policy Enforcement
                         ▼
        ┌────────────────────────────────────────┐
        │   Cilium CNI with Host Firewall        │
        │   - BPF-based packet filtering         │
        │   - kubeProxyReplacement enabled       │
        │   - Identity-based policies            │
        │   - IPv4/IPv6 dual-stack               │
        └────────────────┬───────────────────────┘
                         │
                         │ 5. Application-level mTLS (for cross-cluster)
                         ▼
        ┌────────────────────────────────────────┐
        │   Envoy TCP Proxy (vmselect)           │
        │   - Upstream/Downstream mTLS           │
        │   - SDS for cert hot-reload            │
        │   - SAN validation                     │
        └────────────────┬───────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────────────┐
        │   Victoria Metrics Endpoints           │
        │   - vminsert (write path)              │
        │   - vmselect (read path)               │
        │   - vmstorage (storage layer)          │
        └────────────────────────────────────────┘
```

### Network Flow for Different Node Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Security Controls by Layer                        │
└─────────────────────────────────────────────────────────────────────┘

KUBERNETES NODES (Worker Nodes):
┌──────────────────────────────────────┐
│  Cilium CNI                          │
│  ├─ Host Firewall: ENABLED           │
│  ├─ BPF Packet Filtering             │
│  ├─ Network Policies                 │
│  ├─ Identity-based Access Control    │
│  └─ kubeProxy Replacement: true      │
└──────────────────────────────────────┘

NODEBALANCERS / NILs (Load Balancers):
┌──────────────────────────────────────┐
│  Linode Cloud Firewall (CFW)         │
│  ├─ Managed by: linode-ccm           │
│  ├─ IP Allowlist (dynamically built) │
│  │   ├─ Linmon IPs (IPv4 + IPv6)     │
│  │   ├─ Grafana instances            │
│  │   ├─ Jumpboxes                    │
│  │   ├─ EAA Connectors               │
│  │   ├─ AppBattery systems           │
│  │   ├─ GHAR systems                 │
│  │   └─ Other authorized sources     │
│  └─ Applied via annotation:          │
│      linode-loadbalancer-firewall-acl│
└──────────────────────────────────────┘

INGRESS LAYER (Traefik Pods):
┌──────────────────────────────────────┐
│  Mutual TLS (mTLS)                   │
│  ├─ Client Cert: REQUIRED            │
│  ├─ Verification: ENABLED            │
│  ├─ CA Certificates:                 │
│  │   ├─ vault-linode-ca-crt          │
│  │   ├─ akamai-devcloud-ca           │
│  │   └─ prometheus-gecko-ca          │
│  └─ TLS Options:                     │
│      clientAuth.RequireAndVerifyClientCert│
└──────────────────────────────────────┘
```

### Data Flow Example: Linmon → Victoria Metrics

```
                                    WRITE PATH
┌──────────┐
│  Linmon  │
│ Exporter │
└────┬─────┘
     │ remote_write with client certificate
     │ Source IP: <linmon-ipv4> or <linmon-ipv6>
     ▼
┌────────────────────────────────────────────────────────────────┐
│ Cloud Firewall Check                                           │
│                                                                │
│ IF source_ip IN allowlist:                                    │
│     ALLOW                                                      │
│ ELSE:                                                          │
│     DROP  ← THIS CAUSED THE INCIDENT ON 2024-06-04           │
│                (linmon IPs were missing from allowlist)       │
└────────────────────────┬───────────────────────────────────────┘
                         │ ALLOWED
                         ▼
┌────────────────────────────────────────────────────────────────┐
│ NodeBalancer                                                   │
│ - Receives connection on port 443                             │
│ - Proxy Protocol v2 preserves source IP                       │
│ - Forwards to Traefik pod (anti-affinity distributed)         │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│ Traefik TLS Termination                                        │
│                                                                │
│ 1. TLS handshake initiated                                    │
│ 2. Server presents certificate                                │
│ 3. Server requests CLIENT certificate (mTLS)                  │
│ 4. Client presents certificate                                │
│ 5. Traefik validates:                                         │
│    - Certificate is signed by trusted CA                      │
│    - Certificate is not expired                               │
│    - Certificate chain is valid                               │
│                                                                │
│ IF valid:                                                      │
│     Route to vminsert service                                 │
│ ELSE:                                                          │
│     Reject with TLS error                                     │
└────────────────────────┬───────────────────────────────────────┘
                         │ AUTHORIZED
                         ▼
┌────────────────────────────────────────────────────────────────┐
│ Cilium Network Policy                                          │
│ - Evaluates pod identity                                      │
│ - Checks L3/L4 policies                                       │
│ - Allows traffic to vminsert pods                            │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│ Victoria Metrics vminsert                                      │
│ - Receives Prometheus remote_write request                    │
│ - Processes and stores metrics in vmstorage                   │
└────────────────────────────────────────────────────────────────┘
```

---

## Detailed Component Documentation

### 1. Cloud Firewall (CFW) - Linode Managed

**Purpose:** Network-level access control at the load balancer boundary

**Implementation:**
- Managed by `linode-cloud-controller-manager` (linode-ccm)
- Applied to Linode NodeBalancers via service annotation
- Configured in Traefik helm chart

**Configuration Location:**
```
o11y-helm-charts/charts/o11y-helm-charts/templates/multi-cluster-applications/traefik-application.yaml
```

**Key Annotation:**
```yaml
service.beta.kubernetes.io/linode-loadbalancer-firewall-acl: |
  <IPv4 CIDR list>
  <IPv6 CIDR list>
```

**Allowlist Sources:**
The allowlist is dynamically built from multiple sources:

1. **Static sources** (git submodule: `vendor/salt-kvdata/ips/static/`):
   - `appbattery.json` - AppBattery monitoring systems

2. **Collected sources** (git submodule: `vendor/salt-kvdata-collected/ips/`):
   - `grafana.json` - Grafana instance IPs
   - `jumpbox.json` - Administrative jumpbox IPs
   - `eaa-connectors.json` - Enterprise Application Access connector IPs
   - `ghar.json` - GHAR system IPs

3. **Hardcoded sources** in values files:
   - Linmon IPv4 and IPv6 addresses
   - Anodot jump-box
   - Akamai prod-edw-proxy instances
   - Gen2Obj test instances
   - Other regional cluster endpoints

**Helper Functions:**
```
_traefik_helpers.tpl:
  - traefikCloudFirewallAllowlistv4
  - traefikCloudFirewallAllowlistv6
```

**Known Issues:**
- **CRITICAL:** linode-ccm has a bug preventing firewall rollbacks
  - Once annotation is applied, firewall persists even after annotation removal
  - Only the ACL rules can be modified, not the firewall itself
  - Workaround: Manually edit via Linode Cloud Manager UI
  - Upstream fix: In progress (PR pending)
  - Tracking: OY-912 (linode-ccm update needed)

**Incident Impact (2024-06-04):**
- Linmon IPs were missing from the allowlist after CFW rollout
- Resulted in 48 minutes of data loss (16:13 - 17:01 UTC)
- Rollback was blocked by linode-ccm bug
- Resolution: Added linmon IPs to allowlist + manual Cloud Manager fix

---

### 2. Cilium Network Policies

**Purpose:** Host-based firewall and network policy enforcement at the Kubernetes node level

**Implementation:**
- Full CNI replacement with `kubeProxyReplacement: true`
- eBPF-based packet filtering
- Identity-based access control via etcd kvstore
- Host firewall enabled for node-level protection

**Configuration Location:**
```
o11y-helm-charts/charts/o11y-helm-charts/templates/multi-cluster-applications/cilium-application.yaml
```

**Key Features:**
```yaml
hostFirewall:
  enabled: true

kubeProxyReplacement: true

bpf:
  masquerade: true

tunnel:
  protocol: vxlan

ipv4:
  enabled: true

ipv6:
  enabled: true

localRedirectPolicy: true
```

**Identity Allocation:**
- Backend: etcd cluster
- Enables identity-based network policies
- Supports cross-node policy enforcement

**Monitoring:**
- ServiceMonitor enabled for Prometheus scraping
- Metrics endpoint exposed for observability

**Benefits:**
- Lower latency than iptables-based solutions
- Better performance with eBPF
- Source IP preservation with `externalTrafficPolicy: Local`
- Fine-grained L3/L4/L7 policy enforcement

---

### 3. Mutual TLS (mTLS)

**Purpose:** Certificate-based authentication and encryption

**Implementation Layers:**

#### Layer 1: Traefik Ingress mTLS

**Configuration Location:**
```
o11y-helm-charts/charts/o11y-helm-charts/templates/_traefik_helpers.tpl
```

**TLS Options:**
```yaml
spec:
  clientAuth:
    secretNames:
      - vault-linode-ca-crt
      - akamai-devcloud-ca-crt
      - akamai-prod-ca-crt
      - prometheus-gecko-ca-crt
    clientAuthType: RequireAndVerifyClientCert
```

**Trusted CA Certificates:**
1. `vault-linode-ca-crt` - Primary Vault PKI CA
2. `akamai-devcloud-ca-crt` - Development cloud CA
3. `akamai-prod-ca-crt` - Production Akamai CA
4. `prometheus-gecko-ca-crt` - Gecko/Prometheus infrastructure CA

**Certificate Injection:**
- Certificates mounted as secrets
- Sourced from Vault PKI or cert-manager
- Automatic rotation supported

#### Layer 2: Envoy Proxy mTLS (Cross-Cluster Communication)

**Configuration Location:**
```
o11y-helm-charts/charts/vmselect-envoy/templates/configmap.yaml
o11y-helm-charts/charts/vmselect-envoy/templates/deployment.yaml
```

**Architecture:**
- **Global Select Mode:** Envoy proxies vmselect queries across regional clusters
- **LTS Mode:** Envoy proxies vmstorage communication with mTLS

**Downstream TLS (Client → Envoy):**
```yaml
require_client_certificate: true
validation_context:
  trusted_ca:
    filename: /etc/envoy/certs/ca.crt
```

**Upstream TLS (Envoy → Backend):**
```yaml
common_tls_context:
  tls_certificates:
    - certificate_chain: /etc/envoy/certs/tls.crt
      private_key: /etc/envoy/certs/tls.key
  validation_context:
    trusted_ca:
      filename: /etc/envoy/certs/ca.crt
    match_subject_alt_names:
      - exact: "victoriametrics-foo-us-dev.envoy.local"
      - exact: "victoriametrics-bar-us-dev.envoy.local"
```

**Certificate Hot-Reloading:**
- Uses Envoy Secret Discovery Service (SDS)
- Certificates reloaded without pod restart
- Watched directory: `/etc/envoy/certs/`

**Testing & Development:**
- Certificate generation via `certstrap`
- SAN validation testing
- Certificate rotation scenarios
- Documentation: `o11y-helm-charts/hack/envoy/README.md`

**Certificate Secret Mounting:**
```yaml
volumes:
  - name: tls-certs
    secret:
      secretName: vmselect-ingress-tls
  - name: ca-cert
    secret:
      secretName: vault-linode-ca
```

---

### 4. Enterprise Application Access (EAA)

**Purpose:** Identity-aware access control for management endpoints

**Implementation:**
- EAA connector IPs included in Cloud Firewall allowlist
- Used for administrative access to management clusters
- Provides additional identity verification layer

---

## Security Control Matrix

| Layer | Control | Applied To | Managed By | Config Location |
|-------|---------|------------|------------|-----------------|
| L3/L4 | Cloud Firewall | NodeBalancers/NILs | linode-ccm | traefik-application.yaml |
| L3/L4 | Cilium Host Firewall | Kubernetes Nodes | Cilium Agent | cilium-application.yaml |
| L3/L4 | Cilium Network Policies | Pod-to-Pod | Cilium Agent | cilium-application.yaml |
| L7 | Traefik mTLS | Ingress Traffic | Traefik | traefik-application.yaml |
| L7 | Envoy mTLS | Cross-Cluster | Envoy Proxy | vmselect-envoy chart |
| L7 | EAA | Management Endpoints | Akamai EAA | External |

---

## Operational Procedures

### Adding New Authorized Clients

To add a new client that needs to access Victoria Metrics endpoints:

1. **Obtain client IPs (IPv4 and IPv6)**

2. **Choose allowlist source:**
   - For permanent infrastructure: Add to salt-kvdata JSON files
   - For temporary/testing: Add to values file `extraSourceIPs`

3. **Update the appropriate values file:**
   ```
   o11y-helm-charts/values/<cluster>-values.yaml
   ```

4. **Add client certificate to trusted CAs** (if new CA):
   - Issue certificate from existing Vault PKI, OR
   - Add new CA certificate to Traefik TLS options

5. **Test in staging first:**
   - Apply changes to staging cluster
   - Validate connectivity from client
   - Check Traefik logs for TLS handshake success
   - Verify Cloud Firewall rules in Linode Cloud Manager

6. **Roll out to production:**
   - Merge PR with updated values
   - Monitor during rollout
   - Verify client connectivity
   - Check for alert firing

### Removing Client Access

1. **Remove IPs from allowlist:**
   - Update values file or salt-kvdata JSON
   - Commit and merge PR

2. **Revoke client certificates (if necessary):**
   - Use Vault PKI revocation API
   - Update CRL if distributed

3. **Deploy changes:**
   - Apply helm changes
   - **IMPORTANT:** Due to linode-ccm bug, firewall rules can only be modified, not removed
   - Verify removal in Cloud Manager

### Incident Response: Client Cannot Connect

**Symptoms:**
- Client reports connection timeout or TLS errors
- Remote_write lag alerts firing
- Data gaps in dashboards

**Diagnosis Steps:**

1. **Check Cloud Firewall:**
   ```bash
   # In Linode Cloud Manager, verify:
   # - Firewall is attached to NodeBalancer
   # - Client IP is in allowlist (both IPv4 and IPv6)
   # - No typos in CIDR notation
   ```

2. **Check Traefik logs:**
   ```bash
   kubectl logs -n o11y-system -l app.kubernetes.io/name=traefik | grep <client-ip>
   # Look for TLS handshake errors or certificate validation failures
   ```

3. **Verify client certificate:**
   ```bash
   # Check certificate validity
   openssl x509 -in client.crt -text -noout

   # Verify CA chain
   openssl verify -CAfile ca.crt client.crt
   ```

4. **Check Cilium policies:**
   ```bash
   cilium endpoint list
   cilium policy get
   ```

**Common Issues:**

| Issue | Symptom | Resolution |
|-------|---------|------------|
| Missing IPs in CFW allowlist | Connection timeout | Add IPs to allowlist, deploy |
| Expired client certificate | TLS handshake failure | Renew certificate |
| CA certificate not trusted | Certificate verification failed | Add CA to Traefik TLS options |
| linode-ccm rollback bug | Changes don't take effect | Manually edit in Cloud Manager |
| IPv6 missing | IPv6 clients blocked | Add IPv6 CIDR to allowlist |

---

## Testing & Validation

### Pre-Deployment Testing Checklist

Before rolling out CFW or mTLS changes:

- [ ] Changes tested in staging cluster
- [ ] All known client IPs included (IPv4 AND IPv6)
- [ ] Client certificates validated against CA
- [ ] Traefik TLS options include all necessary CAs
- [ ] Linmon IPs explicitly verified in allowlist
- [ ] Alert thresholds reviewed (avoid pager fatigue)
- [ ] Rollback plan documented (account for linode-ccm bug)
- [ ] Stakeholders notified in #sre-observability

### Post-Deployment Validation

After deploying changes:

- [ ] Check Cloud Firewall rules in Linode Cloud Manager
- [ ] Verify Traefik pods are healthy
- [ ] Test connectivity from each client type:
  - [ ] Linmon
  - [ ] Grafana
  - [ ] Jumpboxes
  - [ ] EAA connectors
- [ ] Monitor remote_write lag alerts
- [ ] Check for TLS errors in Traefik logs
- [ ] Verify no data gaps in dashboards

### Testing mTLS in Development

See: `o11y-helm-charts/hack/envoy/README.md`

1. Generate test certificates with certstrap
2. Configure Envoy with SDS
3. Test certificate rotation
4. Validate SAN matching
5. Test cross-cluster communication

---

## Alerting Strategy

### Current Alert Issues (Identified in Incident)

**Problem:** Remote_write lag alerts did not page for linmon outage

**Root Causes:**
- Alert evaluated lag to BOTH linmon-federation AND LTS
- linmon-federation was not behind, so aggregate alert didn't fire
- Low priority alert due to pager fatigue from Victoria Metrics alerts

**Recommendations:**

1. **Separate linmon alerts by destination:**
   ```yaml
   # Alert specifically for linmon → LTS lag
   - alert: LinmonRemoteWriteLagToLTS
     expr: |
       linmon_remote_write_lag{destination="victoria-metrics-lts"} > 300
     for: 5m
     labels:
       severity: high
     annotations:
       summary: "Linmon cannot write to LTS (potential CFW or mTLS issue)"
   ```

2. **Fix Victoria Metrics alert noise:**
   - Tracking: OY-658
   - Rework thresholds to reduce false positives
   - Increase `for` duration for transient issues
   - Adjust severity levels

3. **Add Cloud Firewall health checks:**
   ```yaml
   - alert: CloudFirewallBlockingTraffic
     expr: |
       rate(traefik_entrypoint_requests_total{code="0"}[5m]) > 10
     labels:
       severity: high
     annotations:
       summary: "Cloud Firewall may be blocking legitimate traffic"
   ```

4. **Add mTLS certificate expiration alerts:**
   ```yaml
   - alert: ClientCertificateExpiringSoon
     expr: |
       (traefik_tls_certs_not_after - time()) / 86400 < 30
     labels:
       severity: warning
     annotations:
       summary: "Client certificate expires in {{ $value }} days"
   ```

---

## Known Issues & Limitations

### 1. linode-ccm Firewall Rollback Bug

**Issue:** Once Cloud Firewall annotation is applied, firewall persists even after removing annotation

**Impact:**
- Cannot cleanly rollback firewall changes
- Must manually remediate via Cloud Manager
- Slows incident response

**Tracking:**
- OY-912: Update linode-ccm to latest version
- Upstream PR: In progress

**Workaround:**
1. In Linode Cloud Manager, set default inbound policy to "ACCEPT"
2. Update allowlist via annotation (can modify rules)
3. Re-enable restrictive policy

### 2. Manual IP Allowlist Management

**Issue:** IP allowlists are manually maintained in values files and JSON

**Impact:**
- Prone to human error (as seen in 2024-06-04 incident)
- Doesn't scale well
- No automated validation

**Future Improvements:**
- Automate IP discovery from infrastructure inventory
- Implement validation checks in CI/CD
- Consider service mesh for identity-based auth (move away from IP allowlists)

### 3. Alert Fatigue

**Issue:** Victoria Metrics alerts fire frequently at low priority

**Impact:**
- Real incidents missed (linmon outage alert didn't page)
- Engineers desensitized to alerts

**Action Items:**
- OY-658: Rework Victoria Metrics alerts
- Separate critical vs. warning alerts
- Implement alert grouping/deduplication

---

## References

### Configuration Files

- Traefik Application: `o11y-helm-charts/charts/o11y-helm-charts/templates/multi-cluster-applications/traefik-application.yaml`
- Traefik Helpers: `o11y-helm-charts/charts/o11y-helm-charts/templates/_traefik_helpers.tpl`
- Cilium Application: `o11y-helm-charts/charts/o11y-helm-charts/templates/multi-cluster-applications/cilium-application.yaml`
- linode-ccm: `o11y-helm-charts/charts/o11y-helm-charts/templates/multi-cluster-applications/linode-ccm-application.yaml`
- Envoy Config: `o11y-helm-charts/charts/vmselect-envoy/templates/configmap.yaml`
- Envoy Deployment: `o11y-helm-charts/charts/vmselect-envoy/templates/deployment.yaml`

### Values Files

- Production vmselect: `o11y-helm-charts/values/vmselect-ord2-us-prod`
- Production Victoria Metrics: `o11y-helm-charts/values/victoriametrics-ord2-us-prod`
- Production management cluster: `o11y-helm-charts/values/o11y-apps-iad3-us-prod`

### External Documentation

- Envoy mTLS setup: `o11y-helm-charts/hack/envoy/README.md`
- Incident retrospective: `#o11y-inc-global-vminsert-and-linmon-connectivity-issues-2024-06-04`

### Related Tickets

- OY-912: Update linode-ccm
- OY-658: Rework Victoria Metrics alerts
- (Track Cloud Firewall enable/disable feature request)

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2024-11-24 | Initial documentation created | Claude Code |

---

## Contact

For questions or issues with security controls:

- Slack: `#sre-observability`
- Incident channel: `#o11y-inc-*`
