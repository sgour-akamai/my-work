# Complete Onboarding Guide: terraform-observability-team Repository

## Welcome to the SRE Observability Team!

This guide will teach you everything you need to know about the `terraform-observability-team` repository as a new team member. No prior knowledge assumed - we'll start from the basics and go deep.

**Team Motto**: *Mors melius quam Nagios* ("Death is better than Nagios")

---

## Table of Contents

1. [What is This Repository?](#1-what-is-this-repository)
2. [What Does the Observability Team Do?](#2-what-does-the-observability-team-do)
3. [The Big Picture: Observability Architecture](#3-the-big-picture-observability-architecture)
4. [Repository Structure Explained](#4-repository-structure-explained)
5. [Key Services Deep Dive](#5-key-services-deep-dive)
6. [How Deployments Work](#6-how-deployments-work)
7. [Important Files You Need to Know](#7-important-files-you-need-to-know)
8. [Documentation Structure](#8-documentation-structure)
9. [Day-to-Day Operations](#9-day-to-day-operations)
10. [Common Tasks with Examples](#10-common-tasks-with-examples)
11. [Things to Be Careful About](#11-things-to-be-careful-about)
12. [Getting Started Checklist](#12-getting-started-checklist)
13. [Glossary](#13-glossary)

---

## 1. What is This Repository?

The `terraform-observability-team` repository serves **two main purposes**:

### Purpose 1: Team Permissions Management

This repository manages who has access to what across multiple platforms using **Infrastructure-as-Code (Terraform)**:

- **GitHub** (public `linode-obs` organization)
- **Bits** (internal Akamai/Linode GitHub - `sre-o11y` teams across ops, Linode, devcloud orgs)
- **PagerDuty** (team memberships, on-call schedules, escalation policies)
- **Vault** (secret storage for team applications)

**Why Terraform?** Because managing permissions manually across 4+ platforms for 10+ people is error-prone and time-consuming. Terraform allows us to:
- Define team membership once
- Apply changes consistently everywhere
- Track permission changes in git history
- Easily onboard/offboard team members

### Purpose 2: Team Documentation Hub

This repository contains all team documentation using **Hugo** (a static site generator):

- **Handbooks**: How we work (on-call, tools, git conventions)
- **Service Documentation**: VictoriaMetrics, Prometheus, Grafana, ArgoCD, etc.
- **MOPs** (Manual Operations Procedures): Step-by-step guides for complex tasks
- **Proposals**: Design documents for major changes (OP-## format)
- **On-call Logs**: Weekly logs of incidents and work completed
- **Runbooks**: How to respond to specific alerts

The documentation is published to **Bits Pages** at:
`https://bits.linode.com/pages/ops/terraform-observability-team/`

---

## 2. What Does the Observability Team Do?

The **SRE Observability team** owns and operates the entire observability infrastructure for Linode/Akamai.

### Metrics Collection & Storage

**Services We Manage**:
- **100+ Prometheus instances** globally (one or more per datacenter)
- **VictoriaMetrics clusters** for long-term storage (LTS) of metrics
- **Thanos** for query federation across Prometheus instances

**Example Metrics**: CPU/memory/disk usage, network traffic, application response times, database performance, Kubernetes pod health

### Logging Infrastructure

**Services We Manage**:
- **Loki** clusters for centralized log storage
- **OpenTelemetry Collectors** (otelgw) for log aggregation
- Integration with Grafana for log querying

### Visualization

**Services We Manage**:
- **Grafana** instances
- **Dashboard management** (via Terraform)
- **User permissions** for Grafana

### Alerting

**Services We Manage**:
- **Alertmanager** (part of Prometheus)
- **PagerDuty** integrations
- **Alert rules** (prometheus_rules repository)
- **Alert routing** (who gets paged for what)

### Deployment & Orchestration

**Services We Manage**:
- **ArgoCD** for GitOps deployment to Kubernetes
- **Kubernetes clusters** running observability workloads
- **Helm charts** for application configuration

### Network Observability

**Services We Manage**:
- **Linmon** - Network monitoring platform
- Specialized VictoriaMetrics clusters for network metrics

---

## 3. The Big Picture: Observability Architecture

### How It All Fits Together

```
1. DATA COLLECTION
   Servers & Apps → node_exporter (metrics) → Prometheus (per DC)

2. LONG-TERM STORAGE
   Prometheus → remote_write → VictoriaMetrics LTS (regional)

3. QUERYING & FEDERATION
   VictoriaMetrics LTS → VictoriaMetrics Global Select → Worldwide queries

4. VISUALIZATION
   Grafana → queries → Prometheus/VictoriaMetrics/Loki → Dashboards

5. ALERTING
   Prometheus Alert Rules → Alertmanager → PagerDuty → On-call Engineer
```

### Regional Architecture

We split the world into **three regions**:

**North America**:
- VictoriaMetrics LTS: ord2 (primary), lax3 (backup)

**Europe**:
- VictoriaMetrics LTS: mad2 (primary), sto2 (backup)

**Asia Pacific**:
- VictoriaMetrics LTS: osa1 (primary), cgk1, sea1 (backup)

**Global**:
- VictoriaMetrics Global Select: Federates queries across all regions

**Why Regional?**
- Reduced latency
- Compliance (data locality)
- Scalability
- Resilience

---

## 4. Repository Structure Explained

```
terraform-observability-team/
├── .github/workflows/     # GitHub Actions CI/CD
├── docs/                  # Hugo documentation site
├── modules/              # Terraform modules
│   ├── pagerduty/       # PagerDuty config
│   ├── github/          # GitHub org management
│   └── bits/            # Bits team management
├── main.tf               # Main Terraform config
├── provider.tf           # Provider setup
├── vars.tf               # Variable definitions
├── team.auto.tfvars      # IMPORTANT: Team members
├── pagerduty_suppression.auto.tfvars  # Alert suppression
├── atlantis.yaml         # Atlantis config
├── .envrc                # Vault authentication
└── .pre-commit-config.yaml  # Code quality hooks
```

### The `/modules/` Directory

**`/modules/pagerduty/`**: Manages all PagerDuty configuration
- Team membership
- On-call schedules (primary, secondary)
- Escalation policies
- Alert suppression rules

**`/modules/github/`**: Manages public `linode-obs` GitHub organization
- Repository creation
- Team memberships
- Repository permissions

**`/modules/bits/`**: Manages internal Bits teams across orgs (ops, Linode, devcloud)
- Team memberships
- Repository permissions
- Webhooks (Atlantis, Slack)
- Branch protection

**Important Repos Managed**:
- `ops/prometheus_rules` - Alert and recording rules
- `ops/prometheus-formula` - Prometheus Salt configuration
- `ops/o11y-helm-charts` - Helm charts for Kubernetes apps
- `ops/palantir` - Kubernetes manifests
- `ops/loki_rules` - Loki alert rules
- `ops/terraform-grafana-config` - Grafana configuration

### The `/docs/` Directory

Hugo-based documentation site:

```
docs/content/
├── Handbooks/           # Team operations
├── Services/            # Service documentation
│   ├── ArgoCD/
│   ├── VictoriaMetrics/
│   ├── Prometheus/
│   ├── Grafana/
│   └── Kubernetes Clusters/
├── mops/                # Manual Operations Procedures
├── on-call/             # On-call logs by year
├── proposals/           # Design proposals (OP-##)
└── Runbooks/            # Alert response guides
```

---

## 5. Key Services Deep Dive

### VictoriaMetrics (Long-Term Metrics Storage)

**What**: Time-series database optimized for storing Prometheus metrics long-term

**Why**:
- **Compression**: 10x more efficient than Prometheus
- **Fast queries**: Much faster for historical data
- **Long retention**: 13 months vs Prometheus's 15 days
- **Compatible**: Works with Prometheus query language (PromQL)

**Architecture** - Three components:

1. **vminsert** (3 replicas): Receives metrics from Prometheus, distributes across storage
2. **vmstorage** (3-6 replicas): Stores compressed metrics on disk
3. **vmselect** (2 replicas): Handles queries from Grafana

**Cluster Types**:

1. **LTS (Long-Term Storage)** - Regional
   - One per continent
   - 13 months retention
   - Examples: victoriametrics-ord2-us-prod, victoriametrics-mad2-es-prod

2. **Global Select** - Worldwide
   - Federates queries across all LTS clusters
   - Does NOT store data, just proxies queries
   - Example: victoriametrics-global-select-iad3-us-prod

**Deployment**: Runs in Kubernetes, managed by ArgoCD

### Prometheus (Real-Time Metrics Collection)

**What**: Time-series database that scrapes (pulls) metrics from servers and applications

**Why**:
- Industry standard
- Pull-based model
- Service discovery (automatically finds what to monitor)
- Alert evaluation in real-time

**Deployment Model - Sharding**:

Multiple Prometheus instances per datacenter to handle scale:

```
Datacenter: Newark (ewr1)
   ├─ prometheus-1a (shard 0) - Monitors 1/3 of targets
   ├─ prometheus-1b (shard 1) - Monitors 1/3 of targets
   └─ prometheus-1c (shard 2) - Monitors 1/3 of targets
```

**Shard Naming**: `a=0, b=1, c=2, d=3, ...`

**High Availability**: Each shard runs 2 replicas

**Configuration**: Managed by Salt formula: `ops/prometheus-formula`

**Data Flow**:
- Local storage: 15 days
- Remote write: Sent to VictoriaMetrics for long-term storage

### Grafana (Visualization)

**What**: Web application for creating dashboards from metrics and logs

**We Manage**:
- Grafana instances (production, staging, dev)
- Datasources (Prometheus, VictoriaMetrics, Loki)
- Dashboards (via Terraform)
- User permissions

**Access**:
- Production: https://grafana.linode.com
- Staging: https://grafana-staging.linode.com

**Datasource Types**:
- **Prometheus**: Real-time data (15 days)
- **VictoriaMetrics**: Historical data (13 months)
- **Loki**: Logs
- **Thanos Query**: Federated Prometheus queries

### ArgoCD (GitOps Deployment)

**What**: GitOps tool that deploys applications to Kubernetes by syncing with Git

**How It Works**:
1. Engineer makes change in Git
2. ArgoCD detects change
3. ArgoCD compares Git vs Kubernetes
4. ArgoCD applies changes
5. Application updated

**Instances**:
- **Production**: https://argocd.infra-o11y-apps.iad3.us.prod.linode.com (uses release tags)
- **Staging**: https://argocd.infra-o11y-apps.rin1.us.staging.linode.com (uses main branch)
- **Dev**: https://argocd.infra-o11y-apps.rin1.us.dev.linode.com (uses main branch)

**Key Repositories**:
- `ops/palantir` - Kubernetes manifests
- `ops/o11y-helm-charts` - Generates Application manifests

### Kubernetes Clusters

**Cluster Types**:

1. **infra-o11y-apps**: Runs ArgoCD, Grafana, centralized services
2. **victoriametrics**: Dedicated VictoriaMetrics clusters
3. **infra-logging**: Loki and centralized logging

**Total**: 30+ clusters across staging/production

### Centralized Logging

**Architecture**:

```
Servers → Fluent Bit → otelgw (per DC) → Loki Distributor
→ Loki Ingester → Object Storage (S3) → Loki Querier → Grafana
```

**Components**:
- **otelgw**: OpenTelemetry Gateway, one per datacenter
- **Loki**: Log aggregation (like Prometheus but for logs)
  - Distributor: Receives logs
  - Ingester: Buffers and stores
  - Querier: Queries logs

---

## 6. How Deployments Work

### This Repository Workflow

```
1. Developer creates PR
2. Atlantis automatically runs terraform plan
3. Team reviews PR
4. PR approved and merged
5. Comment "atlantis apply" on PR
6. Atlantis applies Terraform changes
7. Changes live in PagerDuty/GitHub/Bits
```

**Key**: Plan is automatic, apply is manual via comment

### Application Changes (Kubernetes)

```
1. Change in ops/o11y-helm-charts
   - Example: Upgrade VictoriaMetrics version
   - Edit values file, create PR, merge

2. GitHub Action auto-creates Palantir PR
   - Renders Helm chart
   - Generates updated manifests
   - Creates PR in ops/palantir

3. Review and merge Palantir PR

4. ArgoCD detects change
   - Polls ops/palantir every 3 minutes
   - Status: "OutOfSync"

5. Manual sync in ArgoCD
   - Click "Sync" button
   - Rolls out new pods

6. Verify deployment
   - Check pods running
   - Check Grafana dashboards
```

**Key Points**:
- Two PRs required
- Staging uses main branch, production uses release tags
- Manual sync in ArgoCD for production

---

## 7. Important Files You Need to Know

### `/team.auto.tfvars` - CRITICAL FILE

**Single source of truth for team membership**

**Structure**:

```hcl
# Team Members
observability_members = {
  jdoe = {
    name                 = "Jane Doe"
    email                = "jane.doe@akamai.com"
    job_title            = "Senior SRE"
    github_username      = "jdoe"
    github_admin         = false
    bits_team_maintainer = true
    bits_orgs            = ["ops", "Linode"]
    pd_enabled           = true
  }
}

# On-call Rotation
observability_oncall_primary = [
  "current_oncall",   # MUST be first
  "person2",
  "person3",
  "new_person"        # Add at END
]

# On-call Start Time - Monday 11am ET of current shift
pagerduty_schedule_starttime = "2025-11-18T11:00:00-05:00"
```

**When to Edit**:
- Adding/removing team members
- Changing on-call rotation
- Modifying repository permissions

### `/pagerduty_suppression.auto.tfvars`

**Alert suppression during maintenance**

```hcl
# DCs with alerts suppressed
observability_suppressed_dcs = [
  "iad5",
  "iad05",   # Include both formats
]
```

**Important**: Remove suppression after maintenance!

### `/.envrc`

**Environment variables for Vault authentication**

```bash
export VAULT_ADDR="https://vault.infra.linode.com"
vault login -method=oidc username=$USER
```

**Setup**: Install direnv, then `direnv allow`

### `/.pre-commit-config.yaml`

**Automated code quality checks**

**Hooks**: terraform fmt, validate, markdownlint, vale, prettier

**Setup**: `pre-commit install --install-hooks`

### `/.mise.toml`

**Task runner**

**Common tasks**:
- `mise run oncall` - Create on-call log
- `mise run proposal` - Create new proposal
- `mise run docs-serve` - Serve docs locally

---

## 8. Documentation Structure

### Content Types

**Handbooks** (`docs/content/Handbooks/`):
- `on-call.md` - On-call responsibilities and schedule
- `tools.md` - Standardized tooling
- `git-conventions.md` - Commit messages, branching

**Services** (`docs/content/Services/`):
- Service-specific documentation
- Architecture, deployment, troubleshooting

**MOPs** (`docs/content/mops/`):
- Manual Operations Procedures
- Step-by-step guides for complex tasks
- Includes: Overview, Prerequisites, Procedure, Verification, Rollback

**Proposals** (`docs/content/proposals/`):
- Design documents (OP-## format)
- Statuses: draft, accepted, rejected, done

**On-call Logs** (`docs/content/on-call/YYYY/`):
- Weekly logs of incidents and work
- Created via `mise run oncall`

**Runbooks** (`docs/content/Runbooks/`):
- How to respond to specific alerts

### Creating Content

```bash
cd docs/

# New service docs
hugo new Services/MyService/_index.md

# New MOP
hugo new mops/my-procedure.md

# New proposal
mise run proposal

# New on-call log
mise run oncall
```

---

## 9. Day-to-Day Operations

### On-call Responsibilities

**Shift**: Monday 11am ET → Next Monday 11am ET

**Duties**:

1. **Respond to Pages** (Critical): ≤ 5 min acknowledgment
2. **Monitor Warnings**: ≤ 12 hours acknowledgment
3. **Handle Intake**: Grafana permissions, quick questions
4. **PR Reviews**: Review ops/sre-o11y repos
5. **Reliability Improvement**: One task per week
6. **Attend Post-Mortems**: For incidents you handled

**Handoff**:
- Fill out on-call log
- Post in #o11y-core Slack
- Highlight ongoing issues

### Common Daily Tasks

**Reviewing PRs**:
- Repositories: prometheus_rules, prometheus-formula, o11y-helm-charts, palantir
- Check CI, review changes, verify Terraform plan
- Notifications: #notification-o11y-prs

**Granting Grafana Access**:
1. Determine access level (Viewer/Editor/Admin)
2. Add user in terraform-grafana-config
3. Create PR, merge, apply
4. User receives invite

**Monitoring Alerts**:
- Channels: #notification-o11y, PagerDuty
- Check runbook, follow steps
- Create incident ticket if needed
- Silence if necessary using amtool

---

## 10. Common Tasks with Examples

### Task 1: Add New Team Member

```bash
# 1. Create branch
git checkout -b add-jane-doe

# 2. Edit team.auto.tfvars
vim team.auto.tfvars
# Add to observability_members

# 3. Commit and push
git commit -m "Add Jane Doe to team"
git push origin add-jane-doe

# 4. Create PR, get approval, merge

# 5. Apply
# Comment: "atlantis apply"

# 6. Verify in PagerDuty, GitHub, Bits
```

### Task 2: Add to On-call Rotation

```bash
# Edit team.auto.tfvars
observability_oncall_primary = [
  "current_oncall",   # MUST be first
  "person2",
  "person3",
  "jdoe"              # Add new person LAST
]

# Update start time to current Monday
pagerduty_schedule_starttime = "2025-11-18T11:00:00-05:00"

# PR, merge, apply
```

### Task 3: Suppress Alerts for DC Maintenance

```bash
# Edit pagerduty_suppression.auto.tfvars
observability_suppressed_dcs = [
  "iad5",
  "iad05",
]

# PR, merge, apply immediately

# IMPORTANT: Remove after maintenance!
```

### Task 4: Upgrade VictoriaMetrics

```bash
# 1. Clone o11y-helm-charts
cd ~/repos/o11y-helm-charts

# 2. Update staging values
vim values-files/victoriametrics-ord2-us-staging.yaml
# Change version: v1.99.0 → v1.100.0

# 3. PR, merge

# 4. Wait for Palantir PR auto-creation

# 5. Review and merge Palantir PR

# 6. Sync in ArgoCD staging

# 7. Verify deployment

# 8. Repeat for production
```

### Task 5: Create On-call Log

```bash
cd ~/repos/terraform-observability-team/docs

# Create log for current week
mise run oncall

# Edit throughout week
vim content/on-call/2025/2025-11-18.md

# At end of week, commit
git commit -m "On-call log: 2025-11-18"
git push

# Post link in #o11y-core
```

---

## 11. Things to Be Careful About

### Critical Mistakes to Avoid

**1. DON'T: Edit On-call Without Updating Start Time**

Wrong:
```hcl
observability_oncall_primary = ["person2", "person1"]
pagerduty_schedule_starttime = "2025-11-04T11:00:00-05:00"  # OLD
```

Right:
```hcl
observability_oncall_primary = ["person1", "person2"]  # Current first
pagerduty_schedule_starttime = "2025-11-18T11:00:00-05:00"  # Current Monday
```

**2. DON'T: Commit Secrets to Git**

Always use Vault, never hardcode secrets

**3. DON'T: Force Push to Main/Master**

Never use `git push --force origin main`

**4. DON'T: Skip Atlantis Plan Review**

Always read the entire Terraform plan before applying

**5. DON'T: Amend Other People's Commits**

Create new commits instead

**6. DON'T: Forget to Remove Alert Suppressions**

Set reminders, create follow-up tickets

### Important Gotchas

**Gotcha 1**: Staging uses main branch, production uses release tags

**Gotcha 2**: Upgrade VictoriaMetrics LTS clusters BEFORE Global Select

**Gotcha 3**: Prometheus sharding takes 2-8 hours

**Gotcha 4**: Cilium network policies require pod restarts

**Gotcha 5**: Management IPs can fail silently - always verify

### When to Ask for Help

**Ask immediately if**:
- Critical alert and don't know how to fix
- About to run destructive command
- Unsure about Terraform plan

**Where to Ask**:
- #o11y-core (team private)
- #sre-observability (team public)
- PagerDuty (escalate alert)

**It's better to ask than guess!**

---

## 12. Getting Started Checklist

### Week 1: Setup & Access

- [ ] Get added to team.auto.tfvars
- [ ] Verify PagerDuty, GitHub, Bits, Grafana, ArgoCD access
- [ ] Install tools: asdf/mise, direnv, kubectl, terraform, pre-commit, Hugo
- [ ] Clone repos: terraform-observability-team, o11y-helm-charts, palantir, prometheus_rules
- [ ] Setup terraform repo: `direnv allow`, `pre-commit install`, test `terraform plan`
- [ ] Join Slack channels: #sre-observability, #o11y-core, #notification-o11y, #notification-o11y-prs

### Week 2: Learning

- [ ] Read documentation: README, Handbooks, recent proposals
- [ ] Explore repositories
- [ ] Shadow team members
- [ ] Attend team meetings

### Week 3: First Tasks

- [ ] Make first PR (fix typo in docs)
- [ ] Access Grafana, ArgoCD
- [ ] Query Prometheus/VictoriaMetrics
- [ ] Attend post-mortem (if one occurs)

### Week 4: Increasing Responsibility

- [ ] Handle first alert
- [ ] Review first PR
- [ ] Complete first intake request

### Month 2: On-call Training

- [ ] Read all runbooks
- [ ] Shadow on-call for 2 weeks
- [ ] Get added to rotation
- [ ] First on-call shift (with backup)

### Month 3: Full Team Member

- [ ] Lead first project
- [ ] Contribute to documentation
- [ ] Mentor next new hire

---

## 13. Glossary

**ArgoCD**: GitOps deployment tool for Kubernetes

**Atlantis**: Terraform automation on PRs

**Bits**: Akamai's internal GitHub (bits.linode.com)

**DC**: Datacenter (ewr1, iad3, etc.)

**Grafana**: Visualization platform

**Hugo**: Static site generator for docs

**Loki**: Log aggregation system

**LTS**: Long-Term Storage (VictoriaMetrics)

**Mise**: Task runner and tool manager

**MOP**: Manual Operations Procedure

**mTLS**: Mutual TLS authentication

**NIL**: Network Internet Listener (LoadBalancer)

**otelgw**: OpenTelemetry Gateway

**Prometheus**: Time-series database for metrics

**PromQL**: Prometheus Query Language

**Runbook**: Alert response guide

**Salt**: Configuration management tool

**Shard**: Divided workload instance

**Terraform**: Infrastructure-as-Code

**Vault**: Secret management

**VictoriaMetrics**: Optimized time-series database

---

## Additional Resources

### Internal Systems

- Grafana Production: https://grafana.linode.com
- ArgoCD Production: https://argocd.infra-o11y-apps.iad3.us.prod.linode.com
- PagerDuty: https://akamai.pagerduty.com/schedules/PSFD91L
- Team Docs: https://bits.linode.com/pages/ops/terraform-observability-team/

### Slack Channels

- #sre-observability (public)
- #o11y-core (private)
- #notification-o11y
- #notification-o11y-prs

---

## Final Thoughts

Welcome to the SRE Observability team! This guide is a reference - you're not expected to memorize everything immediately.

**Key Takeaways**:
1. terraform-observability-team manages permissions and docs
2. VictoriaMetrics, Prometheus, Grafana, ArgoCD are core services
3. GitOps workflow: Code → PR → Review → Merge → Deploy
4. On-call is a rotation with training
5. Ask questions - the team is here to help!

**Learning Path**: Week 1 (setup) → Week 2 (learn) → Week 3 (first tasks) → Week 4 (alerts) → Month 2 (on-call) → Month 3 (full contributor)

**Questions?** Ask in #o11y-core!

**Good luck, and welcome to the team!**
