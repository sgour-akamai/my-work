          1 +  # Complete Onboarding Guide: terraform-observability-team Repository
          2 +  
          3 +  ## Welcome to the SRE Observability Team!
          4 +  
          5 +  This guide will teach you everything you need to know about the `terraform-observability-team` 
            + repository as a new team member. No prior knowledge assumed - we'll start from the basics and go 
            + deep.
          6 +  
          7 +  **Team Motto**: *Mors melius quam Nagios* ("Death is better than Nagios")
          8 +  
          9 +  ---
         10 +  
         11 +  ## Table of Contents
         12 +  
         13 +  1. [What is This Repository?](#1-what-is-this-repository)
         14 +  2. [What Does the Observability Team Do?](#2-what-does-the-observability-team-do)
         15 +  3. [The Big Picture: Observability Architecture](#3-the-big-picture-observability-architecture)
         16 +  4. [Repository Structure Explained](#4-repository-structure-explained)
         17 +  5. [Key Services Deep Dive](#5-key-services-deep-dive)
         18 +  6. [How Deployments Work](#6-how-deployments-work)
         19 +  7. [Important Files You Need to Know](#7-important-files-you-need-to-know)
         20 +  8. [Documentation Structure](#8-documentation-structure)
         21 +  9. [Day-to-Day Operations](#9-day-to-day-operations)
         22 +  10. [Common Tasks with Examples](#10-common-tasks-with-examples)
         23 +  11. [Things to Be Careful About](#11-things-to-be-careful-about)
         24 +  12. [Getting Started Checklist](#12-getting-started-checklist)
         25 +  13. [Glossary](#13-glossary)
         26 +  
         27 +  ---
         28 +  
         29 +  ## 1. What is This Repository?
         30 +  
         31 +  The `terraform-observability-team` repository serves **two main purposes**:
         32 +  
         33 +  ### Purpose 1: Team Permissions Management
         34 +  
         35 +  This repository manages who has access to what across multiple platforms using 
            + **Infrastructure-as-Code (Terraform)**:
         36 +  
         37 +  - **GitHub** (public `linode-obs` organization)
         38 +  - **Bits** (internal Akamai/Linode GitHub - `sre-o11y` teams across ops, Linode, devcloud orgs)
         39 +  - **PagerDuty** (team memberships, on-call schedules, escalation policies)
         40 +  - **Vault** (secret storage for team applications)
         41 +  
         42 +  **Why Terraform?** Because managing permissions manually across 4+ platforms for 10+ people is 
            + error-prone and time-consuming. Terraform allows us to:
         43 +  - Define team membership once
         44 +  - Apply changes consistently everywhere
         45 +  - Track permission changes in git history
         46 +  - Easily onboard/offboard team members
         47 +  
         48 +  ### Purpose 2: Team Documentation Hub
         49 +  
         50 +  This repository contains all team documentation using **Hugo** (a static site generator):
         51 +  
         52 +  - **Handbooks**: How we work (on-call, tools, git conventions)
         53 +  - **Service Documentation**: VictoriaMetrics, Prometheus, Grafana, ArgoCD, etc.
         54 +  - **MOPs** (Manual Operations Procedures): Step-by-step guides for complex tasks
         55 +  - **Proposals**: Design documents for major changes (OP-## format)
         56 +  - **On-call Logs**: Weekly logs of incidents and work completed
         57 +  - **Runbooks**: How to respond to specific alerts
         58 +  
         59 +  The documentation is published to **Bits Pages** (internal documentation hosting) at:
         60 +  `https://bits.linode.com/pages/ops/terraform-observability-team/`
         61 +  
         62 +  ---
         63 +  
         64 +  ## 2. What Does the Observability Team Do?
         65 +  
         66 +  The **SRE Observability team** owns and operates the entire observability infrastructure for 
            + Linode/Akamai. Here's what that means:
         67 +  
         68 +  ### Metrics Collection & Storage
         69 +  
         70 +  **What**: Collecting and storing performance metrics from all infrastructure.
         71 +  
         72 +  **Services We Manage**:
         73 +  - **100+ Prometheus instances** globally (one or more per datacenter)
         74 +  - **VictoriaMetrics clusters** for long-term storage (LTS) of metrics
         75 +  - **Thanos** for query federation across Prometheus instances
         76 +  
         77 +  **Example Metrics**:
         78 +  - CPU/memory/disk usage on servers
         79 +  - Network traffic and errors
         80 +  - Application response times and error rates
         81 +  - Database query performance
         82 +  - Kubernetes pod health
         83 +  
         84 +  ### Logging Infrastructure
         85 +  
         86 +  **What**: Collecting, storing, and querying logs from all systems.
         87 +  
         88 +  **Services We Manage**:
         89 +  - **Loki** clusters for centralized log storage
         90 +  - **OpenTelemetry Collectors** (otelgw) for log aggregation
         91 +  - Integration with Grafana for log querying
         92 +  
         93 +  ### Visualization
         94 +  
         95 +  **What**: Providing dashboards and graphs for engineers to understand system behavior.
         96 +  
         97 +  **Services We Manage**:
         98 +  - **Grafana** instances
         99 +  - **Dashboard management** (via Terraform)
        100 +  - **User permissions** for Grafana
        101 +  
        102 +  ### Alerting
        103 +  
        104 +  **What**: Notifying engineers when things go wrong.
        105 +  
        106 +  **Services We Manage**:
        107 +  - **Alertmanager** (part of Prometheus)
        108 +  - **PagerDuty** integrations
        109 +  - **Alert rules** (prometheus_rules repository)
        110 +  - **Alert routing** (who gets paged for what)
        111 +  
        112 +  ### Deployment & Orchestration
        113 +  
        114 +  **What**: Managing how observability services are deployed and updated.
        115 +  
        116 +  **Services We Manage**:
        117 +  - **ArgoCD** for GitOps deployment to Kubernetes
        118 +  - **Kubernetes clusters** running observability workloads
        119 +  - **Helm charts** for application configuration
        120 +  
        121 +  ### Legacy Systems (Being Phased Out)
        122 +  
        123 +  - **Nagios** - Old monitoring system (being replaced by Prometheus)
        124 +  
        125 +  ### Network Observability
        126 +  
        127 +  **What**: Special monitoring for network devices and traffic.
        128 +  
        129 +  **Services We Manage**:
        130 +  - **Linmon** - Network monitoring platform
        131 +  - Specialized VictoriaMetrics clusters for network metrics
        132 +  
        133 +  ---
        134 +  
        135 +  ## 3. The Big Picture: Observability Architecture
        136 +  
        137 +  Let me explain how all these pieces fit together with a real-world example:
        138 +  
        139 +  ### Example: Monitoring a Linode Customer's VM
        140 +  
        141 +  ```
        142 +  1. DATA COLLECTION
        143 +     ┌────────────────────────────────────────────────────────────┐
        144 +     │  Customer's Linode VM (in Newark datacenter - ewr1)        │
        145 +     │  ├─ node_exporter (exposes system metrics)                 │
        146 +     │  └─ Metrics: CPU, memory, disk, network                    │
        147 +     └────────────────────────┬───────────────────────────────────┘
        148 +                              │ (HTTP scrape every 30s)
        149 +                              ▼
        150 +     ┌────────────────────────────────────────────────────────────┐
        151 +     │  Prometheus Shard (prometheus-1a in ewr1)                  │
        152 +     │  ├─ Scrapes 1000s of targets in this datacenter           │
        153 +     │  ├─ Stores metrics locally (15 days retention)            │
        154 +     │  └─ Evaluates alert rules                                 │
        155 +     └────────────────────────┬───────────────────────────────────┘
        156 +                              │ (remote_write)
        157 +                              ▼
        158 +  2. LONG-TERM STORAGE
        159 +     ┌────────────────────────────────────────────────────────────┐
        160 +     │  VictoriaMetrics LTS (North America - ord2/lax3)           │
        161 +     │  ├─ Receives metrics from all NA datacenters              │
        162 +     │  ├─ Compresses and stores metrics (13 months retention)   │
        163 +     │  └─ Fast queries for historical data                      │
        164 +     └────────────────────────┬───────────────────────────────────┘
        165 +                              │
        166 +                              ▼
        167 +  3. QUERYING & FEDERATION
        168 +     ┌────────────────────────────────────────────────────────────┐
        169 +     │  VictoriaMetrics Global Select                             │
        170 +     │  ├─ Federates queries across all regions (NA, EU, AP)     │
        171 +     │  └─ Single query point for worldwide data                 │
        172 +     └────────────────────────┬───────────────────────────────────┘
        173 +                              │ (queries)
        174 +                              ▼
        175 +  4. VISUALIZATION
        176 +     ┌────────────────────────────────────────────────────────────┐
        177 +     │  Grafana                                                    │
        178 +     │  ├─ Dashboards showing VM performance                      │
        179 +     │  └─ Engineers use this to troubleshoot issues             │
        180 +     └────────────────────────────────────────────────────────────┘
        181 +  
        182 +  5. ALERTING (if something goes wrong)
        183 +     ┌────────────────────────────────────────────────────────────┐
        184 +     │  Prometheus Alert Rules                                    │
        185 +     │  └─ "High CPU usage on VM for 5 minutes"                  │
        186 +     └────────────────────────┬───────────────────────────────────┘
        187 +                              │ (fires alert)
        188 +                              ▼
        189 +     ┌────────────────────────────────────────────────────────────┐
        190 +     │  Alertmanager                                               │
        191 +     │  └─ Routes alert based on severity and team               │
        192 +     └────────────────────────┬───────────────────────────────────┘
        193 +                              │
        194 +                              ▼
        195 +     ┌────────────────────────────────────────────────────────────┐
        196 +     │  PagerDuty                                                  │
        197 +     │  └─ Pages on-call engineer                                │
        198 +     └────────────────────────────────────────────────────────────┘
        199 +  ```
        200 +  
        201 +  ### Regional Architecture
        202 +  
        203 +  We split the world into **three regions** for scalability:
        204 +  
        205 +  ```
        206 +  ┌─────────────────────────────────────────────────────────────────┐
        207 +  │                   GLOBAL ARCHITECTURE                            │
        208 +  │                                                                  │
        209 +  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
        210 +  │  │ North America│  │    Europe    │  │ Asia Pacific │          │
        211 +  │  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
        212 +  │  │ VictoriaMetrics  │ VictoriaMetrics  │ VictoriaMetrics          │
        213 +  │  │ LTS Clusters │  │ LTS Clusters │  │ LTS Clusters │          │
        214 +  │  │              │  │              │  │              │          │
        215 +  │  │ ord2 (primary)│  │ mad2 (primary)│  │ osa1 (primary)│          │
        216 +  │  │ lax3 (backup)│  │ sto2 (backup)│  │ cgk1 (backup)│          │
        217 +  │  │              │  │              │  │ sea1 (backup)│          │
        218 +  │  └──────────────┘  └──────────────┘  └──────────────┘          │
        219 +  │         │                 │                  │                  │
        220 +  │         └─────────────────┴──────────────────┘                  │
        221 +  │                           │                                     │
        222 +  │                           ▼                                     │
        223 +  │              ┌────────────────────────┐                         │
        224 +  │              │ VictoriaMetrics        │                         │
        225 +  │              │ Global Select          │                         │
        226 +  │              │ (Query Federation)     │                         │
        227 +  │              └────────────────────────┘                         │
        228 +  └─────────────────────────────────────────────────────────────────┘
        229 +  ```
        230 +  
        231 +  **Why Regional?**
        232 +  - **Reduced latency**: Data stays close to where it's generated
        233 +  - **Compliance**: Some regions require data to stay in-country
        234 +  - **Scalability**: Spreading load across multiple clusters
        235 +  - **Resilience**: Regional failure doesn't impact other regions
        236 +  
        237 +  ---
        238 +  
        239 +  ## 4. Repository Structure Explained
        240 +  
        241 +  Let's walk through the repository structure directory by directory:
        242 +  
        243 +  ```
        244 +  /Users/sgour/terraform-observability-team/
        245 +  ├── .github/                    # GitHub-specific files
        246 +  │   ├── workflows/             # GitHub Actions CI/CD pipelines
        247 +  │   └── CODEOWNERS             # Who reviews PRs for which files
        248 +  ├── .vale/                      # Vale (prose linting) configuration
        249 +  │   └── styles/                # Custom style rules for docs
        250 +  ├── docs/                       # Hugo documentation site (explained later)
        251 +  ├── hack/                       # Utility scripts
        252 +  ├── img/                        # Images for README
        253 +  ├── modules/                    # Terraform modules (explained below)
        254 +  │   ├── pagerduty/             # PagerDuty configuration
        255 +  │   ├── github/                # GitHub (public) configuration
        256 +  │   └── bits/                  # Bits (internal GitHub) configuration
        257 +  ├── scripts/                    # Helper scripts
        258 +  ├── main.tf                     # Main Terraform configuration
        259 +  ├── provider.tf                 # Terraform provider setup
        260 +  ├── vars.tf                     # Variable definitions
        261 +  ├── team.auto.tfvars           # IMPORTANT: Team member definitions
        262 +  ├── pagerduty_suppression.auto.tfvars  # Alert suppression config
        263 +  ├── atlantis.yaml              # Atlantis (Terraform automation) config
        264 +  ├── .envrc                     # Environment variables (Vault auth)
        265 +  ├── .pre-commit-config.yaml    # Pre-commit hooks for code quality
        266 +  ├── .mise.toml                 # Mise task runner configuration
        267 +  └── README.md                  # Repository overview
        268 +  ```
        269 +  
        270 +  ### The `/modules/` Directory (Terraform Modules)
        271 +  
        272 +  Think of Terraform modules as reusable "functions" for infrastructure. Each module manages a 
            + specific platform:
        273 +  
        274 +  #### `/modules/pagerduty/`
        275 +  
        276 +  Manages all PagerDuty configuration:
        277 +  
        278 +  **Files**:
        279 +  - `team.tf` - Team membership (who's on the team)
        280 +  - `escalation_policy.tf` - Who gets alerted when (primary → secondary → round-robin)
        281 +  - `services.tf` - PagerDuty services (e.g., "Prometheus Alerts")
        282 +  - `schedules.tf` - On-call schedules (primary, secondary)
        283 +  - `orchestration.tf` - Event routing and processing
        284 +  - `slack_connections.tf` - Slack channel integrations
        285 +  - `rules.tf` - Alert suppression rules (for maintenance)
        286 +  
        287 +  **What It Does**:
        288 +  - Creates PagerDuty team
        289 +  - Sets up on-call rotation schedule
        290 +  - Creates escalation policies
        291 +  - Configures alert routing
        292 +  - Manages alert suppression during maintenance
        293 +  
        294 +  **Example**: When you're added to the team, Terraform creates your PagerDuty user and adds you to 
            + schedules.
        295 +  
        296 +  #### `/modules/github/`
        297 +  
        298 +  Manages the **public** `linode-obs` GitHub organization:
        299 +  
        300 +  **What It Does**:
        301 +  - Creates repositories
        302 +  - Manages team memberships
        303 +  - Sets repository permissions
        304 +  - Configures branch protection
        305 +  
        306 +  **Example Repos Managed**:
        307 +  - `linode-obs/victoriametrics-operator`
        308 +  - `linode-obs/prometheus-salt-formula`
        309 +  
        310 +  #### `/modules/bits/`
        311 +  
        312 +  Manages the **internal** Bits (Akamai GitHub) teams across multiple organizations:
        313 +  
        314 +  **Organizations Managed**:
        315 +  - `ops` - Main SRE organization
        316 +  - `Linode` - Linode-specific repos
        317 +  - `devcloud` - Devcloud infrastructure
        318 +  - `ansible-collections` - Ansible collections
        319 +  
        320 +  **What It Does**:
        321 +  - Creates `sre-o11y` team in each org
        322 +  - Manages team memberships (member vs maintainer roles)
        323 +  - Sets repository permissions
        324 +  - Configures webhooks (Atlantis, Slack notifications)
        325 +  - Sets up branch protection
        326 +  
        327 +  **Important Repos Managed**:
        328 +  - `ops/prometheus_rules` - Alert and recording rules
        329 +  - `ops/prometheus-formula` - Prometheus Salt configuration
        330 +  - `ops/o11y-helm-charts` - Helm charts for Kubernetes apps
        331 +  - `ops/palantir` - Kubernetes manifests
        332 +  - `ops/loki_rules` - Loki alert rules
        333 +  - `ops/terraform-grafana-config` - Grafana configuration
        334 +  
        335 +  ### The `/docs/` Directory (Documentation Site)
        336 +  
        337 +  This is a Hugo-based documentation site. Let's break it down:
        338 +  
        339 +  ```
        340 +  docs/
        341 +  ├── archetypes/                # Templates for new content
        342 +  │   ├── default.md
        343 +  │   ├── on-call.md            # Template for on-call logs
        344 +  │   └── proposals.md          # Template for proposals
        345 +  ├── content/                   # ACTUAL DOCUMENTATION (main content)
        346 +  │   ├── _index.md             # Homepage
        347 +  │   ├── Handbooks/            # How the team operates
        348 +  │   │   ├── on-call.md        # On-call guide
        349 +  │   │   ├── tools.md          # Tooling standards
        350 +  │   │   ├── git-conventions.md
        351 +  │   │   └── docs/             # How to write docs
        352 +  │   ├── Services/             # Service documentation
        353 +  │   │   ├── ArgoCD/
        354 +  │   │   ├── VictoriaMetrics/
        355 +  │   │   ├── Prometheus/
        356 +  │   │   ├── Grafana/
        357 +  │   │   ├── Kubernetes Clusters/
        358 +  │   │   ├── Centralized Logging/
        359 +  │   │   ├── Network Observability/
        360 +  │   │   ├── Nagios/
        361 +  │   │   └── ...
        362 +  │   ├── mops/                 # Manual Operations Procedures
        363 +  │   │   ├── prometheus-shard.md
        364 +  │   │   ├── victoriametrics-cluster-upgrade.md
        365 +  │   │   └── ...
        366 +  │   ├── on-call/              # On-call logs by year
        367 +  │   │   ├── 2024/
        368 +  │   │   └── 2025/
        369 +  │   ├── projects/             # Active projects
        370 +  │   ├── proposals/            # Design proposals (OP-##)
        371 +  │   │   ├── OP-01-prometheus-sharding.md
        372 +  │   │   ├── OP-03-o11y-helm-charts.md
        373 +  │   │   └── ...
        374 +  │   └── Runbooks/             # Alert response guides
        375 +  │       ├── HighMemoryUsage.md
        376 +  │       └── ...
        377 +  ├── layouts/                  # Custom Hugo templates
        378 +  ├── static/                   # Static files (images, CSS)
        379 +  ├── config.toml               # Hugo configuration
        380 +  └── go.mod                    # Hugo module dependencies
        381 +  ```
        382 +  
        383 +  **How It Works**:
        384 +  1. You write documentation in Markdown in `/docs/content/`
        385 +  2. Hugo builds it into HTML
        386 +  3. GitHub Actions deploys it to Bits Pages
        387 +  4. Team members access it at `https://bits.linode.com/pages/ops/terraform-observability-team/`
        388 +  
        389 +  ---
        390 +  
        391 +  ## 5. Key Services Deep Dive
        392 +  
        393 +  Now let's understand the major services the team manages:
        394 +  
        395 +  ### VictoriaMetrics (Long-Term Metrics Storage)
        396 +  
        397 +  **What is it?**
        398 +  VictoriaMetrics is a time-series database optimized for storing Prometheus metrics long-term. Think
            +  of it as "Prometheus but faster and with more storage capacity."
        399 +  
        400 +  **Why do we use it?**
        401 +  - **Compression**: Stores data 10x more efficiently than Prometheus
        402 +  - **Fast queries**: Queries historical data much faster
        403 +  - **Long retention**: We keep metrics for 13 months vs Prometheus's 15 days
        404 +  - **Compatible**: Works with Prometheus query language (PromQL)
        405 +  
        406 +  **Architecture**:
        407 +  
        408 +  VictoriaMetrics runs as a **cluster** with three components:
        409 +  
        410 +  ```
        411 +  ┌────────────────────────────────────────────────────────────────┐
        412 +  │              VictoriaMetrics Cluster Architecture               │
        413 +  │                                                                 │
        414 +  │  ┌─────────────┐                                               │
        415 +  │  │  vminsert   │ ← Receives metrics from Prometheus            │
        416 +  │  │  (3 replicas)│    (via remote_write)                        │
        417 +  │  └──────┬──────┘                                               │
        418 +  │         │ Distributes data across storage nodes                │
        419 +  │         ▼                                                       │
        420 +  │  ┌─────────────┐                                               │
        421 +  │  │  vmstorage  │ ← Stores compressed metrics on disk           │
        422 +  │  │  (3+ replicas)│   (each replica has full dataset)           │
        423 +  │  └──────┬──────┘                                               │
        424 +  │         │ Serves data to query nodes                           │
        425 +  │         ▼                                                       │
        426 +  │  ┌─────────────┐                                               │
        427 +  │  │  vmselect   │ ← Handles queries from Grafana                │
        428 +  │  │  (2 replicas)│   (PromQL compatible)                        │
        429 +  │  └─────────────┘                                               │
        430 +  └────────────────────────────────────────────────────────────────┘
        431 +  ```
        432 +  
        433 +  **Components Explained**:
        434 +  
        435 +  1. **vminsert** - Ingestion
        436 +     - Receives metrics from Prometheus instances
        437 +     - Validates and processes data
        438 +     - Distributes data across vmstorage nodes
        439 +     - **Replicas**: 3 (for redundancy)
        440 +  
        441 +  2. **vmstorage** - Storage
        442 +     - Stores compressed metrics on disk
        443 +     - Each replica has the full dataset
        444 +     - Handles compaction and retention
        445 +     - **Replicas**: 3-6 depending on cluster size
        446 +  
        447 +  3. **vmselect** - Queries
        448 +     - Receives queries from Grafana
        449 +     - Fetches data from vmstorage nodes
        450 +     - Aggregates results
        451 +     - **Replicas**: 2 (for load balancing)
        452 +  
        453 +  **Cluster Types**:
        454 +  
        455 +  We have two types of VictoriaMetrics clusters:
        456 +  
        457 +  1. **LTS (Long-Term Storage) Clusters** - Regional
        458 +     - One per continent (NA, EU, AP)
        459 +     - Stores metrics from Prometheus instances in that region
        460 +     - **Retention**: 13 months
        461 +     - **Examples**:
        462 +       - `victoriametrics-ord2-us-prod` (North America primary)
        463 +       - `victoriametrics-lax3-us-prod` (North America backup)
        464 +       - `victoriametrics-mad2-es-prod` (Europe primary)
        465 +  
        466 +  2. **Global Select Cluster** - Worldwide
        467 +     - Federates queries across all LTS clusters
        468 +     - Allows querying global data from one place
        469 +     - **Does NOT store data** - just proxies queries
        470 +     - **Example**: `victoriametrics-global-select-iad3-us-prod`
        471 +  
        472 +  **Data Flow Example**:
        473 +  
        474 +  ```
        475 +  Prometheus in Newark (ewr1)
        476 +      │ remote_write
        477 +      ▼
        478 +  VictoriaMetrics LTS (ord2) - North America
        479 +      │
        480 +      │ Query from Grafana
        481 +      ▼
        482 +  VictoriaMetrics Global Select
        483 +      │ Queries all regions
        484 +      ├─── ord2 (North America)
        485 +      ├─── mad2 (Europe)
        486 +      └─── osa1 (Asia Pacific)
        487 +      │
        488 +      ▼ Combined results
        489 +  Grafana Dashboard
        490 +  ```
        491 +  
        492 +  **How It's Deployed**:
        493 +  - Runs in **Kubernetes**
        494 +  - Managed by **ArgoCD** (GitOps)
        495 +  - Configuration in `ops/o11y-helm-charts`
        496 +  - Manifests in `ops/palantir`
        497 +  
        498 +  **Important Files**:
        499 +  - Configuration: `o11y-helm-charts/values-files/victoriametrics-{dc}-{country}-{env}.yaml`
        500 +  - Documentation: `terraform-observability-team/docs/content/Services/VictoriaMetrics/`
        501 +  
        502 +  ### Prometheus (Real-Time Metrics Collection)
        503 +  
        504 +  **What is it?**
        505 +  Prometheus is a time-series database that **scrapes** (pulls) metrics from servers and 
            + applications.
        506 +  
        507 +  **Why do we use it?**
        508 +  - **Industry standard** for metrics collection
        509 +  - **Pull-based model** - Prometheus scrapes targets, they don't push to it
        510 +  - **Service discovery** - Automatically finds what to monitor
        511 +  - **Alert evaluation** - Runs alert rules in real-time
        512 +  
        513 +  **Deployment Model - Sharding**:
        514 +  
        515 +  We run **multiple Prometheus instances per datacenter** to handle scale. This is called 
            + **sharding**.
        516 +  
        517 +  ```
        518 +  Datacenter: Newark (ewr1)
        519 +     │
        520 +     ├─ prometheus-1a  (shard 0) ─ Monitors targets with hash % 3 == 0
        521 +     ├─ prometheus-1b  (shard 1) ─ Monitors targets with hash % 3 == 1
        522 +     └─ prometheus-1c  (shard 2) ─ Monitors targets with hash % 3 == 2
        523 +  ```
        524 +  
        525 +  **How Sharding Works**:
        526 +  1. Each Prometheus instance is assigned a **shard number**
        527 +  2. Targets (servers, apps) are distributed across shards using a hash function
        528 +  3. Each shard monitors ~1/3 of the targets
        529 +  
        530 +  **Shard Naming Convention**:
        531 +  - `prometheus-1a` → Shard 0
        532 +  - `prometheus-1b` → Shard 1
        533 +  - `prometheus-1c` → Shard 2
        534 +  - Pattern: `a=0, b=1, c=2, d=3, ...`
        535 +  
        536 +  **High Availability**:
        537 +  - Each shard runs **2 replicas**
        538 +  - If one replica fails, the other keeps working
        539 +  - Both replicas scrape the same targets (duplicate data, but ensures availability)
        540 +  
        541 +  **Configuration**:
        542 +  - Managed by **Salt formula**: `ops/prometheus-formula`
        543 +  - Alert rules in: `ops/prometheus_rules`
        544 +  - Recording rules in: `ops/prometheus_rules`
        545 +  
        546 +  **Where Data Goes**:
        547 +  - **Local storage**: 15 days retention
        548 +  - **Remote write**: Sent to VictoriaMetrics for long-term storage
        549 +  
        550 +  **Important Concepts**:
        551 +  
        552 +  1. **Scraping**: Prometheus pulls metrics from targets every 15-30 seconds
        553 +     ```
        554 +     Prometheus → HTTP GET http://server:9100/metrics → node_exporter
        555 +     ```
        556 +  
        557 +  2. **Service Discovery**: Prometheus automatically finds what to monitor
        558 +     - **Salt grains** - Server metadata tells Prometheus what the server does
        559 +     - **Kubernetes SD** - Auto-discovers pods in Kubernetes
        560 +  
        561 +  3. **Relabeling**: Modifying metric labels before storage
        562 +     - Add datacenter label
        563 +     - Add team ownership label
        564 +     - Filter out unwanted metrics
        565 +  
        566 +  **Important Files**:
        567 +  - Documentation: `terraform-observability-team/docs/content/Services/Prometheus/`
        568 +  - Sharding guide: `terraform-observability-team/docs/content/Services/Prometheus/sharding.md`
        569 +  
        570 +  ### Grafana (Visualization)
        571 +  
        572 +  **What is it?**
        573 +  Grafana is a web application for creating dashboards and visualizations from metrics and logs.
        574 +  
        575 +  **What do we manage?**
        576 +  - **Grafana instances** (production, staging, dev)
        577 +  - **Datasources** (connections to Prometheus, VictoriaMetrics, Loki)
        578 +  - **Dashboards** (via Terraform)
        579 +  - **User permissions** (teams, folders, access control)
        580 +  
        581 +  **How Users Access It**:
        582 +  - **Production**: https://grafana.linode.com
        583 +  - **Staging**: https://grafana-staging.linode.com
        584 +  
        585 +  **Datasource Types**:
        586 +  
        587 +  1. **Prometheus** - Real-time data (last 15 days)
        588 +     - Example: `prometheus-ewr1-1a-us-prod`
        589 +  
        590 +  2. **VictoriaMetrics** - Historical data (13 months)
        591 +     - Example: `victoriametrics-ord2-us-prod`
        592 +  
        593 +  3. **Loki** - Logs
        594 +     - Example: `loki-na-prod`
        595 +  
        596 +  4. **Thanos Query** - Federated Prometheus queries
        597 +     - Queries across multiple Prometheus instances
        598 +  
        599 +  **Dashboard Management**:
        600 +  - Dashboards stored as JSON
        601 +  - Managed in: `ops/terraform-grafana-config`
        602 +  - Changes deployed via Terraform
        603 +  
        604 +  **Permissions**:
        605 +  - Users organized into **teams**
        606 +  - Teams granted access to **folders**
        607 +  - Folders contain dashboards
        608 +  - We manage permissions via Terraform
        609 +  
        610 +  **Important Files**:
        611 +  - Configuration: `ops/terraform-grafana-config`
        612 +  - Documentation: `terraform-observability-team/docs/content/Services/Grafana/`
        613 +  
        614 +  ### ArgoCD (GitOps Deployment)
        615 +  
        616 +  **What is it?**
        617 +  ArgoCD is a GitOps tool that deploys applications to Kubernetes by syncing with Git repositories.
        618 +  
        619 +  **GitOps Concept**:
        620 +  - **Desired state** is defined in Git
        621 +  - ArgoCD **continuously monitors** Git
        622 +  - When Git changes, ArgoCD **automatically syncs** to Kubernetes
        623 +  - Git is the **single source of truth**
        624 +  
        625 +  **How It Works**:
        626 +  
        627 +  ```
        628 +  1. Engineer makes change
        629 +     ↓
        630 +  2. Git repository updated (ops/palantir)
        631 +     ↓
        632 +  3. ArgoCD detects change
        633 +     ↓
        634 +  4. ArgoCD compares Git vs Kubernetes
        635 +     ↓
        636 +  5. ArgoCD applies changes to Kubernetes
        637 +     ↓
        638 +  6. Application updated
        639 +  ```
        640 +  
        641 +  **ArgoCD Instances**:
        642 +  
        643 +  We run one ArgoCD per environment:
        644 +  
        645 +  - **Production**: https://argocd.infra-o11y-apps.iad3.us.prod.linode.com
        646 +    - Manages ~20 production Kubernetes clusters
        647 +    - Uses Git **release tags** (stable)
        648 +  
        649 +  - **Staging**: https://argocd.infra-o11y-apps.rin1.us.staging.linode.com
        650 +    - Manages ~10 staging clusters
        651 +    - Uses Git **main branch** (latest)
        652 +  
        653 +  - **Dev**: https://argocd.infra-o11y-apps.rin1.us.dev.linode.com
        654 +    - Manages ~5 dev clusters
        655 +    - Uses Git **main branch**
        656 +  
        657 +  **Key Concepts**:
        658 +  
        659 +  1. **Application** - A deployed workload
        660 +     - Example: VictoriaMetrics cluster in ord2
        661 +     - Defined by: Name, Git repo, path, target cluster
        662 +  
        663 +  2. **ApplicationSet** - Template for creating multiple similar Applications
        664 +     - Example: Deploy VictoriaMetrics to all LTS clusters
        665 +     - Uses generators to create Applications dynamically
        666 +  
        667 +  3. **Sync Policy**
        668 +     - **Manual**: Changes require clicking "Sync" button
        669 +     - **Automatic**: Changes deploy automatically
        670 +  
        671 +  4. **Health**
        672 +     - **Healthy**: All resources running correctly
        673 +     - **Degraded**: Some resources have issues
        674 +     - **Progressing**: Deployment in progress
        675 +  
        676 +  **Repositories ArgoCD Watches**:
        677 +  
        678 +  1. **ops/palantir** - Primary repo for Kubernetes manifests
        679 +     - Kustomize-based overlays
        680 +     - Bootstrap configs
        681 +     - Core components
        682 +  
        683 +  2. **ops/o11y-helm-charts** - Generates Application manifests
        684 +     - Helm chart with values files per cluster
        685 +     - Generates ArgoCD Application YAML
        686 +  
        687 +  **Important Files**:
        688 +  - Documentation: `terraform-observability-team/docs/content/Services/ArgoCD/`
        689 +  
        690 +  ### Kubernetes Clusters
        691 +  
        692 +  **What Clusters Do We Manage?**
        693 +  
        694 +  The team manages **30+ Kubernetes clusters** across three types:
        695 +  
        696 +  #### 1. infra-o11y-apps Clusters
        697 +  
        698 +  **Purpose**: Run centralized observability services
        699 +  
        700 +  **What Runs Here**:
        701 +  - ArgoCD
        702 +  - Grafana
        703 +  - Some Prometheus instances
        704 +  - Support services (cert-manager, external-dns)
        705 +  
        706 +  **Naming**: `infra-o11y-apps-{dc}-{country}-{env}`
        707 +  - Example: `infra-o11y-apps-iad3-us-prod`
        708 +  
        709 +  **Count**: ~5 clusters (1 per environment + extras)
        710 +  
        711 +  #### 2. VictoriaMetrics Clusters
        712 +  
        713 +  **Purpose**: Run VictoriaMetrics LTS clusters
        714 +  
        715 +  **What Runs Here**:
        716 +  - vminsert pods
        717 +  - vmstorage pods
        718 +  - vmselect pods
        719 +  - Monitoring exporters
        720 +  
        721 +  **Naming**: `victoriametrics-{dc}-{country}-{env}`
        722 +  - Example: `victoriametrics-ord2-us-prod`
        723 +  
        724 +  **Count**: ~15 clusters (NA, EU, AP regions × staging/prod)
        725 +  
        726 +  #### 3. infra-logging Clusters
        727 +  
        728 +  **Purpose**: Run centralized logging infrastructure
        729 +  
        730 +  **What Runs Here**:
        731 +  - Loki distributors
        732 +  - Loki ingesters
        733 +  - Loki queriers
        734 +  - OpenTelemetry collectors (otelgw)
        735 +  
        736 +  **Naming**: `infra-logging-{dc}-{country}-{env}`
        737 +  - Example: `infra-logging-cjj1-us-staging`
        738 +  
        739 +  **Count**: ~10 clusters
        740 +  
        741 +  **Cluster Deployment Process**:
        742 +  
        743 +  Deploying a new cluster involves multiple teams and repositories:
        744 +  
        745 +  ```
        746 +  1. Terraform (Infrastructure)
        747 +     ├─ Repo: Linode/terraform-module-infra
        748 +     ├─ Action: Provision Linodes, IPs, DNS
        749 +     └─ Output: Server nodes created
        750 +  
        751 +  2. Salt (Server Configuration)
        752 +     ├─ Repo: ops/salt-pillar
        753 +     ├─ Action: Accept minion keys, set grains
        754 +     └─ Output: Servers configured
        755 +  
        756 +  3. Vault (Secrets Management)
        757 +     ├─ Action: Store approle secret, kubeconfig
        758 +     ├─ PKI: Add cluster domain to allowed list
        759 +     └─ Output: Secrets available for apps
        760 +  
        761 +  4. Ansible (Kubernetes Installation)
        762 +     ├─ Repo: ops/ansible-playbooks
        763 +     ├─ Playbook: Kubespray
        764 +     └─ Output: Kubernetes cluster running
        765 +  
        766 +  5. ArgoCD (Join Cluster)
        767 +     ├─ Repo: ops/palantir
        768 +     ├─ Command: make join-cluster
        769 +     └─ Output: ArgoCD can deploy to cluster
        770 +  
        771 +  6. Application Configuration
        772 +     ├─ Repo: ops/o11y-helm-charts
        773 +     ├─ Action: Create values file
        774 +     └─ Repo: ops/palantir
        775 +         ├─ Action: Create overlays
        776 +         └─ Output: Apps deployed via ArgoCD
        777 +  ```
        778 +  
        779 +  **Important Files**:
        780 +  - Documentation: `terraform-observability-team/docs/content/Services/Kubernetes 
            + Clusters/cluster-deployment.md`
        781 +  
        782 +  ### Centralized Logging
        783 +  
        784 +  **Purpose**: Collect logs from all infrastructure in one place for searching and alerting.
        785 +  
        786 +  **Architecture**:
        787 +  
        788 +  ```
        789 +  Servers (all DCs)
        790 +      │ Send logs via Fluent Bit / Promtail
        791 +      ▼
        792 +  OpenTelemetry Collector (otelgw) - Per DC
        793 +      │ Aggregates logs, adds labels
        794 +      ▼
        795 +  Loki Distributor - Regional
        796 +      │ Receives logs, hashes by labels
        797 +      ▼
        798 +  Loki Ingester - Regional
        799 +      │ Buffers logs, creates chunks
        800 +      ▼
        801 +  Object Storage (S3)
        802 +      │ Long-term log storage
        803 +      ▼
        804 +  Loki Querier - Regional
        805 +      │ Queries logs from ingesters + S3
        806 +      ▼
        807 +  Grafana - Global
        808 +      │ User searches logs
        809 +  ```
        810 +  
        811 +  **Components**:
        812 +  
        813 +  1. **otelgw** (OpenTelemetry Gateway)
        814 +     - One per datacenter
        815 +     - Aggregates logs from local servers
        816 +     - Adds metadata (datacenter, environment)
        817 +     - Forwards to Loki
        818 +  
        819 +  2. **Loki**
        820 +     - Distributed log aggregation system
        821 +     - Like "Prometheus but for logs"
        822 +     - **Doesn't index log content** (indexes labels only)
        823 +     - Very cost-effective
        824 +  
        825 +  **Loki Components**:
        826 +  - **Distributor**: Receives logs, validates, forwards
        827 +  - **Ingester**: Buffers logs, creates chunks, stores to S3
        828 +  - **Querier**: Queries logs from ingesters and S3
        829 +  - **Query Frontend**: Splits large queries, caching
        830 +  
        831 +  **Log Flow Example**:
        832 +  
        833 +  ```
        834 +  1. Web server generates log:
        835 +     "2025-11-20 10:00:00 GET /api/v4/linodes 200 45ms"
        836 +  
        837 +  2. Fluent Bit on server sends to otelgw:
        838 +     {
        839 +       "log": "GET /api/v4/linodes 200 45ms",
        840 +       "timestamp": "2025-11-20T10:00:00Z"
        841 +     }
        842 +  
        843 +  3. otelgw adds labels:
        844 +     {
        845 +       "log": "GET /api/v4/linodes 200 45ms",
        846 +       "datacenter": "ewr1",
        847 +       "service": "api",
        848 +       "environment": "prod"
        849 +     }
        850 +  
        851 +  4. Loki stores with labels:
        852 +     Labels: {datacenter="ewr1", service="api", environment="prod"}
        853 +     Log Line: "GET /api/v4/linodes 200 45ms"
        854 +  
        855 +  5. Engineer searches in Grafana:
        856 +     Query: {service="api", datacenter="ewr1"}
        857 +  ```
        858 +  
        859 +  **Important Files**:
        860 +  - Documentation: `terraform-observability-team/docs/content/Services/Centralized Logging/`
        861 +  
        862 +  ---
        863 +  
        864 +  ## 6. How Deployments Work
        865 +  
        866 +  This section explains how changes move from your laptop to production.
        867 +  
        868 +  ### Deployment Workflow: This Repository
        869 +  
        870 +  **For terraform-observability-team**:
        871 +  
        872 +  ```
        873 +  ┌─────────────────────────────────────────────────────────────────┐
        874 +  │ Step 1: Developer Creates PR                                    │
        875 +  ├─────────────────────────────────────────────────────────────────┤
        876 +  │  git checkout -b add-new-team-member                            │
        877 +  │  vim team.auto.tfvars  # Add new person                         │
        878 +  │  git commit -m "Add Jane Doe to team"                           │
        879 +  │  git push origin add-new-team-member                            │
        880 +  │  # Create PR on Bits                                            │
        881 +  └─────────────────────────────────────────────────────────────────┘
        882 +                              ▼
        883 +  ┌─────────────────────────────────────────────────────────────────┐
        884 +  │ Step 2: Atlantis Automatically Runs Plan                        │
        885 +  ├─────────────────────────────────────────────────────────────────┤
        886 +  │  Atlantis (bot) comments on PR:                                 │
        887 +  │                                                                  │
        888 +  │  Terraform Plan:                                                │
        889 +  │  + pagerduty_user.jane_doe                                      │
        890 +  │  + github_team_membership.jane_doe                              │
        891 +  │  + bits_team_membership.jane_doe                                │
        892 +  │                                                                  │
        893 +  │  Plan: 3 to add, 0 to change, 0 to destroy                      │
        894 +  └─────────────────────────────────────────────────────────────────┘
        895 +                              ▼
        896 +  ┌─────────────────────────────────────────────────────────────────┐
        897 +  │ Step 3: Team Reviews PR                                         │
        898 +  ├─────────────────────────────────────────────────────────────────┤
        899 +  │  Required Reviewers: ops/sre-o11y team                          │
        900 +  │  ✅ Check Terraform plan looks correct                          │
        901 +  │  ✅ Approve PR                                                  │
        902 +  └─────────────────────────────────────────────────────────────────┘
        903 +                              ▼
        904 +  ┌─────────────────────────────────────────────────────────────────┐
        905 +  │ Step 4: Merge PR                                                │
        906 +  ├─────────────────────────────────────────────────────────────────┤
        907 +  │  PR is merged to main branch                                    │
        908 +  │  (Atlantis does NOT apply automatically)                        │
        909 +  └─────────────────────────────────────────────────────────────────┘
        910 +                              ▼
        911 +  ┌─────────────────────────────────────────────────────────────────┐
        912 +  │ Step 5: Apply Changes                                           │
        913 +  ├─────────────────────────────────────────────────────────────────┤
        914 +  │  Comment on merged PR: "atlantis apply"                         │
        915 +  │                                                                  │
        916 +  │  Atlantis runs:                                                 │
        917 +  │  + Creates PagerDuty user                                       │
        918 +  │  + Adds to GitHub teams                                         │
        919 +  │  + Adds to Bits teams                                           │
        920 +  │                                                                  │
        921 +  │  Apply Complete! Resources: 3 added, 0 changed, 0 destroyed     │
        922 +  └─────────────────────────────────────────────────────────────────┘
        923 +                              ▼
        924 +  ┌─────────────────────────────────────────────────────────────────┐
        925 +  │ Step 6: Verify                                                  │
        926 +  ├─────────────────────────────────────────────────────────────────┤
        927 +  │  ✅ Jane receives PagerDuty invite                              │
        928 +  │  ✅ Jane appears in GitHub org                                  │
        929 +  │  ✅ Jane can access Bits repos                                  │
        930 +  └─────────────────────────────────────────────────────────────────┘
        931 +  ```
        932 +  
        933 +  **Key Points**:
        934 +  - **Atlantis automates Terraform**
        935 +  - **Plan is automatic**, apply is manual
        936 +  - **Always review the plan** before applying
        937 +  - **Vault secrets** are automatically loaded
        938 +  - **Apply happens via comment**: `atlantis apply`
        939 +  
        940 +  ### Deployment Workflow: Application Changes (Kubernetes)
        941 +  
        942 +  **For ops/o11y-helm-charts → ops/palantir → ArgoCD**:
        943 +  
        944 +  ```
        945 +  ┌─────────────────────────────────────────────────────────────────┐
        946 +  │ Step 1: Change Application Configuration                        │
        947 +  ├─────────────────────────────────────────────────────────────────┤
        948 +  │  Repo: ops/o11y-helm-charts                                     │
        949 +  │                                                                  │
        950 +  │  Example: Upgrade VictoriaMetrics version                       │
        951 +  │                                                                  │
        952 +  │  Edit: values-files/victoriametrics-ord2-us-staging.yaml        │
        953 +  │  Change:                                                         │
        954 +  │    victoriametrics:                                             │
        955 +  │      version: v1.99.0  →  v1.100.0                              │
        956 +  │                                                                  │
        957 +  │  git commit -m "Upgrade VictoriaMetrics to v1.100.0"            │
        958 +  │  git push, create PR, get review, merge                         │
        959 +  └─────────────────────────────────────────────────────────────────┘
        960 +                              ▼
        961 +  ┌─────────────────────────────────────────────────────────────────┐
        962 +  │ Step 2: GitHub Action Auto-Creates Palantir PR                  │
        963 +  ├─────────────────────────────────────────────────────────────────┤
        964 +  │  GitHub Action in o11y-helm-charts runs:                        │
        965 +  │    1. Renders Helm chart with new values                        │
        966 +  │    2. Generates updated Application manifests                   │
        967 +  │    3. Creates PR in ops/palantir with changes                   │
        968 +  │                                                                  │
        969 +  │  Palantir PR shows:                                             │
        970 +  │    - Updated VictoriaMetrics Application YAML                   │
        971 +  │    - New container image version                                │
        972 +  └─────────────────────────────────────────────────────────────────┘
        973 +                              ▼
        974 +  ┌─────────────────────────────────────────────────────────────────┐
        975 +  │ Step 3: Review and Merge Palantir PR                            │
        976 +  ├─────────────────────────────────────────────────────────────────┤
        977 +  │  Team reviews Palantir PR:                                      │
        978 +  │    ✅ Check image version is correct                            │
        979 +  │    ✅ Verify only staging cluster affected                      │
        980 +  │    ✅ Approve and merge                                         │
        981 +  └─────────────────────────────────────────────────────────────────┘
        982 +                              ▼
        983 +  ┌─────────────────────────────────────────────────────────────────┐
        984 +  │ Step 4: ArgoCD Detects Change                                   │
        985 +  ├─────────────────────────────────────────────────────────────────┤
        986 +  │  ArgoCD polls ops/palantir every 3 minutes                      │
        987 +  │  Detects: VictoriaMetrics Application changed                  │
        988 +  │  Status: "OutOfSync"                                            │
        989 +  └─────────────────────────────────────────────────────────────────┘
        990 +                              ▼
        991 +  ┌─────────────────────────────────────────────────────────────────┐
        992 +  │ Step 5: Manual Sync in ArgoCD                                   │
        993 +  ├─────────────────────────────────────────────────────────────────┤
        994 +  │  Engineer logs into ArgoCD UI                                   │
        995 +  │  Clicks "Sync" on VictoriaMetrics Application                   │
        996 +  │  ArgoCD applies changes to Kubernetes:                          │
        997 +  │    - Rolls out new vminsert pods (v1.100.0)                     │
        998 +  │    - Rolls out new vmselect pods (v1.100.0)                     │
        999 +  │    - Rolls out new vmstorage pods (v1.100.0)                    │
       1000 +  │  Status: "Synced" + "Healthy"                                   │
       1001 +  └─────────────────────────────────────────────────────────────────┘
       1002 +                              ▼
       1003 +  ┌─────────────────────────────────────────────────────────────────┐
       1004 +  │ Step 6: Verify Deployment                                       │
       1005 +  ├─────────────────────────────────────────────────────────────────┤
       1006 +  │  kubectl get pods -n victoriametrics                            │
       1007 +  │  → All pods running with new image                              │
       1008 +  │                                                                  │
       1009 +  │  Check Grafana dashboards                                       │
       1010 +  │  → Metrics still flowing                                        │
       1011 +  │  → No errors in logs                                            │
       1012 +  └─────────────────────────────────────────────────────────────────┘
       1013 +  ```
       1014 +  
       1015 +  **Key Points**:
       1016 +  - **Two PRs required**: One in o11y-helm-charts, one auto-generated in palantir
       1017 +  - **Staging uses main branch**, production uses release tags
       1018 +  - **Manual sync in ArgoCD** (staging/dev auto-sync, prod is manual)
       1019 +  - **Always verify** after deployment
       1020 +  
       1021 +  ### Repository Relationship Diagram
       1022 +  
       1023 +  ```
       1024 +  ┌────────────────────────────────────────────────────────────────┐
       1025 +  │                    SOURCE OF TRUTH                              │
       1026 +  │                                                                 │
       1027 +  │  ┌──────────────────────┐                                      │
       1028 +  │  │ ops/o11y-helm-charts │ ← Engineers make changes here        │
       1029 +  │  │ (Helm Chart + Values)│                                      │
       1030 +  │  └──────────┬───────────┘                                      │
       1031 +  │             │                                                   │
       1032 +  │             │ GitHub Action                                    │
       1033 +  │             │ (helm template + render)                         │
       1034 +  │             ▼                                                   │
       1035 +  │  ┌──────────────────────┐                                      │
       1036 +  │  │  ops/palantir        │ ← Auto-generated manifests           │
       1037 +  │  │  (Kubernetes YAML)   │                                      │
       1038 +  │  └──────────┬───────────┘                                      │
       1039 +  │             │                                                   │
       1040 +  │             │ ArgoCD polls every 3min                          │
       1041 +  │             ▼                                                   │
       1042 +  │  ┌──────────────────────┐                                      │
       1043 +  │  │  ArgoCD              │ ← Detects changes, syncs             │
       1044 +  │  └──────────┬───────────┘                                      │
       1045 +  │             │                                                   │
       1046 +  │             │ kubectl apply                                    │
       1047 +  │             ▼                                                   │
       1048 +  │  ┌──────────────────────┐                                      │
       1049 +  │  │  Kubernetes Cluster  │ ← Applications running               │
       1050 +  │  └──────────────────────┘                                      │
       1051 +  └────────────────────────────────────────────────────────────────┘
       1052 +  ```
       1053 +  
       1054 +  ---
       1055 +  
       1056 +  ## 7. Important Files You Need to Know
       1057 +  
       1058 +  ### Critical Files (Edit With Care!)
       1059 +  
       1060 +  #### `/team.auto.tfvars`
       1061 +  
       1062 +  **What**: Single source of truth for team membership
       1063 +  
       1064 +  **When to Edit**:
       1065 +  - Adding/removing team members
       1066 +  - Changing on-call rotation
       1067 +  - Modifying repository permissions
       1068 +  
       1069 +  **Structure**:
       1070 +  
       1071 +  ```hcl
       1072 +  # Team Members
       1073 +  observability_members = {
       1074 +    jdoe = {
       1075 +      name                 = "Jane Doe"
       1076 +      email                = "jane.doe@akamai.com"
       1077 +      job_title            = "Senior SRE"
       1078 +      github_username      = "jdoe"
       1079 +      github_admin         = false          # Admin on linode-obs GitHub org
       1080 +      bits_team_maintainer = true           # Maintainer role on Bits teams
       1081 +      bits_orgs            = ["ops", "Linode"]  # Which Bits orgs
       1082 +      pd_enabled           = true           # Add to PagerDuty
       1083 +    }
       1084 +    # ... more team members
       1085 +  }
       1086 +  
       1087 +  # On-call Rotation (Primary)
       1088 +  observability_oncall_primary = [
       1089 +    "current_oncall",   # MUST be first (currently on-call)
       1090 +    "person2",
       1091 +    "person3",
       1092 +    "new_person"        # Add new people at END
       1093 +  ]
       1094 +  
       1095 +  # On-call Schedule Start Time
       1096 +  # IMPORTANT: Set to Monday 11am ET of current on-call's shift
       1097 +  pagerduty_schedule_starttime = "2025-11-18T11:00:00-05:00"
       1098 +  
       1099 +  # Repository Configurations
       1100 +  observability_bits_repos = {
       1101 +    ops = [
       1102 +      {
       1103 +        name  = "prometheus_rules"
       1104 +        hooks = ["notification-o11y-prs", "atlantis"]
       1105 +        permissions = {
       1106 +          pull = false
       1107 +          push = true
       1108 +          admin = false
       1109 +        }
       1110 +        branch_protection = {
       1111 +          pattern = "main"
       1112 +          require_code_owners_review = true
       1113 +          required_approving_review_count = 1
       1114 +        }
       1115 +      }
       1116 +      # ... more repos
       1117 +    ]
       1118 +  }
       1119 +  ```
       1120 +  
       1121 +  **Important Notes**:
       1122 +  - **On-call order matters!** Current on-call must be first
       1123 +  - **Adding to on-call**: Put new person LAST
       1124 +  - **Update schedule_starttime** when changing on-call
       1125 +  - **Review carefully** - affects real permissions
       1126 +  
       1127 +  #### `/pagerduty_suppression.auto.tfvars`
       1128 +  
       1129 +  **What**: Alert suppression during maintenance
       1130 +  
       1131 +  **When to Edit**:
       1132 +  - DC scaling events (temporarily suppress alerts)
       1133 +  - Maintenance windows
       1134 +  
       1135 +  **Structure**:
       1136 +  
       1137 +  ```hcl
       1138 +  # DCs with alerts suppressed
       1139 +  observability_suppressed_dcs = [
       1140 +    "iad5",    # Include both formats
       1141 +    "iad05",   # (some alerts use zero-padded)
       1142 +  ]
       1143 +  
       1144 +  # Services with suppression rules
       1145 +  observability_suppressed_services = [
       1146 +    "PJ95HJI",  # Prometheus Alerts service ID
       1147 +    # ... more service IDs
       1148 +  ]
       1149 +  ```
       1150 +  
       1151 +  **Process**:
       1152 +  1. Edit file to add DC
       1153 +  2. `atlantis plan` to verify
       1154 +  3. Merge and `atlantis apply`
       1155 +  4. Suppression active
       1156 +  5. **IMPORTANT**: Remove DC after maintenance!
       1157 +  
       1158 +  #### `/atlantis.yaml`
       1159 +  
       1160 +  **What**: Atlantis automation configuration
       1161 +  
       1162 +  **When to Edit**: Rarely (only if changing Terraform workflow)
       1163 +  
       1164 +  **Key Settings**:
       1165 +  
       1166 +  ```yaml
       1167 +  version: 3
       1168 +  automerge: false
       1169 +  delete_source_branch_on_merge: false
       1170 +  
       1171 +  projects:
       1172 +  - name: terraform-observability-team
       1173 +    dir: .
       1174 +    workspace: default
       1175 +    terraform_version: v1.5.7
       1176 +  
       1177 +    # Automatic plan on PR
       1178 +    autoplan:
       1179 +      when_modified: ["*.tf", "*.tfvars"]
       1180 +      enabled: true
       1181 +  
       1182 +    # Requirements before apply
       1183 +    apply_requirements:
       1184 +      - approved      # PR must be approved
       1185 +      - mergeable     # PR must be mergeable
       1186 +  
       1187 +    workflow: observability-team
       1188 +  
       1189 +  workflows:
       1190 +    observability-team:
       1191 +      plan:
       1192 +        steps:
       1193 +        - run: vault login -method=oidc  # Authenticate to Vault
       1194 +        - init
       1195 +        - plan
       1196 +      apply:
       1197 +        steps:
       1198 +        - run: vault login -method=oidc
       1199 +        - apply
       1200 +  ```
       1201 +  
       1202 +  **Don't Touch Unless**: You know what you're doing with Atlantis
       1203 +  
       1204 +  ### Configuration Files
       1205 +  
       1206 +  #### `/.envrc`
       1207 +  
       1208 +  **What**: Environment variables for local development (loaded by direnv)
       1209 +  
       1210 +  **Contents**:
       1211 +  
       1212 +  ```bash
       1213 +  export VAULT_ADDR="https://vault.infra.linode.com"
       1214 +  export VAULT_NAMESPACE="infra"
       1215 +  
       1216 +  # Auto-login to Vault when entering directory
       1217 +  vault login -method=oidc username=$USER
       1218 +  ```
       1219 +  
       1220 +  **Usage**:
       1221 +  1. Install direnv: `brew install direnv`
       1222 +  2. Add to shell: `eval "$(direnv hook zsh)"`
       1223 +  3. Allow in this directory: `direnv allow`
       1224 +  4. Now auto-authenticated to Vault when you `cd` here
       1225 +  
       1226 +  #### `/.pre-commit-config.yaml`
       1227 +  
       1228 +  **What**: Automated code quality checks before commits
       1229 +  
       1230 +  **Hooks**:
       1231 +  - `terraform fmt` - Format Terraform code
       1232 +  - `terraform validate` - Validate Terraform syntax
       1233 +  - `markdownlint` - Lint Markdown docs
       1234 +  - `vale` - Prose style checking
       1235 +  - `prettier` - Format YAML/JSON/Markdown
       1236 +  - `trailing-whitespace` - Remove trailing whitespace
       1237 +  - `end-of-file-fixer` - Ensure files end with newline
       1238 +  
       1239 +  **Setup**:
       1240 +  ```bash
       1241 +  pre-commit install --install-hooks
       1242 +  ```
       1243 +  
       1244 +  **Usage**: Runs automatically on `git commit`
       1245 +  
       1246 +  **Skip if needed** (not recommended):
       1247 +  ```bash
       1248 +  git commit --no-verify
       1249 +  ```
       1250 +  
       1251 +  #### `/.mise.toml`
       1252 +  
       1253 +  **What**: Task runner configuration (like Makefile but better)
       1254 +  
       1255 +  **Common Tasks**:
       1256 +  
       1257 +  ```bash
       1258 +  # Create new on-call log for this week
       1259 +  mise run oncall
       1260 +  
       1261 +  # Create new proposal
       1262 +  mise run proposal
       1263 +  
       1264 +  # Run pre-commit on all files
       1265 +  mise run pre-commit-all
       1266 +  
       1267 +  # Serve documentation locally
       1268 +  mise run docs-serve
       1269 +  ```
       1270 +  
       1271 +  **View all tasks**:
       1272 +  ```bash
       1273 +  mise tasks
       1274 +  ```
       1275 +  
       1276 +  ### Documentation Files
       1277 +  
       1278 +  #### `/docs/config.toml`
       1279 +  
       1280 +  **What**: Hugo site configuration
       1281 +  
       1282 +  **Key Settings**:
       1283 +  
       1284 +  ```toml
       1285 +  baseURL = "https://bits.linode.com/pages/ops/terraform-observability-team/"
       1286 +  title = "SRE Observability Team"
       1287 +  theme = "docsy"
       1288 +  
       1289 +  [params]
       1290 +    description = "SRE Observability Team Documentation"
       1291 +    github_repo = "https://bits.linode.com/ops/terraform-observability-team"
       1292 +    github_branch = "main"
       1293 +  ```
       1294 +  
       1295 +  **When to Edit**: Changing site metadata, theme settings
       1296 +  
       1297 +  ---
       1298 +  
       1299 +  ## 8. Documentation Structure
       1300 +  
       1301 +  ### How Documentation is Organized
       1302 +  
       1303 +  The team uses **Hugo** with the **Docsy** theme for documentation.
       1304 +  
       1305 +  **Why Hugo?**
       1306 +  - Static site generator (fast, secure)
       1307 +  - Markdown-based (easy to write)
       1308 +  - Version controlled (in git)
       1309 +  - Searchable
       1310 +  - Navigation sidebar auto-generated
       1311 +  
       1312 +  ### Content Types
       1313 +  
       1314 +  #### Handbooks (`/docs/content/Handbooks/`)
       1315 +  
       1316 +  **Purpose**: How the team operates
       1317 +  
       1318 +  **Files**:
       1319 +  - `on-call.md` - On-call guide (schedule, PagerDuty setup, responsibilities)
       1320 +  - `tools.md` - Standardized tooling (Go, Jsonnet, Kubernetes, etc.)
       1321 +  - `git-conventions.md` - Commit message format, branching strategy
       1322 +  - `docs/` - How to write documentation
       1323 +  
       1324 +  **When to Update**: Process changes, new tools adopted
       1325 +  
       1326 +  #### Services (`/docs/content/Services/`)
       1327 +  
       1328 +  **Purpose**: Documentation for each service we manage
       1329 +  
       1330 +  **Structure**:
       1331 +  ```
       1332 +  Services/
       1333 +  ├── ArgoCD/
       1334 +  │   ├── _index.md              # Overview
       1335 +  │   ├── repositories.md        # Repo management
       1336 +  │   └── troubleshooting.md
       1337 +  ├── VictoriaMetrics/
       1338 +  │   ├── _index.md
       1339 +  │   ├── architecture.md
       1340 +  │   ├── upgrade.md
       1341 +  │   └── troubleshooting.md
       1342 +  ├── Prometheus/
       1343 +  │   ├── _index.md
       1344 +  │   ├── sharding.md
       1345 +  │   ├── Updating/
       1346 +  │   │   └── updating.md
       1347 +  │   └── troubleshooting.md
       1348 +  └── ...
       1349 +  ```
       1350 +  
       1351 +  **When to Update**: Service upgrades, architecture changes, new troubleshooting steps
       1352 +  
       1353 +  #### MOPs (`/docs/content/mops/`)
       1354 +  
       1355 +  **Purpose**: Manual Operations Procedures - step-by-step guides for complex tasks
       1356 +  
       1357 +  **MOP Structure**:
       1358 +  
       1359 +  ```markdown
       1360 +  # MOP: Prometheus Shard
       1361 +  
       1362 +  ## Overview
       1363 +  High-level description of the procedure.
       1364 +  
       1365 +  ## Prerequisites
       1366 +  - [ ] Access to Salt master
       1367 +  - [ ] 2-4 hours of focused time
       1368 +  - [ ] Approval from team lead
       1369 +  
       1370 +  ## Procedure
       1371 +  
       1372 +  ### Step 1: Prepare
       1373 +  Detailed instructions...
       1374 +  
       1375 +  ### Step 2: Execute
       1376 +  More instructions...
       1377 +  
       1378 +  ## Verification
       1379 +  How to verify the procedure succeeded.
       1380 +  
       1381 +  ## Rollback Plan
       1382 +  How to undo changes if something goes wrong.
       1383 +  
       1384 +  ## References
       1385 +  - [Related Documentation](link)
       1386 +  ```
       1387 +  
       1388 +  **Examples**:
       1389 +  - `prometheus-shard.md` - Adding a new Prometheus shard
       1390 +  - `victoriametrics-cluster-upgrade.md` - Upgrading VictoriaMetrics
       1391 +  
       1392 +  **When to Create**: Complex multi-step procedures that are done infrequently
       1393 +  
       1394 +  #### Proposals (`/docs/content/proposals/`)
       1395 +  
       1396 +  **Purpose**: Design documents for major changes
       1397 +  
       1398 +  **Proposal Naming**: `OP-##-descriptive-name.md`
       1399 +  - OP = Observability Proposal
       1400 +  - ## = Sequential number (01, 02, 03...)
       1401 +  
       1402 +  **Proposal Template**:
       1403 +  
       1404 +  ```markdown
       1405 +  ---
       1406 +  title: "OP-05: My Proposal Title"
       1407 +  status: "accepted"  # draft | accepted | rejected | done
       1408 +  date: 2025-11-20
       1409 +  ---
       1410 +  
       1411 +  ## Summary
       1412 +  One paragraph summary.
       1413 +  
       1414 +  ## Motivation
       1415 +  Why are we doing this?
       1416 +  
       1417 +  ## Proposal
       1418 +  Detailed design.
       1419 +  
       1420 +  ## Alternatives Considered
       1421 +  What else did we think about?
       1422 +  
       1423 +  ## Implementation Plan
       1424 +  How will we do this?
       1425 +  
       1426 +  ## Success Metrics
       1427 +  How do we know it worked?
       1428 +  ```
       1429 +  
       1430 +  **Statuses**:
       1431 +  - `draft` - Being written
       1432 +  - `accepted` - Approved, not implemented
       1433 +  - `done` - Implemented
       1434 +  - `rejected` - Not approved
       1435 +  
       1436 +  **Creating a Proposal**:
       1437 +  ```bash
       1438 +  cd docs/
       1439 +  mise run proposal
       1440 +  # OR
       1441 +  hugo new proposals/OP-##-my-proposal.md
       1442 +  ```
       1443 +  
       1444 +  **Approval Process**:
       1445 +  1. Create proposal as draft
       1446 +  2. Share with team for feedback
       1447 +  3. Present in team meeting
       1448 +  4. Minimum 2 approvals required
       1449 +  5. Update status to "accepted"
       1450 +  
       1451 +  #### On-call Logs (`/docs/content/on-call/YYYY/`)
       1452 +  
       1453 +  **Purpose**: Weekly logs of on-call work
       1454 +  
       1455 +  **Structure**:
       1456 +  
       1457 +  ```markdown
       1458 +  ---
       1459 +  title: "On-call: 2025-11-18"
       1460 +  date: 2025-11-18
       1461 +  author: "Jane Doe"
       1462 +  ---
       1463 +  
       1464 +  ## Summary
       1465 +  Brief summary of the week.
       1466 +  
       1467 +  ## Incidents
       1468 +  ### [INC-1234] Production Prometheus Down
       1469 +  - **When**: 2025-11-18 14:00 UTC
       1470 +  - **Impact**: 5 minutes of data loss
       1471 +  - **Root Cause**: Out of disk space
       1472 +  - **Resolution**: Cleaned up old WAL files
       1473 +  - **Follow-up**: Created ticket to increase disk size
       1474 +  
       1475 +  ## Reliability Improvements
       1476 +  - Automated disk cleanup script
       1477 +  - Added disk space alerting
       1478 +  
       1479 +  ## Intake & Requests
       1480 +  - Granted Grafana access to 3 new users
       1481 +  - Helped Product team with dashboard creation
       1482 +  
       1483 +  ## Notes
       1484 +  - VictoriaMetrics upgrade planned for next week
       1485 +  - Need to review Prometheus sharding in iad5
       1486 +  ```
       1487 +  
       1488 +  **Creating On-call Log**:
       1489 +  ```bash
       1490 +  cd docs/
       1491 +  mise run oncall
       1492 +  # Creates: on-call/2025/2025-11-18.md (for current Monday)
       1493 +  ```
       1494 +  
       1495 +  **Handoff Process**:
       1496 +  1. Monday 11am ET - on-call shift changes
       1497 +  2. Outgoing on-call fills out summary
       1498 +  3. Posts link in #o11y-core Slack thread
       1499 +  4. Incoming on-call reads to catch up
       1500 +  
       1501 +  #### Runbooks (`/docs/content/Runbooks/`)
       1502 +  
       1503 +  **Purpose**: How to respond to specific alerts
       1504 +  
       1505 +  **Runbook Structure**:
       1506 +  
       1507 +  ```markdown
       1508 +  # Alert: HighMemoryUsage
       1509 +  
       1510 +  ## Summary
       1511 +  This alert fires when a server's memory usage exceeds 90% for 5 minutes.
       1512 +  
       1513 +  ## Impact
       1514 +  - Potential performance degradation
       1515 +  - Risk of OOM killer terminating processes
       1516 +  
       1517 +  ## Investigation Steps
       1518 +  1. Check which process is using memory:
       1519 +     ```bash
       1520 +     ssh server
       1521 +     top -o %MEM
       1522 +     ```
       1523 +  
       1524 +  2. Check for memory leaks:
       1525 +     ```bash
       1526 +     ps aux --sort=-%mem | head -n 10
       1527 +     ```
       1528 +  
       1529 +  ## Resolution
       1530 +  - Restart offending process
       1531 +  - Increase memory if consistently high
       1532 +  - Check for memory leak in application
       1533 +  
       1534 +  ## Escalation
       1535 +  If you can't resolve in 30 minutes, escalate to:
       1536 +  - #team-infrastructure
       1537 +  ```
       1538 +  
       1539 +  **When to Create**: For any alert that pages
       1540 +  
       1541 +  ### Writing Documentation
       1542 +  
       1543 +  **Creating New Pages**:
       1544 +  
       1545 +  ```bash
       1546 +  cd docs/
       1547 +  
       1548 +  # New service documentation
       1549 +  hugo new Services/MyService/_index.md
       1550 +  
       1551 +  # New MOP
       1552 +  hugo new mops/my-procedure.md
       1553 +  
       1554 +  # New proposal
       1555 +  mise run proposal
       1556 +  
       1557 +  # New on-call log
       1558 +  mise run oncall
       1559 +  ```
       1560 +  
       1561 +  **Markdown Tips**:
       1562 +  
       1563 +  ```markdown
       1564 +  # Headers
       1565 +  Use # for title, ## for sections, ### for subsections
       1566 +  
       1567 +  # Links
       1568 +  [Link Text](https://example.com)
       1569 +  [Internal Link]({{< ref "path/to/page.md" >}})
       1570 +  
       1571 +  # Code Blocks
       1572 +  ```bash
       1573 +  command here
       1574 +  ```
       1575 +  
       1576 +  # Images
       1577 +  ![Alt Text](/static/images/diagram.png)
       1578 +  
       1579 +  # Admonitions (special boxes)
       1580 +  {{< alert title="Warning" >}}
       1581 +  This is important!
       1582 +  {{< /alert >}}
       1583 +  ```
       1584 +  
       1585 +  **Documentation Standards**:
       1586 +  1. **Clarity**: Write for someone unfamiliar with the topic
       1587 +  2. **Examples**: Include real examples, not just theory
       1588 +  3. **Up-to-date**: Update docs when processes change
       1589 +  4. **Searchable**: Use descriptive headers and titles
       1590 +  
       1591 +  **Building Locally**:
       1592 +  
       1593 +  ```bash
       1594 +  cd docs/
       1595 +  mise run docs-serve
       1596 +  # Open http://localhost:1313
       1597 +  ```
       1598 +  
       1599 +  **Publishing**:
       1600 +  - Merged to `main` branch → GitHub Action builds → Published to Bits Pages
       1601 +  
       1602 +  ---
       1603 +  
       1604 +  ## 9. Day-to-Day Operations
       1605 +  
       1606 +  ### On-call Responsibilities
       1607 +  
       1608 +  **On-call Shift**: Monday 11am ET → Next Monday 11am ET
       1609 +  
       1610 +  **Primary On-call Duties**:
       1611 +  
       1612 +  1. **Respond to Pages** (Critical alerts)
       1613 +     - Acknowledgment: ≤ 5 minutes
       1614 +     - Begin investigation immediately
       1615 +     - Update incident ticket
       1616 +     - Escalate if needed
       1617 +  
       1618 +  2. **Monitor Warnings** (Non-critical alerts)
       1619 +     - Acknowledgment: ≤ 12 hours
       1620 +     - Investigate during business hours
       1621 +     - Create tickets for follow-up
       1622 +  
       1623 +  3. **Handle Intake Requests**
       1624 +     - Grafana permission requests
       1625 +     - Nagios user management
       1626 +     - Quick questions in Slack
       1627 +  
       1628 +  4. **PR Reviews**
       1629 +     - Review PRs for ops/sre-o11y repos
       1630 +     - Priority: Blocking changes first
       1631 +  
       1632 +  5. **Reliability Improvement**
       1633 +     - Choose ONE reliability task per week
       1634 +     - Examples: Automate toil, improve documentation, fix flaky alerts
       1635 +  
       1636 +  6. **Attend Post-Mortems**
       1637 +     - For incidents you responded to
       1638 +     - Share learnings with team
       1639 +  
       1640 +  **On-call Handoff**:
       1641 +  1. Fill out on-call log summary
       1642 +  2. Post in #o11y-core Slack thread
       1643 +  3. Highlight any ongoing issues
       1644 +  4. Transfer any active incidents
       1645 +  
       1646 +  ### Common Daily Tasks
       1647 +  
       1648 +  #### Reviewing PRs
       1649 +  
       1650 +  **Repositories We Review**:
       1651 +  - `ops/prometheus_rules`
       1652 +  - `ops/prometheus-formula`
       1653 +  - `ops/o11y-helm-charts`
       1654 +  - `ops/palantir`
       1655 +  - `ops/loki_rules`
       1656 +  - `ops/terraform-grafana-config`
       1657 +  - `terraform-observability-team`
       1658 +  
       1659 +  **Review Checklist**:
       1660 +  - [ ] Read PR description
       1661 +  - [ ] Check CI passes
       1662 +  - [ ] Review code changes
       1663 +  - [ ] Verify Terraform plan (if applicable)
       1664 +  - [ ] Check for secrets in diff
       1665 +  - [ ] Ensure tests added (if applicable)
       1666 +  - [ ] Approve or request changes
       1667 +  
       1668 +  **Slack Notifications**: #notification-o11y-prs
       1669 +  
       1670 +  #### Granting Grafana Access
       1671 +  
       1672 +  **Request Format**: Usually in #sre-observability or Jira (OY project)
       1673 +  
       1674 +  **Process**:
       1675 +  1. Determine access level needed
       1676 +     - Viewer: Read dashboards
       1677 +     - Editor: Create/edit dashboards
       1678 +     - Admin: Manage users (rare)
       1679 +  
       1680 +  2. Add user in Terraform:
       1681 +     ```bash
       1682 +     cd ~/path/to/terraform-grafana-config
       1683 +     vim users.tf
       1684 +     # Add user definition
       1685 +     git commit -m "Add Jane Doe to Grafana"
       1686 +     git push, create PR
       1687 +     ```
       1688 +  
       1689 +  3. After PR merged:
       1690 +     - Atlantis applies
       1691 +     - User receives invite email
       1692 +  
       1693 +  4. Notify requester
       1694 +  
       1695 +  **Time-sensitive**: Try to complete within 1 business day
       1696 +  
       1697 +  #### Monitoring Alerts
       1698 +  
       1699 +  **Alert Channels**:
       1700 +  - **#notification-o11y** - All observability alerts
       1701 +  - **PagerDuty** - Critical alerts (pages)
       1702 +  - **Email** - Low-priority warnings
       1703 +  
       1704 +  **Triage Process**:
       1705 +  
       1706 +  1. **Check Alert Severity**
       1707 +     - **Critical**: Page immediately
       1708 +     - **Warning**: Investigate during business hours
       1709 +     - **Info**: Log for awareness
       1710 +  
       1711 +  2. **Check Runbook**
       1712 +     - Most alerts link to runbook
       1713 +     - Follow investigation steps
       1714 +  
       1715 +  3. **Create Incident Ticket** (if needed)
       1716 +     - Jira project: OY
       1717 +     - Include alert details
       1718 +     - Track resolution
       1719 +  
       1720 +  4. **Silence if Necessary**
       1721 +     ```bash
       1722 +     # Silence alert for maintenance
       1723 +     amtool silence add \
       1724 +       --alertmanager.url=https://alertmanager.linode.com \
       1725 +       --comment="Datacenter maintenance" \
       1726 +       --duration=2h \
       1727 +       alertname="HighCPU" \
       1728 +       datacenter="iad5"
       1729 +     ```
       1730 +  
       1731 +  5. **Post-Incident**
       1732 +     - Document in on-call log
       1733 +     - Create follow-up tickets
       1734 +     - Update runbook if needed
       1735 +  
       1736 +  ### Weekly Team Rituals
       1737 +  
       1738 +  **Monday 11am ET: On-call Handoff**
       1739 +  - Outgoing on-call posts summary
       1740 +  - Incoming on-call reviews
       1741 +  
       1742 +  **Wednesday 10am ET: Team Sync** (if scheduled)
       1743 +  - Current work updates
       1744 +  - Blocker discussion
       1745 +  - Knowledge sharing
       1746 +  
       1747 +  **Fridays: Reliability Improvement Time**
       1748 +  - Work on tech debt
       1749 +  - Improve automation
       1750 +  - Update documentation
       1751 +  
       1752 +  ---
       1753 +  
       1754 +  ## 10. Common Tasks with Examples
       1755 +  
       1756 +  Let me show you how to do common tasks step-by-step.
       1757 +  
       1758 +  ### Task 1: Add a New Team Member
       1759 +  
       1760 +  **Scenario**: Jane Doe is joining the team.
       1761 +  
       1762 +  **Steps**:
       1763 +  
       1764 +  ```bash
       1765 +  # 1. Clone repo (if not already)
       1766 +  cd ~/repos
       1767 +  git clone bits.linode.com:ops/terraform-observability-team
       1768 +  cd terraform-observability-team
       1769 +  
       1770 +  # 2. Create branch
       1771 +  git checkout -b add-jane-doe
       1772 +  
       1773 +  # 3. Edit team.auto.tfvars
       1774 +  vim team.auto.tfvars
       1775 +  
       1776 +  # Add to observability_members:
       1777 +  observability_members = {
       1778 +    # ... existing members ...
       1779 +  
       1780 +    jdoe = {
       1781 +      name                 = "Jane Doe"
       1782 +      email                = "jane.doe@akamai.com"
       1783 +      job_title            = "SRE II"
       1784 +      github_username      = "jdoe-akamai"
       1785 +      github_admin         = false
       1786 +      bits_team_maintainer = true
       1787 +      bits_orgs            = ["ops", "Linode"]
       1788 +      pd_enabled           = true
       1789 +    }
       1790 +  }
       1791 +  
       1792 +  # 4. Commit and push
       1793 +  git add team.auto.tfvars
       1794 +  git commit -m "Add Jane Doe to SRE O11y team"
       1795 +  git push origin add-jane-doe
       1796 +  
       1797 +  # 5. Create PR on Bits
       1798 +  # Visit: bits.linode.com/ops/terraform-observability-team
       1799 +  # Click "Create Pull Request"
       1800 +  
       1801 +  # 6. Wait for Atlantis to run plan
       1802 +  # Review plan in PR comments
       1803 +  
       1804 +  # 7. Get PR approved by team member
       1805 +  
       1806 +  # 8. Merge PR
       1807 +  
       1808 +  # 9. Apply changes
       1809 +  # Comment on merged PR: "atlantis apply"
       1810 +  
       1811 +  # 10. Verify
       1812 +  # - Check PagerDuty: Jane should appear in team
       1813 +  # - Check GitHub: Jane should be in linode-obs org
       1814 +  # - Check Bits: Jane should be in ops/sre-o11y team
       1815 +  ```
       1816 +  
       1817 +  ### Task 2: Add Someone to On-call Rotation
       1818 +  
       1819 +  **Scenario**: Jane is trained and ready for on-call.
       1820 +  
       1821 +  **Important**: On-call rotation order matters!
       1822 +  
       1823 +  **Steps**:
       1824 +  
       1825 +  ```bash
       1826 +  # 1. Determine current on-call
       1827 +  # Check PagerDuty schedule or ask in Slack
       1828 +  
       1829 +  # 2. Create branch
       1830 +  git checkout -b add-jane-oncall
       1831 +  
       1832 +  # 3. Edit team.auto.tfvars
       1833 +  vim team.auto.tfvars
       1834 +  
       1835 +  # BEFORE:
       1836 +  observability_oncall_primary = [
       1837 +    "current_oncall",
       1838 +    "person2",
       1839 +    "person3",
       1840 +  ]
       1841 +  pagerduty_schedule_starttime = "2025-11-18T11:00:00-05:00"
       1842 +  
       1843 +  # AFTER:
       1844 +  observability_oncall_primary = [
       1845 +    "current_oncall",   # Must be first!
       1846 +    "person2",
       1847 +    "person3",
       1848 +    "jdoe"              # Add new person LAST
       1849 +  ]
       1850 +  # Update to Monday of current on-call's shift:
       1851 +  pagerduty_schedule_starttime = "2025-11-18T11:00:00-05:00"
       1852 +  
       1853 +  # 4. Commit, push, PR, merge, apply (same as Task 1)
       1854 +  ```
       1855 +  
       1856 +  **Why Order Matters**:
       1857 +  - PagerDuty schedule starts at `pagerduty_schedule_starttime`
       1858 +  - Rotates through the list in order
       1859 +  - If order changes, rotation gets messed up
       1860 +  - **Always**: Current on-call first, new person last
       1861 +  
       1862 +  ### Task 3: Suppress Alerts for Datacenter Maintenance
       1863 +  
       1864 +  **Scenario**: Datacenter iad5 is being scaled, expect alerts.
       1865 +  
       1866 +  **Steps**:
       1867 +  
       1868 +  ```bash
       1869 +  # 1. Create branch
       1870 +  git checkout -b suppress-iad5
       1871 +  
       1872 +  # 2. Edit pagerduty_suppression.auto.tfvars
       1873 +  vim pagerduty_suppression.auto.tfvars
       1874 +  
       1875 +  # Add DCs (include both formats!):
       1876 +  observability_suppressed_dcs = [
       1877 +    "iad5",
       1878 +    "iad05",   # Some alerts use zero-padded
       1879 +  ]
       1880 +  
       1881 +  # 3. Commit and push
       1882 +  git add pagerduty_suppression.auto.tfvars
       1883 +  git commit -m "Suppress alerts for iad5 during scaling"
       1884 +  git push origin suppress-iad5
       1885 +  
       1886 +  # 4. Create PR, get approved, merge
       1887 +  
       1888 +  # 5. Apply immediately
       1889 +  # Comment: "atlantis apply"
       1890 +  
       1891 +  # 6. After maintenance completes, REMOVE suppression
       1892 +  git checkout -b unsuppress-iad5
       1893 +  vim pagerduty_suppression.auto.tfvars
       1894 +  # Remove iad5, iad05 from list
       1895 +  git commit -m "Remove iad5 alert suppression"
       1896 +  # Push, PR, merge, apply
       1897 +  ```
       1898 +  
       1899 +  **Important**: Don't forget to remove suppression after!
       1900 +  
       1901 +  ### Task 4: Upgrade VictoriaMetrics
       1902 +  
       1903 +  **Scenario**: New VictoriaMetrics version released, need to upgrade staging.
       1904 +  
       1905 +  **Steps**:
       1906 +  
       1907 +  ```bash
       1908 +  # 1. Review changelog
       1909 +  # Check: github.com/VictoriaMetrics/VictoriaMetrics/releases
       1910 +  
       1911 +  # 2. Clone o11y-helm-charts
       1912 +  cd ~/repos
       1913 +  git clone bits.linode.com:ops/o11y-helm-charts
       1914 +  cd o11y-helm-charts
       1915 +  
       1916 +  # 3. Create branch
       1917 +  git checkout -b victoriametrics-v1.100.0
       1918 +  
       1919 +  # 4. Update staging cluster values
       1920 +  vim values-files/victoriametrics-ord2-us-staging.yaml
       1921 +  
       1922 +  # Change version:
       1923 +  victoriametrics:
       1924 +    version: v1.99.0  →  v1.100.0
       1925 +  
       1926 +  # 5. Commit and push
       1927 +  git add values-files/victoriametrics-ord2-us-staging.yaml
       1928 +  git commit -m "Upgrade VictoriaMetrics staging to v1.100.0"
       1929 +  git push origin victoriametrics-v1.100.0
       1930 +  
       1931 +  # 6. Create PR, get reviewed, merge
       1932 +  
       1933 +  # 7. Wait for GitHub Action to create Palantir PR
       1934 +  # Check: bits.linode.com/ops/palantir/pulls
       1935 +  
       1936 +  # 8. Review and merge Palantir PR
       1937 +  
       1938 +  # 9. Sync in ArgoCD
       1939 +  # - Visit ArgoCD staging: argocd.infra-o11y-apps.rin1.us.staging.linode.com
       1940 +  # - Find victoriametrics-ord2 application
       1941 +  # - Click "Sync"
       1942 +  # - Wait for deployment to complete
       1943 +  
       1944 +  # 10. Verify
       1945 +  kubectl --context victoriametrics-ord2-us-staging get pods -n victoriametrics
       1946 +  # All pods should show new image version
       1947 +  
       1948 +  # Check Grafana dashboards for errors
       1949 +  
       1950 +  # 11. If successful, repeat for production clusters
       1951 +  ```
       1952 +  
       1953 +  ### Task 5: Create On-call Log
       1954 +  
       1955 +  **Scenario**: It's Monday, time to start your on-call shift.
       1956 +  
       1957 +  **Steps**:
       1958 +  
       1959 +  ```bash
       1960 +  # 1. Navigate to docs directory
       1961 +  cd ~/repos/terraform-observability-team/docs
       1962 +  
       1963 +  # 2. Create on-call log
       1964 +  mise run oncall
       1965 +  # This creates: content/on-call/2025/2025-11-18.md
       1966 +  
       1967 +  # 3. Edit throughout the week
       1968 +  vim content/on-call/2025/2025-11-18.md
       1969 +  
       1970 +  # Add incidents, improvements, notes
       1971 +  
       1972 +  # 4. At end of week, fill out summary
       1973 +  vim content/on-call/2025/2025-11-18.md
       1974 +  
       1975 +  ## Summary
       1976 +  Quiet week. Responded to 2 minor alerts. Improved disk cleanup automation.
       1977 +  
       1978 +  # 5. Commit and push
       1979 +  git add content/on-call/2025/2025-11-18.md
       1980 +  git commit -m "On-call log: 2025-11-18"
       1981 +  git push origin main
       1982 +  
       1983 +  # 6. Post link in Slack
       1984 +  # In #o11y-core:
       1985 +  # "On-call handoff: 
            + https://bits.linode.com/pages/ops/terraform-observability-team/on-call/2025/2025-11-18/"
       1986 +  ```
       1987 +  
       1988 +  ### Task 6: Deploy New Kubernetes Cluster
       1989 +  
       1990 +  **This is a complex multi-day task involving multiple teams.**
       1991 +  
       1992 +  **Prerequisites**:
       1993 +  - Approval from management
       1994 +  - Infrastructure planned (node count, sizes, datacenter)
       1995 +  - Cluster name decided
       1996 +  
       1997 +  **Steps** (abbreviated - see full MOP in docs):
       1998 +  
       1999 +  ```bash
       2000 +  # Day 1: Infrastructure Provisioning
       2001 +  # 1. Create Terraform PR in Linode/terraform-module-infra
       2002 +  # 2. Add cluster definition
       2003 +  # 3. Apply Terraform
       2004 +  # 4. Linodes created
       2005 +  
       2006 +  # Day 2: Server Configuration
       2007 +  # 5. Ask #sre-salt to accept minion keys
       2008 +  # 6. Set Salt grains for cluster nodes
       2009 +  # 7. Run highstate
       2010 +  
       2011 +  # Day 3: Vault & Secrets
       2012 +  # 8. Store cluster approle secret in Vault
       2013 +  # 9. Store kubeconfig in Vault
       2014 +  # 10. Update Vault PKI to allow cluster domain
       2015 +  
       2016 +  # Day 4: Kubernetes Installation
       2017 +  # 11. Run Kubespray Ansible playbook
       2018 +  # 12. Wait 1-2 hours for completion
       2019 +  # 13. Verify cluster accessible
       2020 +  
       2021 +  # Day 5: ArgoCD Integration
       2022 +  # 14. Join cluster to ArgoCD (make join-cluster)
       2023 +  # 15. Create cluster labels
       2024 +  
       2025 +  # Day 6: Application Configuration
       2026 +  # 16. Create values file in o11y-helm-charts
       2027 +  # 17. Create overlays in palantir
       2028 +  # 18. Merge PRs
       2029 +  
       2030 +  # Day 7: Deploy Applications
       2031 +  # 19. Sync applications in ArgoCD
       2032 +  # 20. Verify all apps healthy
       2033 +  # 21. Update documentation
       2034 +  ```
       2035 +  
       2036 +  **This task requires coordination with**:
       2037 +  - SRE Infrastructure (Terraform)
       2038 +  - SRE Salt (minion keys)
       2039 +  - SRE Observability (that's you!)
       2040 +  
       2041 +  ---
       2042 +  
       2043 +  ## 11. Things to Be Careful About
       2044 +  
       2045 +  ### Critical Mistakes to Avoid
       2046 +  
       2047 +  #### 1. DON'T: Edit On-call Rotation Without Updating Start Time
       2048 +  
       2049 +  **Wrong**:
       2050 +  ```hcl
       2051 +  observability_oncall_primary = [
       2052 +    "person2",  # Reordered list
       2053 +    "person1",
       2054 +    "person3",
       2055 +  ]
       2056 +  pagerduty_schedule_starttime = "2025-11-04T11:00:00-05:00"  # OLD DATE
       2057 +  ```
       2058 +  
       2059 +  **Right**:
       2060 +  ```hcl
       2061 +  observability_oncall_primary = [
       2062 +    "person1",  # Current on-call FIRST
       2063 +    "person2",
       2064 +    "person3",
       2065 +  ]
       2066 +  pagerduty_schedule_starttime = "2025-11-18T11:00:00-05:00"  # Current Monday
       2067 +  ```
       2068 +  
       2069 +  **Why**: PagerDuty rotation starts from the date and follows the list. Wrong date = wrong person 
            + on-call.
       2070 +  
       2071 +  #### 2. DON'T: Commit Secrets to Git
       2072 +  
       2073 +  **Wrong**:
       2074 +  ```bash
       2075 +  # In a file:
       2076 +  VAULT_TOKEN=hvs.1234567890abcdef
       2077 +  DATABASE_PASSWORD=supersecret123
       2078 +  ```
       2079 +  
       2080 +  **Right**:
       2081 +  ```bash
       2082 +  # Store in Vault:
       2083 +  vault kv put infra/prod/myapp/secrets password=supersecret123
       2084 +  
       2085 +  # Reference in code:
       2086 +  password = data.vault_generic_secret.myapp.data["password"]
       2087 +  ```
       2088 +  
       2089 +  **Prevention**: Pre-commit hooks help catch this, but always review your diffs!
       2090 +  
       2091 +  #### 3. DON'T: Force Push to Main/Master
       2092 +  
       2093 +  **Wrong**:
       2094 +  ```bash
       2095 +  git push --force origin main
       2096 +  ```
       2097 +  
       2098 +  **Why**: Overwrites history, breaks everyone's local copies, loses work.
       2099 +  
       2100 +  **If you need to fix a commit**: Create a new commit or revert.
       2101 +  
       2102 +  #### 4. DON'T: Skip Atlantis Plan Review
       2103 +  
       2104 +  **Wrong**:
       2105 +  ```
       2106 +  # PR is merged
       2107 +  # Comment: atlantis apply
       2108 +  # (without reading the plan)
       2109 +  ```
       2110 +  
       2111 +  **Right**:
       2112 +  ```
       2113 +  # PR is created
       2114 +  # Atlantis posts plan
       2115 +  # READ THE ENTIRE PLAN
       2116 +  # Verify:
       2117 +  #   - Resources to add/change/destroy
       2118 +  #   - No unexpected changes
       2119 +  #   - No secrets in output
       2120 +  # Then approve, merge, apply
       2121 +  ```
       2122 +  
       2123 +  **Why**: Terraform can destroy resources. Always verify plans.
       2124 +  
       2125 +  #### 5. DON'T: Amend Other People's Commits
       2126 +  
       2127 +  **Wrong**:
       2128 +  ```bash
       2129 +  git commit --amend
       2130 +  git push --force
       2131 +  # (on a commit authored by someone else)
       2132 +  ```
       2133 +  
       2134 +  **Why**: Rewrites history, attributes your changes to someone else.
       2135 +  
       2136 +  **Right**: Create a new commit.
       2137 +  
       2138 +  #### 6. DON'T: Forget to Remove Alert Suppressions
       2139 +  
       2140 +  **Problem**: DC maintenance done, but suppression still active = missed alerts
       2141 +  
       2142 +  **Prevention**:
       2143 +  - Set calendar reminder
       2144 +  - Add comment in PR: "Remove suppression after YYYY-MM-DD"
       2145 +  - Create follow-up ticket
       2146 +  
       2147 +  ### Important Gotchas
       2148 +  
       2149 +  #### Gotcha 1: Staging vs Production Branches
       2150 +  
       2151 +  **Staging**: Uses `main` branch of ops/palantir
       2152 +  **Production**: Uses release tags (e.g., `v2.1.0`)
       2153 +  
       2154 +  **Implication**: Changes appear in staging immediately, production only after release.
       2155 +  
       2156 +  **Process**:
       2157 +  1. Merge to main → staging deployed
       2158 +  2. Test in staging
       2159 +  3. Create release tag
       2160 +  4. Production deployed
       2161 +  
       2162 +  #### Gotcha 2: VictoriaMetrics Upgrade Order
       2163 +  
       2164 +  **Wrong Order**: Global Select first, then LTS clusters
       2165 +  
       2166 +  **Right Order**: LTS clusters first, Global Select last
       2167 +  
       2168 +  **Why**: Global Select queries LTS clusters. If versions are incompatible, queries break.
       2169 +  
       2170 +  #### Gotcha 3: Prometheus Sharding is Time-Consuming
       2171 +  
       2172 +  **Time Required**: 2-8 hours
       2173 +  
       2174 +  **Why**:
       2175 +  - Salt configuration changes
       2176 +  - Restarting Prometheus (data replay)
       2177 +  - Validation
       2178 +  
       2179 +  **Plan Ahead**: Don't start Friday afternoon!
       2180 +  
       2181 +  #### Gotcha 4: Cilium Restarts Required for Network Policies
       2182 +  
       2183 +  **Problem**: New Cilium network policy not enforcing
       2184 +  
       2185 +  **Solution**: Restart Cilium pods
       2186 +  ```bash
       2187 +  kubectl -n kube-system delete pod -l k8s-app=cilium
       2188 +  ```
       2189 +  
       2190 +  **Why**: Some policy changes require pod restart to take effect.
       2191 +  
       2192 +  #### Gotcha 5: Management IPs Can Fail Silently
       2193 +  
       2194 +  **Problem**: Linode provisioned but eth1 (management IP) not configured
       2195 +  
       2196 +  **Check**:
       2197 +  ```bash
       2198 +  ssh server
       2199 +  ip addr show eth1
       2200 +  # Should show IP address
       2201 +  ```
       2202 +  
       2203 +  **Fix**: Re-run Ansible or manually configure.
       2204 +  
       2205 +  ### When to Ask for Help
       2206 +  
       2207 +  **Ask immediately if**:
       2208 +  - Critical alert and you don't know how to fix
       2209 +  - About to run a destructive command
       2210 +  - Unsure about Terraform plan
       2211 +  
       2212 +  **Ask within 30 minutes if**:
       2213 +  - Warning alert and runbook doesn't help
       2214 +  - Stuck on troubleshooting
       2215 +  
       2216 +  **Ask in next business day if**:
       2217 +  - Documentation unclear
       2218 +  - Process question
       2219 +  
       2220 +  **Where to Ask**:
       2221 +  - **#o11y-core** (team private channel) - Team questions
       2222 +  - **#sre-observability** (team public channel) - General questions
       2223 +  - **PagerDuty** (escalate alert) - Can't resolve critical alert
       2224 +  - **Tag @team-sre-o11y** - Need team response
       2225 +  
       2226 +  **It's better to ask than guess!**
       2227 +  
       2228 +  ---
       2229 +  
       2230 +  ## 12. Getting Started Checklist
       2231 +  
       2232 +  ### Week 1: Setup & Access
       2233 +  
       2234 +  - [ ] **Access to Systems**
       2235 +    - [ ] Get added to `team.auto.tfvars` by manager/team lead
       2236 +    - [ ] Verify PagerDuty account created
       2237 +    - [ ] Verify GitHub org access (linode-obs)
       2238 +    - [ ] Verify Bits team access (ops/sre-o11y)
       2239 +    - [ ] Grafana access (https://grafana.linode.com)
       2240 +    - [ ] ArgoCD access (prod, staging, dev)
       2241 +  
       2242 +  - [ ] **Local Development Setup**
       2243 +    - [ ] Install Homebrew (macOS)
       2244 +    - [ ] Install asdf or mise: `brew install asdf` or `brew install mise`
       2245 +    - [ ] Install direnv: `brew install direnv`
       2246 +    - [ ] Add to shell config: `eval "$(direnv hook zsh)"`
       2247 +    - [ ] Install linode-cli: `brew install linode-cli`
       2248 +    - [ ] Install kubectl: `brew install kubectl`
       2249 +    - [ ] Install terraform: `brew install terraform`
       2250 +    - [ ] Install pre-commit: `brew install pre-commit`
       2251 +    - [ ] Install Hugo: `brew install hugo`
       2252 +  
       2253 +  - [ ] **Repository Setup**
       2254 +    ```bash
       2255 +    mkdir ~/repos
       2256 +    cd ~/repos
       2257 +  
       2258 +    # Clone main repos
       2259 +    git clone bits.linode.com:ops/terraform-observability-team
       2260 +    git clone bits.linode.com:ops/o11y-helm-charts
       2261 +    git clone bits.linode.com:ops/palantir
       2262 +    git clone bits.linode.com:ops/prometheus_rules
       2263 +  
       2264 +    # Setup terraform-observability-team
       2265 +    cd terraform-observability-team
       2266 +    asdf install  # or mise install
       2267 +    direnv allow
       2268 +    pre-commit install --install-hooks
       2269 +  
       2270 +    # Test Terraform
       2271 +    vault login -method=oidc username=$USER
       2272 +    terraform plan
       2273 +    # Should succeed without errors
       2274 +    ```
       2275 +  
       2276 +  - [ ] **Slack Channels**
       2277 +    - [ ] Join #sre-observability (public)
       2278 +    - [ ] Get added to #o11y-core (private)
       2279 +    - [ ] Join #notification-o11y
       2280 +    - [ ] Join #notification-o11y-prs
       2281 +    - [ ] Join #notification-prometheus
       2282 +  
       2283 +  ### Week 2: Learning the Codebase
       2284 +  
       2285 +  - [ ] **Read Documentation**
       2286 +    - [ ] Repository README
       2287 +    - [ ] Handbook: On-call Guide
       2288 +    - [ ] Handbook: Tools
       2289 +    - [ ] Handbook: Git Conventions
       2290 +    - [ ] Browse Services documentation
       2291 +    - [ ] Read 2-3 recent proposals
       2292 +  
       2293 +  - [ ] **Explore Repositories**
       2294 +    - [ ] Browse terraform-observability-team structure
       2295 +    - [ ] Review team.auto.tfvars (understand team structure)
       2296 +    - [ ] Look at o11y-helm-charts (understand app configuration)
       2297 +    - [ ] Explore palantir (see Kubernetes manifests)
       2298 +  
       2299 +  - [ ] **Shadow Team Members**
       2300 +    - [ ] Shadow current on-call for a week
       2301 +    - [ ] Attend team meetings
       2302 +    - [ ] Watch someone do a PR review
       2303 +    - [ ] Watch someone deploy to staging
       2304 +  
       2305 +  ### Week 3: First Tasks
       2306 +  
       2307 +  - [ ] **Make First PR**
       2308 +    - [ ] Fix a typo in documentation
       2309 +    - [ ] Add yourself to a team meeting doc
       2310 +    - [ ] Practice PR → review → merge workflow
       2311 +  
       2312 +  - [ ] **Learn Key Services**
       2313 +    - [ ] Access Grafana, explore dashboards
       2314 +    - [ ] Access ArgoCD, browse applications
       2315 +    - [ ] Query Prometheus/VictoriaMetrics from Grafana
       2316 +    - [ ] Search logs in Loki
       2317 +  
       2318 +  - [ ] **Attend Post-Mortem** (if one occurs)
       2319 +    - [ ] Observe incident response
       2320 +    - [ ] Understand RCA process
       2321 +  
       2322 +  ### Week 4: Increasing Responsibility
       2323 +  
       2324 +  - [ ] **Handle First Alert**
       2325 +    - [ ] Acknowledge warning alert
       2326 +    - [ ] Follow runbook
       2327 +    - [ ] Document resolution
       2328 +  
       2329 +  - [ ] **Review First PR**
       2330 +    - [ ] Review PR in ops/prometheus_rules or similar
       2331 +    - [ ] Provide feedback
       2332 +    - [ ] Approve or request changes
       2333 +  
       2334 +  - [ ] **Complete First Intake Request**
       2335 +    - [ ] Grant Grafana access
       2336 +    - [ ] Or handle Nagios user request
       2337 +  
       2338 +  ### Month 2: On-call Training
       2339 +  
       2340 +  - [ ] **On-call Preparation**
       2341 +    - [ ] Read all alert runbooks
       2342 +    - [ ] Practice silencing alerts with amtool
       2343 +    - [ ] Review escalation procedures
       2344 +    - [ ] Shadow on-call for 2nd week
       2345 +  
       2346 +  - [ ] **Add to On-call Rotation**
       2347 +    - [ ] Team lead adds you to rotation
       2348 +    - [ ] Receive first on-call shift assignment
       2349 +  
       2350 +  - [ ] **First On-call Shift**
       2351 +    - [ ] Create on-call log
       2352 +    - [ ] Handle alerts (with backup support)
       2353 +    - [ ] Complete handoff
       2354 +  
       2355 +  ### Month 3: Full Team Member
       2356 +  
       2357 +  - [ ] **Lead First Project**
       2358 +    - [ ] Small improvement or automation
       2359 +    - [ ] Write proposal if needed
       2360 +    - [ ] Implement and deploy
       2361 +  
       2362 +  - [ ] **Contribute to Documentation**
       2363 +    - [ ] Update outdated docs
       2364 +    - [ ] Add new runbook
       2365 +    - [ ] Write MOP for procedure you learned
       2366 +  
       2367 +  - [ ] **Mentor Next New Hire**
       2368 +    - [ ] Share this guide
       2369 +    - [ ] Answer questions
       2370 +    - [ ] Pair on tasks
       2371 +  
       2372 +  ---
       2373 +  
       2374 +  ## 13. Glossary
       2375 +  
       2376 +  ### Terms & Acronyms
       2377 +  
       2378 +  **ArgoCD**: GitOps continuous deployment tool for Kubernetes
       2379 +  
       2380 +  **Atlantis**: Terraform automation tool that runs plans/applies on PRs
       2381 +  
       2382 +  **Bits**: Akamai's internal GitHub instance (bits.linode.com)
       2383 +  
       2384 +  **CCM**: Cloud Controller Manager - Kubernetes component managing cloud resources (NodeBalancers, 
            + firewalls)
       2385 +  
       2386 +  **Cilium**: Container Network Interface (CNI) providing network policies and encryption
       2387 +  
       2388 +  **DC**: Datacenter (e.g., ewr1 = Newark, iad3 = Ashburn)
       2389 +  
       2390 +  **direnv**: Tool to load environment variables when entering a directory
       2391 +  
       2392 +  **GitOps**: Deployment methodology using Git as source of truth
       2393 +  
       2394 +  **Grafana**: Visualization platform for metrics and logs
       2395 +  
       2396 +  **Hugo**: Static site generator used for team documentation
       2397 +  
       2398 +  **Kustomize**: Tool for customizing Kubernetes YAML files
       2399 +  
       2400 +  **Linode**: Akamai's cloud computing platform (VMs, Kubernetes, networking)
       2401 +  
       2402 +  **Loki**: Log aggregation system (like Prometheus but for logs)
       2403 +  
       2404 +  **LTS**: Long-Term Storage (VictoriaMetrics clusters storing metrics for 13 months)
       2405 +  
       2406 +  **Mise**: Task runner and tool version manager (like Make + asdf)
       2407 +  
       2408 +  **MOP**: Manual Operations Procedure - step-by-step guide for complex tasks
       2409 +  
       2410 +  **mTLS**: Mutual TLS - two-way certificate authentication
       2411 +  
       2412 +  **Nagios**: Legacy monitoring system (being replaced)
       2413 +  
       2414 +  **NIL**: Network Internet Listener - LoadBalancer service exposed externally
       2415 +  
       2416 +  **On-call**: Engineer responsible for responding to alerts during their shift
       2417 +  
       2418 +  **OpenTelemetry (OTel)**: Observability framework for metrics, logs, traces
       2419 +  
       2420 +  **otelgw**: OpenTelemetry Gateway - aggregates telemetry per datacenter
       2421 +  
       2422 +  **Palantir**: Repository containing Kubernetes manifests for ArgoCD
       2423 +  
       2424 +  **PagerDuty**: Incident management and on-call scheduling platform
       2425 +  
       2426 +  **Preflight**: Checks run before an operation (e.g., Cilium upgrade)
       2427 +  
       2428 +  **Prometheus**: Time-series database for metrics collection
       2429 +  
       2430 +  **PromQL**: Prometheus Query Language
       2431 +  
       2432 +  **Runbook**: Documentation for responding to specific alerts
       2433 +  
       2434 +  **Salt**: Configuration management tool (like Ansible/Puppet)
       2435 +  
       2436 +  **Shard**: One instance in a group of divided workload (e.g., Prometheus shards)
       2437 +  
       2438 +  **SRE**: Site Reliability Engineering
       2439 +  
       2440 +  **Terraform**: Infrastructure-as-Code tool
       2441 +  
       2442 +  **Thanos**: Prometheus long-term storage and query federation
       2443 +  
       2444 +  **Vault**: Secret management platform
       2445 +  
       2446 +  **VictoriaMetrics**: Time-series database optimized for Prometheus metrics
       2447 +  
       2448 +  **vminsert**: VictoriaMetrics component for data ingestion
       2449 +  
       2450 +  **vmselect**: VictoriaMetrics component for queries
       2451 +  
       2452 +  **vmstorage**: VictoriaMetrics component for data storage
       2453 +  
       2454 +  ---
       2455 +  
       2456 +  ## Additional Resources
       2457 +  
       2458 +  ### Official Documentation
       2459 +  
       2460 +  - **Team Docs**: https://bits.linode.com/pages/ops/terraform-observability-team/
       2461 +  - **Confluence**: Search "SRE Observability" for cross-team docs
       2462 +  - **Jira**: https://track.akamai.com/jira/projects/OY
       2463 +  
       2464 +  ### External Tools Documentation
       2465 +  
       2466 +  - **Prometheus**: https://prometheus.io/docs/
       2467 +  - **VictoriaMetrics**: https://docs.victoriametrics.com/
       2468 +  - **Grafana**: https://grafana.com/docs/
       2469 +  - **ArgoCD**: https://argo-cd.readthedocs.io/
       2470 +  - **Kubernetes**: https://kubernetes.io/docs/
       2471 +  - **Terraform**: https://www.terraform.io/docs/
       2472 +  - **Loki**: https://grafana.com/docs/loki/
       2473 +  
       2474 +  ### Internal Systems
       2475 +  
       2476 +  - **Grafana Production**: https://grafana.linode.com
       2477 +  - **ArgoCD Production**: https://argocd.infra-o11y-apps.iad3.us.prod.linode.com
       2478 +  - **ArgoCD Staging**: https://argocd.infra-o11y-apps.rin1.us.staging.linode.com
       2479 +  - **PagerDuty Schedule**: https://akamai.pagerduty.com/schedules/PSFD91L
       2480 +  - **Vault**: https://vault.infra.linode.com
       2481 +  
       2482 +  ### Slack Channels
       2483 +  
       2484 +  - **#sre-observability**: Team public channel
       2485 +  - **#o11y-core**: Team private channel
       2486 +  - **#notification-o11y**: Alert notifications
       2487 +  - **#notification-o11y-prs**: PR notifications
       2488 +  - **#notification-prometheus**: Prometheus-specific alerts
       2489 +  - **#sre-salt**: Salt team (for minion keys, etc.)
       2490 +  - **#sre-infrastructure**: Infrastructure team
       2491 +  
       2492 +  ### People to Know
       2493 +  
       2494 +  Check `team.auto.tfvars` for current team roster.
       2495 +  
       2496 +  **Team Lead/Manager**: Ask in #o11y-core
       2497 +  
       2498 +  **On-call**: `@firechief-sre-o11y` in Slack
       2499 +  
       2500 +  ---
       2501 +  
       2502 +  ## Final Thoughts
       2503 +  
       2504 +  Welcome to the SRE Observability team! This guide covers a lot, and **you're not expected to 
            + memorize everything immediately**. Use it as a reference as you work through your first few weeks.
       2505 +  
       2506 +  **Key Takeaways**:
       2507 +  
       2508 +  1. **terraform-observability-team** manages team permissions and documentation
       2509 +  2. **VictoriaMetrics, Prometheus, Grafana, ArgoCD** are our core services
       2510 +  3. **GitOps workflow**: Code → PR → Review → Merge → Deploy
       2511 +  4. **On-call is a rotation**, and you'll be trained before your first shift
       2512 +  5. **Ask questions** - the team is here to help!
       2513 +  
       2514 +  **Learning Path**:
       2515 +  - Week 1: Setup and access
       2516 +  - Week 2: Read and explore
       2517 +  - Week 3: First small tasks
       2518 +  - Week 4: First alerts and reviews
       2519 +  - Month 2: On-call training
       2520 +  - Month 3: Full contributor
       2521 +  
       2522 +  **Remember**: Everyone on the team started where you are now. It takes time to learn all the 
            + systems, and that's okay. Focus on understanding the fundamentals first, and the details will come 
            + with experience.
       2523 +  
       2524 +  **Questions?** Ask in #o11y-core!
       2525 +  
       2526 +  **Good luck, and welcome to the team!**
       2527 +  
       2528 +  ---
       2529 +  
       2530 +  **Document Version**: 1.0
       2531 +  **Created**: 2025-11-20
       2532 +  **Last Updated**: 2025-11-20
       2533 +  **Maintained By**: SRE Observability Team
