# Teleport Security and Access Control Guide for O11y Infrastructure

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [What is Teleport?](#what-is-teleport)
3. [Architecture Overview](#architecture-overview)
4. [Security Model](#security-model)
5. [Access Control & RBAC](#access-control--rbac)
6. [How Operators Access Clusters](#how-operators-access-clusters)
7. [Integration with O11y Security Controls](#integration-with-o11y-security-controls)
8. [Vault Integration](#vault-integration)
9. [Operational Procedures](#operational-procedures)
10. [Troubleshooting](#troubleshooting)
11. [References](#references)

---

## Executive Summary

Teleport is the **primary access mechanism** for all O11y Kubernetes clusters, providing secure, certificate-based authentication and auditable access to infrastructure. It replaces traditional kubectl authentication with a unified access plane that supports:

- **GitHub SSO authentication** - No shared credentials or SSH keys
- **Short-lived certificates** - 12-hour TTL with automatic expiration
- **Reverse tunnel architecture** - No need to expose Kubernetes APIs publicly
- **Complete audit logging** - Every kubectl command is logged with user identity
- **Role-based access control** - Fine-grained permissions based on GitHub team membership

### Key Facts

| Component | Details |
|-----------|---------|
| **Deployment Model** | Control plane on management clusters (o11y-apps), agents on regional clusters |
| **Authentication** | GitHub SSO with certificate-based access (12h TTL) |
| **Production Endpoint** | `teleport.infra-o11y-apps.iad3.us.prod.linode.com` |
| **Staging Endpoint** | `teleport.infra-o11y-apps.rin1.us.staging.linode.com` |
| **Security Model** | mTLS, certificate-based auth, reverse tunnels, RBAC |
| **Break-Glass Access** | Admin kubeconfigs stored in Vault at `infra/{env}/sre-o11y/{cluster-name}` |

---

## What is Teleport?

Teleport is an **identity-aware, multi-protocol access proxy** that provides:

### Core Capabilities

1. **Certificate-based authentication** - Eliminates shared secrets and SSH keys
2. **Single Sign-On (SSO)** - Seamless GitHub authentication integration
3. **Multi-protocol support** - SSH, Kubernetes API, HTTPS, databases
4. **Audit logging** - Complete session recording and replay capabilities
5. **Short-lived certificates** - Automatic certificate expiration (12-hour default)
6. **Role-based access control (RBAC)** - Fine-grained permissions management
7. **Zero-trust access** - Identity verification for every connection

### Why Teleport?

| Traditional Kubectl Access | Teleport Access |
|---------------------------|-----------------|
| Long-lived tokens/certs in kubeconfig | Short-lived certificates (12h) from GitHub SSO |
| Static file (`~/.kube/config`) | Dynamic certificate (`~/.tsh/keys/`) |
| Easy to copy/share (security risk) | Impossible to share (tied to user identity) |
| Cluster secret rotation required for revocation | Instant revocation via Teleport |
| Limited audit context in K8s logs | Full session recording with user identity |
| Multiple kubeconfig contexts | Single `tsh kube login` command |
| Requires VPN or public endpoint | Reverse tunnel (works behind NAT) |
| No 2FA/MFA support | GitHub MFA + optional TOTP |
| Manual certificate rotation | Automatic re-authentication |

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Teleport Architecture                           │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────┐
│   Engineer   │
│  (tsh CLI /  │
│   Web UI)    │
└──────┬───────┘
       │ 1. Authenticate with GitHub SSO
       │ 2. Request kubectl access
       ▼
┌────────────────────────────────────────────────────────────────┐
│ Teleport Cluster (Management Cluster)                          │
│ ┌────────────────┐         ┌──────────────────┐               │
│ │ Teleport Proxy │◄────────┤ Teleport Auth    │               │
│ │ - Web UI       │         │ - Cert Authority │               │
│ │ - API Gateway  │         │ - RBAC Engine    │               │
│ │ - LoadBalancer │         │ - Audit Log      │               │
│ └────────┬───────┘         └──────────────────┘               │
│          │                                                     │
│          │ teleport.{fqdn}:443 (multiplexed)                  │
└──────────┼─────────────────────────────────────────────────────┘
           │
           │ 3. Reverse tunnel (mTLS)
           │    Port 3024
           ▼
┌────────────────────────────────────────────────────────────────┐
│ Regional K8s Cluster                                           │
│ ┌──────────────────────────────────────┐                      │
│ │ Teleport Kube Agent                  │                      │
│ │ - Registers cluster with Teleport    │                      │
│ │ - Proxies kubectl API requests       │                      │
│ │ - Enforces RBAC policies             │                      │
│ │ - 3 replicas (HA)                    │                      │
│ └────────────┬─────────────────────────┘                      │
│              │                                                 │
│              │ 4. kubectl API requests                        │
│              ▼                                                 │
│ ┌──────────────────────────────────────┐                      │
│ │ Kubernetes API Server                │                      │
│ └──────────────────────────────────────┘                      │
└────────────────────────────────────────────────────────────────┘
```

### Deployment Model

#### Teleport Cluster (Control Plane)

**Location:** Management clusters (o11y-apps)

| Environment | Cluster Name | Teleport FQDN | Purpose |
|-------------|--------------|---------------|---------|
| Staging | o11y-apps-rin1-us-staging | `teleport.infra-o11y-apps.rin1.us.staging.linode.com` | Staging clusters access |
| Production | o11y-apps-iad3-us-prod | `teleport.infra-o11y-apps.iad3.us.prod.linode.com` | Production clusters access |

**Components:**
- **Teleport Auth Service** - Certificate authority and RBAC engine
- **Teleport Proxy Service** - Public-facing gateway (LoadBalancer + external-dns)
- **Teleport Operator** - Kubernetes operator for managing Teleport resources

**Configuration Details:**
- **Chart:** `teleport-cluster` v14.0.3
- **Chart mode:** Standalone (all-in-one deployment)
- **Proxy listener mode:** Multiplex (all protocols on port 443)
- **High availability:** 1 auth replica, 2 proxy replicas
- **Image:** Custom from Linode Artifactory: `linode-docker.artifactory.linode.com/o11y/teleport`
- **Authentication:** GitHub SSO
- **External DNS:** Automatic DNS record creation via external-dns

#### Teleport Kube Agent (Data Plane)

**Location:** Every regional Kubernetes cluster (Victoria Metrics, Prometheus, etc.)

**Purpose:**
- Registers the Kubernetes cluster with Teleport control plane
- Proxies kubectl API requests from authenticated users
- Enforces RBAC policies defined in Teleport
- Provides audit logging for all kubectl commands

**Configuration Details:**
- **Chart:** `teleport-kube-agent` v14.0.3
- **Role:** `kube` (Kubernetes access only)
- **High availability:** 3 replicas with pod anti-affinity
- **Join method:** Token-based authentication (tokens stored in Vault)
- **TLS:** Uses Vault-issued Linode CA certificates (`vault-linode-ca`)
- **Monitoring:** Prometheus PodMonitor enabled (30s scrape interval)
- **Labels:** Automatically tagged with `environment` and `datacenter`

**Configuration Files:**
- Teleport Cluster: `o11y-helm-charts/charts/o11y-helm-charts/templates/o11y-apps-applications/teleport-cluster-application.yaml`
- Teleport Kube Agent: `o11y-helm-charts/charts/o11y-helm-charts/templates/multi-cluster-applications/teleport-kube-agent-application.yaml`
- Proxy Helper: `o11y-helm-charts/charts/o11y-helm-charts/templates/_helpers.tpl` (lines 164-171)

---

## Security Model

### Defense-in-Depth Architecture

Teleport is part of a comprehensive **defense-in-depth** security architecture for O11y clusters:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    External Clients (Prometheus, Grafana, etc.)     │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                         │ Layer 1: Source IP Check
                         ▼
        ┌────────────────────────────────────────┐
        │   Linode Cloud Firewall (CFW)          │
        │   - IP Allowlist (IPv4 + IPv6)         │
        │   - Applied to NodeBalancer/NIL        │
        └────────────────┬───────────────────────┘
                         │
                         │ Layer 2: Load Balancing
                         ▼
        ┌────────────────────────────────────────┐
        │   Linode NodeBalancer (NIL)            │
        │   - Proxy Protocol v2                  │
        │   - ExternalTrafficPolicy: Local       │
        └────────────────┬───────────────────────┘
                         │
                         │ Layer 3: TLS Termination + Client Cert
                         ▼
        ┌────────────────────────────────────────┐
        │   Traefik Ingress Controller           │
        │   - mTLS with RequireAndVerifyClientCert│
        │   - Multiple CA trust anchors          │
        └────────────────┬───────────────────────┘
                         │
                         │ Layer 4: Network Policy Enforcement
                         ▼
        ┌────────────────────────────────────────┐
        │   Cilium CNI with Host Firewall        │
        │   - BPF-based packet filtering         │
        │   - Identity-based policies            │
        └────────────────┬───────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────────────┐
        │   Kubernetes API / Victoria Metrics    │
        └────────────────────────────────────────┘
```

### Certificate-Based Authentication

Teleport eliminates shared secrets (SSH keys, kubeconfig tokens) with short-lived certificates:

#### Certificate Lifecycle

```
1. User authenticates with GitHub SSO
   ↓
2. Teleport Auth issues X.509 client certificate
   ↓
3. Certificate includes:
   - User principal (GitHub username)
   - Roles and permissions
   - Kubernetes clusters allowed
   - Expiration timestamp (default: 12 hours)
   ↓
4. Certificate automatically expires
   ↓
5. User must re-authenticate to get new certificate
```

#### Benefits of Certificate-Based Auth

- **No long-lived credentials to steal** - Certificates expire automatically
- **Automatic credential rotation** - No manual rotation needed
- **Cannot be shared** - Tied to user identity via GitHub SSO
- **Instant revocation** - Revoke via Teleport (no need to rotate cluster secrets)
- **Strong authentication** - GitHub MFA enforced
- **Full audit trail** - Every certificate issuance logged

### Mutual TLS (mTLS)

All communication between Teleport components uses **mutual TLS**:

#### mTLS Implementation

**Teleport Proxy ↔ Teleport Auth:**
- Both sides present certificates
- Mutual verification enforced
- CA: Vault Linode CA (`vault-linode-ca`)

**Teleport Proxy ↔ Teleport Kube Agent:**
- Reverse tunnel over mTLS (port 3024)
- Agent presents join token on first connection
- Subsequent connections use issued certificate
- CA: Vault Linode CA

**User ↔ Teleport Proxy:**
- TLS for web UI (port 443)
- Client certificate required for API access
- Server certificate issued by Vault or cert-manager

#### Shared Certificate Infrastructure

Teleport shares the **Vault Linode CA** with other O11y services:
- Traefik ingress mTLS
- Envoy cross-cluster communication
- VictoriaMetrics endpoints
- Consistent trust model across infrastructure

### Reverse Tunnel Architecture

Teleport Kube Agents initiate **outbound connections** to the Teleport Proxy, eliminating inbound firewall requirements:

#### How Reverse Tunnels Work

1. Kube Agent connects to Teleport Proxy on port **3024** (reverse tunnel port)
2. Connection authenticated with join token (stored in Vault)
3. Persistent mTLS tunnel established
4. Proxy forwards kubectl requests through this tunnel
5. Agent proxies requests to local Kubernetes API server

#### Benefits of Reverse Tunnels

- **Clusters behind NAT can be accessed** - No public IP required
- **No need to expose Kubernetes API publicly** - API remains internal
- **Centralized access control** - All access goes through Teleport proxy
- **Works across cloud providers and VPCs** - No VPN required
- **Firewall-friendly** - Only outbound connections needed

---

## Access Control & RBAC

### Role-Based Access Control (RBAC)

Teleport RBAC is **separate** from Kubernetes RBAC, providing an additional security layer:

#### RBAC Flow

```
1. User authenticates to Teleport with GitHub credentials
   ↓
2. Teleport assigns roles based on GitHub team membership
   ↓
3. Roles define:
   - Which Kubernetes clusters user can access
   - Which namespaces are allowed
   - What kubectl commands are permitted
   ↓
4. Teleport enforces policies BEFORE reaching Kubernetes API
   ↓
5. Kubernetes RBAC provides additional layer (defense in depth)
```

#### Example RBAC Configuration

**Scenario: SRE Observability Team Member**

```
1. User: member of GitHub team "sre-observability"
   ↓
2. Teleport role: "o11y-admin" (assigned to sre-observability team)
   ↓
3. Role permissions:
   - kubernetes_groups: ["system:masters"]
   - kubernetes_labels:
       environment: ["prod", "staging"]
   ↓
4. User can access: All prod and staging o11y clusters
   ↓
5. User kubectl access: Full admin (system:masters)
   ↓
6. Kubernetes RBAC: Validates system:masters has permissions
```

#### Teleport Role Example

```yaml
apiVersion: resources.teleport.dev/v1
kind: TeleportRole
metadata:
  name: o11y-readonly
spec:
  allow:
    kubernetes_groups:
      - view
    kubernetes_labels:
      environment: ["prod", "staging"]
    kubernetes_resources:
      - kind: pod
        namespace: "o11y-system"
        name: "*"
```

#### GitHub Team Mapping

```yaml
apiVersion: resources.teleport.dev/v1
kind: TeleportGithubConnector
metadata:
  name: github
spec:
  teams_to_roles:
    - organization: linode
      team: sre-observability
      roles:
        - o11y-admin
        - kubernetes-admin
```

### Audit Logging

Teleport logs **every kubectl command** with complete context:

#### Logged Information

- User identity (GitHub username)
- Source IP address
- Timestamp (UTC)
- Kubernetes cluster accessed
- Namespace and resource
- kubectl command and arguments
- Success/failure status
- Session duration

#### Audit Log Storage

- **Primary storage:** Teleport Auth pod filesystem
- **Retention:** Configurable (default: 90 days)
- **Session replay:** Interactive kubectl sessions can be replayed in web UI
- **Optional forwarding:** Can forward to external SIEM for compliance

---

## How Operators Access Clusters

### Step-by-Step Access Guide

#### 1. Install tsh CLI

Download and install the Teleport client (`tsh`):

- **Installation docs:** https://goteleport.com/docs/installation/
- **Supported platforms:** Linux, macOS, Windows

#### 2. Authenticate to Teleport

**For Production Clusters:**

```bash
tsh login -k no --proxy=teleport.infra-o11y-apps.iad3.us.prod.linode.com
```

**For Staging Clusters:**

```bash
tsh login -k no --proxy=teleport.infra-o11y-apps.rin1.us.staging.linode.com
```

**What Happens:**
1. Browser opens with GitHub OAuth login
2. Authenticate with GitHub credentials (MFA enforced)
3. Teleport issues 12-hour client certificate
4. Certificate stored in `~/.tsh/keys/`

#### 3. List Available Clusters

```bash
tsh kube ls
```

**Example Output:**
```
Kube Cluster Name                    Labels
------------------------------------ --------------------------------------
victoriametrics-ord2-us-prod         environment=prod datacenter=ord2-us
vmselect-ord2-us-prod                environment=prod datacenter=ord2-us
prometheus-osa1-jp-prod              environment=prod datacenter=osa1-jp
o11y-apps-iad3-us-prod               environment=prod datacenter=iad3-us
```

#### 4. Login to Specific Cluster

```bash
tsh kube login -k no --set-context-name "{{.KubeName}}" victoriametrics-ord2-us-prod
```

**Pro Tips:**

**Login to all clusters at once:**
```bash
tsh kube login -k no --set-context-name '{{.KubeName}}' --all
```

**Use friendly context names:**
```bash
tsh kube login --set-context-name 'vm-prod-ord2' victoriametrics-ord2-us-prod
```

#### 5. Use kubectl Normally

```bash
kubectl get nodes
kubectl get pods -n o11y-system
kubectl logs -n o11y-system deploy/victoriametrics-vmselect
```

**Behind the scenes:**
- kubectl sends request to `teleport.{fqdn}:443`
- Teleport validates your certificate
- Teleport checks RBAC permissions
- Teleport logs the request
- Teleport forwards via reverse tunnel to Kube Agent
- Kube Agent proxies to Kubernetes API
- Response returned to you

### Teleport Web UI Access

Alternatively, access clusters via the Teleport web interface:

**Production:**
- URL: https://teleport.infra-o11y-apps.iad3.us.prod.linode.com
- Lists all available clusters
- Click cluster name to get connection instructions
- Can copy `tsh` commands directly

**Staging:**
- URL: https://teleport.infra-o11y-apps.rin1.us.staging.linode.com

### Shell Aliases for Easy Switching

Since staging and production use different Teleport instances:

**Bash/ZSH Aliases:**
```bash
alias tsh-stage='tsh login -k no --proxy=teleport.infra-o11y-apps.rin1.us.staging.linode.com'
alias tsh-prod='tsh login -k no --proxy=teleport.infra-o11y-apps.iad3.us.prod.linode.com'
```

**TSH Native Aliases:**
```yaml
# ~/.tsh/config/config.yaml
aliases:
  prod: "tsh login -k no --proxy=teleport.infra-o11y-apps.iad3.us.prod.linode.com:443 teleport.infra-o11y-apps.iad3.us.prod.linode.com"
  staging: "tsh login -k no --proxy=teleport.infra-o11y-apps.rin1.us.staging.linode.com:443 teleport.infra-o11y-apps.rin1.us.staging.linode.com"
```

Then use: `tsh prod` or `tsh staging`

### Certificate Status and Renewal

**Check certificate status:**
```bash
tsh status
```

**Example output:**
```
> Profile URL:        https://teleport.infra-o11y-apps.iad3.us.prod.linode.com:443
  Logged in as:       sgour
  Cluster:            teleport.infra-o11y-apps.iad3.us.prod.linode.com
  Roles:              o11y-admin, kubernetes-admin
  Logins:             sgour
  Kubernetes:         enabled
  Valid until:        2025-11-27 23:45 UTC [valid for 11h30m0s]
  Extensions:         permit-agent-forwarding, permit-port-forwarding
```

**Renew certificate (if expired):**
```bash
# Re-login to Teleport
tsh login --proxy=teleport.infra-o11y-apps.iad3.us.prod.linode.com

# Re-login to cluster
tsh kube login <cluster-name>
```

---

## Integration with O11y Security Controls

### Security Control Layers

Teleport is part of a **multi-layered security architecture**:

| Layer | Control | Purpose | Config Location |
|-------|---------|---------|-----------------|
| **Network (L3/L4)** |
| 1 | Cloud Firewall (CFW) | IP allowlisting on NodeBalancers | `traefik-application.yaml` |
| 2 | Cilium Host Firewall | BPF-based packet filtering on nodes | `cilium-application.yaml` |
| 3 | Cilium Network Policies | Pod-to-pod access control | `cilium-application.yaml` |
| **Application (L7)** |
| 4 | Traefik mTLS | Certificate-based ingress authentication | `traefik-application.yaml` |
| 5 | Teleport | Identity-aware cluster access | `teleport-cluster-application.yaml` |
| 6 | Envoy mTLS | Cross-cluster VictoriaMetrics encryption | `vmselect-envoy` chart |
| **Identity & Access** |
| 7 | GitHub SSO | User authentication | Teleport config |
| 8 | Vault PKI | Certificate issuance and rotation | `terraform-vault-config` |
| 9 | Vault AppRole | Kubernetes service authentication | Vault config |

### How Teleport Fits into Data Flow

**Example: Engineer accessing Victoria Metrics cluster**

```
1. Engineer runs: tsh login --proxy=teleport.infra-o11y-apps.iad3.us.prod.linode.com
   ↓
2. GitHub SSO authentication (with MFA)
   ↓
3. Teleport Auth issues client certificate (12h TTL)
   ↓
4. Engineer runs: tsh kube login victoriametrics-ord2-us-prod
   ↓
5. Engineer runs: kubectl get pods -n o11y-system
   ↓
6. kubectl → Teleport Proxy (TLS with client cert)
   ↓
7. Teleport Proxy validates:
   - Certificate signed by Teleport CA
   - Certificate not expired
   - User has access to cluster (RBAC check)
   ↓
8. Teleport Proxy logs the request (audit log)
   ↓
9. Teleport Proxy → Teleport Kube Agent (reverse tunnel, port 3024, mTLS)
   ↓
10. Teleport Kube Agent → Kubernetes API (localhost)
    ↓
11. Kubernetes RBAC validates user permissions
    ↓
12. Response returned through reverse path
    ↓
13. Engineer receives kubectl output
```

### Teleport + Traefik mTLS

For **external access** to Victoria Metrics endpoints (Prometheus remote_write):

```
Infra Prometheus
  ↓ (remote_write with client cert)
Cloud Firewall (IP allowlist check)
  ↓
NodeBalancer
  ↓
Traefik (mTLS termination - RequireAndVerifyClientCert)
  ↓
Cilium Network Policy
  ↓
Victoria Metrics vminsert
```

**Key Difference:**
- **Teleport:** For interactive human access (kubectl, SSH, etc.)
- **Traefik mTLS:** For automated system-to-system access (Prometheus scraping, remote_write, etc.)

### Certificate Monitoring

All certificates monitored for expiration:

**Monitoring Tools:**
- **grafana-client-cert-collector** - Monitors Grafana datasource client certificates
- **vault_pki_exporter** - Monitors Vault-issued certificates
- **Prometheus alerts** - Alert on certificates expiring within 30 days

**Certificates Monitored:**
- Teleport client certificates (user-issued, 12h TTL)
- Teleport CA certificates (Vault PKI)
- Traefik mTLS certificates
- Envoy cross-cluster certificates
- Kubernetes cluster CA certificates (yearly rotation)

**Alert Example:**
```yaml
- alert: TeleportCertificateExpiringSoon
  expr: |
    (teleport_certificate_expiry_seconds - time()) / 86400 < 30
  labels:
    severity: warning
  annotations:
    summary: "Teleport certificate expires in {{ $value }} days"
```

---

## Vault Integration

### Vault Secrets for Teleport

Teleport relies on **HashiCorp Vault** for secret management:

#### 1. Join Tokens

**Purpose:** Authenticate Teleport Kube Agents when joining Teleport cluster

**Storage Location:**
```
infra/{env}/sre-o11y/teleport-join-{cluster-name}
```

**Key:** `joinToken`

**Properties:**
- **TTL:** 1 hour (must be used within 1 hour of generation)
- **Single-use:** Token consumed on first successful join
- **Not base64 encoded:** Stored as plaintext (helm chart handles encoding)

**Example:**
```bash
# Store token in Vault
vault kv patch infra/prod/sre-o11y/teleport-join-victoriametrics-ord2-us-prod \
  joinToken=abcd1234567890-example-token
```

**Injection:** ArgoCD Vault Plugin injects token into Kube Agent deployment

#### 2. TLS Certificates

**Server Certificate:** `teleport-tls`
- Used for HTTPS on Teleport Proxy (port 443)
- Issued by Vault PKI or cert-manager
- Auto-renewal via cert-manager

**CA Certificate:** `vault-linode-ca`
- Root CA from Vault PKI
- Used to verify client certificates
- Shared with Traefik, Envoy, and other O11y services
- Mounted in Teleport pods as secret

**Certificate Secret Mounting:**
```yaml
tls:
  existingSecretName: teleport-tls       # Server cert
  existingCASecretName: vault-linode-ca  # CA cert for mTLS
```

#### 3. Admin Kubeconfigs (Break-Glass Access)

**Purpose:** Emergency access if Teleport is unavailable

**Storage Location:**
```
infra/{env}/sre-o11y/{cluster-name}
```

**Key:** `kubeconfig`

**Usage:**
```bash
# Retrieve kubeconfig from Vault
vault kv get -field kubeconfig infra/prod/sre-o11y/o11y-apps-iad3-us-prod > o11y-apps-prod.yaml

# Use with kubectl
kubectl --kubeconfig o11y-apps-prod.yaml get pods -n o11y-system
```

**Important:**
- **Only use in emergencies** (e.g., Teleport outage)
- Admin kubeconfigs bypass all Teleport audit logging
- Full admin access (system:masters)

### Vault AppRole for ArgoCD

Teleport applications managed by ArgoCD with Vault integration:

**Vault Authentication:**
- ArgoCD uses Vault AppRole
- JWT-based authentication from Kubernetes service accounts
- Short-lived tokens issued

**Secret Injection Workflow:**
```
1. ArgoCD detects Teleport application sync
   ↓
2. ArgoCD Vault Plugin reads Vault path
   ↓
3. Vault validates ArgoCD AppRole JWT
   ↓
4. Vault returns join token
   ↓
5. ArgoCD injects token into Helm values
   ↓
6. Teleport Kube Agent deployed with token
   ↓
7. Agent authenticates to Teleport cluster
```

### Secret Rotation Procedures

**Join Token Rotation:**
1. Generate new token: `tctl tokens add --ttl=1h --type=kube`
2. Store in Vault: `vault kv patch infra/{env}/sre-o11y/teleport-join-{cluster} joinToken={token}`
3. Sync ArgoCD application
4. Verify agent reconnects

**TLS Certificate Rotation:**
- Automatic via cert-manager
- Vault PKI TTL-based rotation
- No manual intervention required

**Admin Kubeconfig Rotation:**
- Documented in: `terraform-observability-team/docs/content/Services/Kubernetes Clusters/rotating-secrets.md`
- Requires cluster CA rotation (yearly process)

---

## Operational Procedures

### Connecting a New Kubernetes Cluster to Teleport

#### Prerequisites

- [ ] Cluster deployed and accessible
- [ ] ArgoCD managing the cluster
- [ ] Vault CLI access (see [Vault authentication docs](https://kb.linode.com/display/services/Vault+-+How-to+-+Authenticate+to+the+CLI))
- [ ] kubectl access to the **management cluster** where Teleport runs

#### Step 1: Determine Target Teleport Cluster

| Cluster Environment | Teleport Cluster | Management Cluster Context |
|---------------------|------------------|---------------------------|
| Staging | o11y-apps-rin1-us-staging | `o11y-apps-rin1-us-staging` |
| Production | o11y-apps-iad3-us-prod | `o11y-apps-iad3-us-prod` |

#### Step 2: Generate Join Token

Set kubecontext to the **management cluster**:

```bash
# For staging clusters
kubectl config use-context o11y-apps-rin1-us-staging

# For production clusters
kubectl config use-context o11y-apps-iad3-us-prod
```

Generate the join token:

```bash
kubectl exec -it deploy/teleport-cluster-auth -c teleport -n teleport-cluster -- tctl tokens add --ttl=1h --type=kube
```

**Example output:**
```
The invite token: abcd1234567890-example-token-do-not-use
```

**Important:**
- Token expires in **1 hour**
- Must be used before expiration
- Type must be `kube` for Kubernetes clusters
- Do **NOT** base64 encode (helm chart does this)

#### Step 3: Store Join Token in Vault

Authenticate to Vault:

```bash
vault login -method=ldap username=<your-username>
```

Store the token in Vault (plaintext, no base64 encoding):

```bash
# For staging
vault kv patch infra/staging/sre-o11y/teleport-join-<cluster-name> joinToken=<token-from-step-2>

# For production
vault kv patch infra/prod/sre-o11y/teleport-join-<cluster-name> joinToken=<token-from-step-2>
```

**Example:**
```bash
vault kv patch infra/prod/sre-o11y/teleport-join-victoriametrics-ord2-us-prod \
  joinToken=abcd1234567890-example-token-do-not-use
```

#### Step 4: Sync ArgoCD Application

Login to the ArgoCD instance managing the **target cluster**:

- Staging clusters: `https://argocd.{staging-cluster-fqdn}`
- Production clusters: `https://argocd.{prod-cluster-fqdn}`

Sync the Teleport Kube Agent application:

1. Navigate to: `teleport-kube-agent-{cluster-name}`
2. Click **Sync**
3. Wait for sync to complete
4. Verify all resources are green (Healthy + Synced)

#### Step 5: Verification

**Check Deployment Status:**

```bash
# Set context to target cluster
kubectl config use-context <cluster-name>

# Check teleport-kube-agent pods
kubectl get pods -n teleport-kube-agent

# Expected: 3 pods running (HA configuration)
```

**Test Access via Teleport:**

```bash
# Login to Teleport
tsh login --proxy=teleport.infra-o11y-apps.iad3.us.prod.linode.com

# List available clusters
tsh kube ls

# Expected: New cluster appears in list

# Login to the cluster
tsh kube login <cluster-name>

# Test kubectl access
kubectl get nodes
kubectl get pods -n o11y-system
```

**Check Teleport UI:**

1. Navigate to: `https://teleport.{management-cluster-fqdn}`
2. Go to **Resources** → **Servers**
3. Verify new cluster appears with correct labels:
   - `environment: {env}`
   - `datacenter: {dc}`

---

## Troubleshooting

### Agent Fails to Join

**Symptoms:**
- Pods in `CrashLoopBackOff`
- Logs show: "invalid join token" or "failed to connect to auth server"

**Diagnosis:**

```bash
kubectl logs -n teleport-kube-agent deployment/teleport-kube-agent
```

**Common Causes and Resolutions:**

| Issue | Symptom in Logs | Resolution |
|-------|-----------------|------------|
| Token expired | "invalid token" | Generate new token, update Vault, re-sync ArgoCD |
| Token base64 encoded | "failed to decode token" | Re-store token in plaintext in Vault |
| Wrong Vault path | "secret not found" | Verify path: `infra/{env}/sre-o11y/teleport-join-{cluster-name}` |
| Wrong token type | "invalid token type" | Regenerate with `--type=kube` |
| Network connectivity | "connection refused" | Check reverse tunnel port 3024 accessible |

### Cluster Not Showing in `tsh kube ls`

**Symptoms:**
- Agent pods running successfully
- Cluster doesn't appear in Teleport UI or `tsh kube ls` output

**Diagnosis:**

```bash
# Check agent logs
kubectl logs -n teleport-kube-agent deployment/teleport-kube-agent

# Check Teleport auth logs
kubectl logs -n teleport-cluster deployment/teleport-cluster-auth -c teleport
```

**Common Causes and Resolutions:**

| Issue | Resolution |
|-------|------------|
| `kubeClusterName` mismatch | Verify name in helm values matches registration |
| Agent not connected | Check reverse tunnel connection in agent logs |
| RBAC permissions issue | Verify Teleport roles grant access to cluster |
| Certificate issue | Check `existingCASecretName` is `vault-linode-ca` |
| Wrong proxy address | Verify `proxyAddr` in values matches environment |

### Certificate Expiration

**Symptoms:**
- `tsh kube login` fails with certificate error
- kubectl commands fail with: "Unable to connect to the server: x509: certificate has expired"

**Diagnosis:**

```bash
# Check certificate status
tsh status

# Look for "Valid until" field
```

**Resolution:**

```bash
# Re-login to Teleport to get fresh certificate
tsh login --proxy=teleport.infra-o11y-apps.iad3.us.prod.linode.com

# Re-login to the Kubernetes cluster
tsh kube login <cluster-name>

# Verify new certificate
tsh status
```

**Default certificate lifetime:** 12 hours

### Cannot Connect via kubectl

**Symptoms:**
- kubectl commands hang or timeout
- Error: "Unable to connect to the server: dial tcp: i/o timeout"

**Diagnosis Steps:**

1. **Check Teleport certificate:**
   ```bash
   tsh status
   # Verify "Valid until" hasn't expired
   ```

2. **Check kubeconfig context:**
   ```bash
   kubectl config current-context
   # Should show cluster name from Teleport
   ```

3. **Check Teleport proxy reachability:**
   ```bash
   curl -v https://teleport.infra-o11y-apps.iad3.us.prod.linode.com
   # Should return Teleport web UI
   ```

4. **Check Kube Agent health:**
   ```bash
   kubectl get pods -n teleport-kube-agent
   # All 3 pods should be Running
   ```

5. **Check reverse tunnel:**
   ```bash
   kubectl logs -n teleport-kube-agent deployment/teleport-kube-agent | grep "reverse tunnel"
   # Should show "connected" or "established"
   ```

**Common Resolutions:**
- Re-login to Teleport: `tsh login --proxy=...`
- Re-login to cluster: `tsh kube login <cluster-name>`
- Check network connectivity to Teleport proxy
- Verify Kube Agent is running and connected

### Teleport Outage - Break Glass Access

**Scenario:** Teleport control plane is down and you need emergency cluster access

**Resolution: Use Admin Kubeconfig from Vault**

```bash
# 1. Authenticate to Vault
vault login -method=ldap username=<your-username>

# 2. Retrieve admin kubeconfig
vault kv get -field kubeconfig infra/prod/sre-o11y/o11y-apps-iad3-us-prod > /tmp/emergency-kubeconfig.yaml

# 3. Use with kubectl
kubectl --kubeconfig /tmp/emergency-kubeconfig.yaml get pods -n teleport-cluster

# 4. Check Teleport control plane status
kubectl --kubeconfig /tmp/emergency-kubeconfig.yaml get pods -n teleport-cluster
kubectl --kubeconfig /tmp/emergency-kubeconfig.yaml logs -n teleport-cluster deploy/teleport-cluster-auth -c teleport
```

**IMPORTANT:**
- This bypasses all Teleport audit logging
- Only use in true emergencies
- Document usage in incident channel
- Revert to normal Teleport access ASAP

### Health Check Commands

**Kubernetes Health:**
```bash
kubectl get pods -n teleport-kube-agent
kubectl get pods -n teleport-cluster
```

**Teleport Status (from management cluster):**
```bash
kubectl exec -it deploy/teleport-cluster-auth -c teleport -n teleport-cluster -- tctl status

# Expected output:
# Cluster: teleport.{fqdn}
# Version: 14.0.3
# CA pins: sha256:xxxx
# Connected nodes: <number-of-clusters>
```

**User Certificate Status:**
```bash
tsh status

# Shows:
# - Logged in user
# - Certificate expiration
# - Available clusters
# - Current cluster context
```

**List Connected Clusters:**
```bash
kubectl exec -it deploy/teleport-cluster-auth -c teleport -n teleport-cluster -- tctl nodes ls

# Shows all registered Kube Agents
```

---

## References

### Configuration Files

**O11y Helm Charts Repository:**
- Teleport Cluster Application: `o11y-helm-charts/charts/o11y-helm-charts/templates/o11y-apps-applications/teleport-cluster-application.yaml`
- Teleport Kube Agent Application: `o11y-helm-charts/charts/o11y-helm-charts/templates/multi-cluster-applications/teleport-kube-agent-application.yaml`
- Proxy Helper Template: `o11y-helm-charts/charts/o11y-helm-charts/templates/_helpers.tpl` (lines 164-171)

**Values Files:**
- Production o11y-apps: `o11y-helm-charts/values/o11y-apps-iad3-us-prod`
- Staging o11y-apps: `o11y-helm-charts/values/o11y-apps-rin1-us-staging`

### Internal Documentation

**Terraform Observability Team Docs:**
- Connecting New Clusters: `terraform-observability-team/docs/content/Services/Kubernetes Clusters/teleport-kube-agent.md`
- Kubernetes Deployments Overview: `terraform-observability-team/docs/content/Services/Kubernetes Clusters/_index.md`
- Rotating Secrets: `terraform-observability-team/docs/content/Services/Kubernetes Clusters/rotating-secrets.md`
- Vault Authentication: https://kb.linode.com/display/services/Vault+-+How-to+-+Authenticate+to+the+CLI
- Teleport Access Guide: https://kb.linode.com/pages/viewpage.action?pageId=151726139

**Existing Comprehensive Docs:**
- Teleport Infrastructure Documentation: `/Users/sgour/teleport-o11y-infrastructure.md`
- O11y Kubernetes Security Controls: `/Users/sgour/o11y-k8s-security-controls_updated.md`

### External Documentation

**Official Teleport Docs:**
- Main Documentation: https://goteleport.com/docs/
- Kubernetes Access: https://goteleport.com/docs/kubernetes-access/introduction/
- Helm Chart Reference (teleport-kube-agent): https://goteleport.com/docs/reference/helm-reference/teleport-kube-agent/
- GitHub SSO: https://goteleport.com/docs/access-controls/sso/github/
- Access Requests: https://goteleport.com/features/access-requests/
- RBAC Configuration: https://goteleport.com/docs/access-controls/reference/

**Teleport Repository:**
- Main Repo: `/Users/sgour/teleport/`
- RBAC Examples: `/Users/sgour/teleport/examples/resources/role.yaml`
- OIDC Connector: `/Users/sgour/teleport/examples/resources/oidc-connector.yaml`
- Helm Chart Examples: `/Users/sgour/teleport/examples/chart/`

### Related Components

**Security & Certificate Management:**
- O11y Security Controls: `/Users/sgour/o11y-k8s-security-controls_updated.md`
- Vault PKI: `terraform-observability-team/docs/content/Services/Vault PKI Exporter/vault_pki_exporter.md`
- Certificate Monitoring: `terraform-observability-team/docs/content/Services/Grafana/Client Cert Monitoring.md`

**Monitoring & Observability:**
- Prometheus Metrics: Teleport Kube Agents expose metrics on `:3000/metrics`
- Key Metrics:
  - `teleport_connected_resources` - Connected clusters count
  - `teleport_reverse_tunnels_connected` - Active reverse tunnels
  - `teleport_failed_connect_to_node_attempts_total` - Connection failures
  - `teleport_failed_login_attempts_total` - Failed authentications
- Grafana Dashboard: "Teleport Agent Metrics" (created from PodMonitor)

### Support

**For Questions or Issues:**
- Slack: `#sre-observability`
- Incident Channel: `#o11y-inc-*`
- Teleport Upstream: https://github.com/gravitational/teleport/discussions

**Related Tickets:**
- OY-912: Update linode-ccm (affects Teleport networking)
- OY-658: Rework Victoria Metrics alerts (affects monitoring)

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2025-11-27 | Initial consolidated security and access control guide created | Claude Code |

---

## Appendix: Quick Reference Commands

### Common Teleport Operations

```bash
# Login to production Teleport
tsh login --proxy=teleport.infra-o11y-apps.iad3.us.prod.linode.com

# Login to staging Teleport
tsh login --proxy=teleport.infra-o11y-apps.rin1.us.staging.linode.com

# List available clusters
tsh kube ls

# Login to specific cluster
tsh kube login <cluster-name>

# Login to all clusters at once
tsh kube login --set-context-name '{{.KubeName}}' --all

# Check certificate status
tsh status

# Logout
tsh logout
```

### Troubleshooting Commands

```bash
# Check Teleport cluster status (from management cluster)
kubectl exec -it deploy/teleport-cluster-auth -c teleport -n teleport-cluster -- tctl status

# List connected clusters
kubectl exec -it deploy/teleport-cluster-auth -c teleport -n teleport-cluster -- tctl nodes ls

# Check Kube Agent health
kubectl get pods -n teleport-kube-agent
kubectl logs -n teleport-kube-agent deployment/teleport-kube-agent

# Generate new join token
kubectl exec -it deploy/teleport-cluster-auth -c teleport -n teleport-cluster -- tctl tokens add --ttl=1h --type=kube

# Get admin kubeconfig from Vault (emergency only)
vault kv get -field kubeconfig infra/prod/sre-o11y/<cluster-name> > emergency-kubeconfig.yaml
```

### Vault Commands

```bash
# Login to Vault
vault login -method=ldap username=<your-username>

# Store join token
vault kv patch infra/prod/sre-o11y/teleport-join-<cluster-name> joinToken=<token>

# Retrieve admin kubeconfig
vault kv get -field kubeconfig infra/prod/sre-o11y/<cluster-name>
```

---

**End of Document**
