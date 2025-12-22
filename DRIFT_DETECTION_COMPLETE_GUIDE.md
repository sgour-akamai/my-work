# Terraform Drift Detection - Complete Implementation Guide

> **Author:** Shivanshu Gour
> **Date:** December 2024
> **Ticket:** OY-2004

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Components](#components)
4. [Part 1: Local Testing Setup](#part-1-local-testing-setup)
5. [Part 2: Production-Like Setup on Linode](#part-2-production-like-setup-on-linode)
6. [Part 3: Demo Steps](#part-3-demo-steps)
7. [All Configuration Files](#all-configuration-files)
8. [Troubleshooting](#troubleshooting)
9. [Production Implementation Notes](#production-implementation-notes)

---

## Overview

### What This Does

Automatically detects when infrastructure has drifted from what's defined in Terraform (i.e., someone made manual changes outside of Terraform), exports metrics to Prometheus, and triggers alerts.

### Why We Need This

- Detect manual changes made to infrastructure
- Alert the team when drift is detected
- Maintain infrastructure-as-code integrity
- Track drift patterns over time

### The Approach

Instead of running Terraform directly from GitHub Actions (which would require Vault credentials), we leverage **Atlantis** which already has access to run Terraform plans. We:

1. Trigger Atlantis via a GitHub PR comment
2. Use a custom Atlantis workflow that runs a drift detection script after the plan
3. The script writes metrics to node_exporter's textfile collector
4. Prometheus scrapes the metrics and fires alerts

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         COMPLETE FLOW                                    │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐
│  GitHub Actions  │  Runs nightly at 2 AM UTC (cron)
│  (Scheduled)     │  Also supports manual workflow_dispatch
└────────┬─────────┘
         │
         │ 1. Creates/updates "drift-detection" branch
         │ 2. Creates/updates draft PR
         │ 3. Comments "atlantis plan -p drift-check"
         ▼
┌──────────────────┐
│  GitHub Webhook  │  Sends PR comment event
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│    Atlantis      │  Receives webhook on :4141/events
│    Server        │
└────────┬─────────┘
         │
         │ Runs custom "drift_detection" workflow:
         │   - terraform init
         │   - terraform plan
         │   - run: drift-detection.sh
         ▼
┌──────────────────┐
│  Drift Detection │  Parses plan output
│  Script          │  Extracts: add/change/destroy counts
└────────┬─────────┘
         │
         │ Writes .prom file to textfile collector directory
         ▼
┌──────────────────┐
│  node_exporter   │  textfile collector reads .prom files
│  (:9100)         │  Exposes metrics on /metrics endpoint
└────────┬─────────┘
         │
         │ Prometheus scrapes every 15s
         ▼
┌──────────────────┐
│  Prometheus      │  Stores metrics in TSDB
│  (:9090)         │  Evaluates alert rules
└────────┬─────────┘
         │
         │ If linode_atlantis_drift_total > 0
         ▼
┌──────────────────┐
│  Alert Manager   │  Routes alert to Slack
│  → Slack         │  #notification-o11y
└──────────────────┘
```

---

## Components

### 1. Drift Detection Script (`drift-detection.sh`)

**Purpose:** Parses Terraform plan output and writes Prometheus metrics

**Location:** `/opt/atlantis/scripts/drift-detection.sh` (on Atlantis server)

**What it does:**
- Reads the binary `.tfplan` file using `terraform show`
- Extracts the "Plan: X to add, Y to change, Z to destroy" line
- Writes metrics to `/var/lib/prometheus-node-exporter/atlantis_drift_*.prom`

### 2. Atlantis Server Config (`repos.yaml`)

**Purpose:** Defines the custom `drift_detection` workflow

**Location:** `/etc/atlantis/repos.yaml` (on Atlantis server)

### 3. Repo Config (`atlantis.yaml`)

**Purpose:** Defines the `drift-check` project that uses the custom workflow

**Location:** Root of the Terraform repo

### 4. GitHub Actions Workflow (`drift-detection.yml`)

**Purpose:** Triggers drift detection on a schedule

**Location:** `.github/workflows/drift-detection.yml` in the repo

### 5. Prometheus Alert Rule (`alert_rules.yml`)

**Purpose:** Fires alert when drift is detected

**Location:** `/etc/prometheus/alert_rules.yml` (on Prometheus server)

---

## Part 1: Local Testing Setup

This section covers testing the drift detection script locally on your Mac before deploying to a server.

### Prerequisites

- macOS with Homebrew
- Terraform installed
- Git access to the repo

### Step 1.1: Create Directory Structure

```bash
mkdir -p ~/drift-detection-test/{prometheus,node_exporter,atlantis,scripts,textfile_collector,sample-terraform,.github/workflows}
```

**Directory purposes:**
| Directory | Purpose |
|-----------|---------|
| `prometheus/` | Prometheus config and data |
| `node_exporter/` | node_exporter configs |
| `atlantis/` | Atlantis server config |
| `scripts/` | Drift detection script |
| `textfile_collector/` | Where .prom metric files get written |
| `sample-terraform/` | Simple TF config for testing |
| `.github/workflows/` | GitHub Actions workflow |

### Step 1.2: Install Required Tools

```bash
brew install prometheus node_exporter atlantis
```

Verify installations:
```bash
prometheus --version
node_exporter --version
atlantis version
```

### Step 1.3: Create the Drift Detection Script

```bash
cat > ~/drift-detection-test/scripts/drift-detection.sh << 'EOF'
#!/bin/bash
set -euo pipefail

TEXTFILE_DIR="${TEXTFILE_DIR:-/var/lib/prometheus-node-exporter}"
REPO_NAME="${REPO_NAME:-unknown}"
PROJECT_NAME="${PROJECT_NAME:-default}"
PLANFILE="${PLANFILE:-}"
DIR="${DIR:-.}"

METRICS_FILE="${TEXTFILE_DIR}/atlantis_drift_${PROJECT_NAME}.prom"

DRIFT_DETECTED=0
RESOURCES_TO_ADD=0
RESOURCES_TO_CHANGE=0
RESOURCES_TO_DESTROY=0

echo "=== Drift Detection Script ==="
echo "PLANFILE: ${PLANFILE:-not set}"
echo "REPO_NAME: $REPO_NAME"
echo "PROJECT_NAME: $PROJECT_NAME"
echo "DIR: $DIR"

# For local testing, use test directory
if [[ "$TEXTFILE_DIR" == "/var/lib/prometheus-node-exporter" ]] && [[ -d "$HOME/drift-detection-test/textfile_collector" ]]; then
    TEXTFILE_DIR="$HOME/drift-detection-test/textfile_collector"
    METRICS_FILE="${TEXTFILE_DIR}/atlantis_drift_${PROJECT_NAME}.prom"
fi

if [[ -n "$PLANFILE" ]] && [[ -f "$PLANFILE" ]]; then
    echo "Reading plan file..."

    # Check if it's a binary plan file (from Atlantis) or text file (local test)
    if file "$PLANFILE" | grep -q "text"; then
        PLAN_TEXT=$(cat "$PLANFILE")
    else
        # Binary plan file - use terraform show
        cd "$DIR"
        PLAN_TEXT=$(terraform show -no-color "$PLANFILE" 2>/dev/null || cat "$PLANFILE" 2>/dev/null || echo "")
    fi

    echo "Plan output length: ${#PLAN_TEXT} chars"

    if echo "$PLAN_TEXT" | grep -q "No changes"; then
        echo "Result: No changes detected"
        DRIFT_DETECTED=0
    elif echo "$PLAN_TEXT" | grep -q "0 to add, 0 to change, 0 to destroy"; then
        echo "Result: No changes detected"
        DRIFT_DETECTED=0
    else
        PLAN_LINE=$(echo "$PLAN_TEXT" | grep -E "Plan: [0-9]+ to add" | head -1 || true)

        if [[ -n "$PLAN_LINE" ]]; then
            echo "Found: $PLAN_LINE"
            RESOURCES_TO_ADD=$(echo "$PLAN_LINE" | grep -oE '[0-9]+ to add' | grep -oE '[0-9]+' || echo "0")
            RESOURCES_TO_CHANGE=$(echo "$PLAN_LINE" | grep -oE '[0-9]+ to change' | grep -oE '[0-9]+' || echo "0")
            RESOURCES_TO_DESTROY=$(echo "$PLAN_LINE" | grep -oE '[0-9]+ to destroy' | grep -oE '[0-9]+' || echo "0")

            TOTAL=$((RESOURCES_TO_ADD + RESOURCES_TO_CHANGE + RESOURCES_TO_DESTROY))
            if [[ $TOTAL -gt 0 ]]; then
                DRIFT_DETECTED=1
                echo "Result: DRIFT DETECTED"
            fi
        else
            if echo "$PLAN_TEXT" | grep -qE "will be (created|destroyed|updated|replaced)"; then
                DRIFT_DETECTED=1
                RESOURCES_TO_ADD=$(echo "$PLAN_TEXT" | grep -c "will be created" || echo "0")
                RESOURCES_TO_CHANGE=$(echo "$PLAN_TEXT" | grep -c "will be updated" || echo "0")
                RESOURCES_TO_DESTROY=$(echo "$PLAN_TEXT" | grep -c "will be destroyed" || echo "0")
                echo "Result: DRIFT DETECTED (counted from resource lines)"
            fi
        fi
    fi
else
    echo "Warning: PLANFILE not set or not found"
fi

echo ""
echo "Drift: $DRIFT_DETECTED | Add: $RESOURCES_TO_ADD | Change: $RESOURCES_TO_CHANGE | Destroy: $RESOURCES_TO_DESTROY"

cat > "$METRICS_FILE" << METRICS
# HELP linode_atlantis_drift_total Whether drift was detected (1=drift, 0=no drift)
# TYPE linode_atlantis_drift_total gauge
linode_atlantis_drift_total{repo="${REPO_NAME}",project="${PROJECT_NAME}"} ${DRIFT_DETECTED}
# HELP linode_atlantis_resources_drift_added_total Resources to be added
# TYPE linode_atlantis_resources_drift_added_total gauge
linode_atlantis_resources_drift_added_total{repo="${REPO_NAME}",project="${PROJECT_NAME}"} ${RESOURCES_TO_ADD}
# HELP linode_atlantis_resources_drift_changed_total Resources to be changed
# TYPE linode_atlantis_resources_drift_changed_total gauge
linode_atlantis_resources_drift_changed_total{repo="${REPO_NAME}",project="${PROJECT_NAME}"} ${RESOURCES_TO_CHANGE}
# HELP linode_atlantis_resources_drift_destroyed_total Resources to be destroyed
# TYPE linode_atlantis_resources_drift_destroyed_total gauge
linode_atlantis_resources_drift_destroyed_total{repo="${REPO_NAME}",project="${PROJECT_NAME}"} ${RESOURCES_TO_DESTROY}
METRICS

echo "Metrics written to $METRICS_FILE"
EOF

chmod +x ~/drift-detection-test/scripts/drift-detection.sh
```

### Step 1.4: Create Sample Terraform Config

```bash
cat > ~/drift-detection-test/sample-terraform/main.tf << 'EOF'
terraform {
  required_version = ">= 1.0.0"

  backend "local" {
    path = "terraform.tfstate"
  }

  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
    null = {
      source  = "hashicorp/null"
      version = "~> 3.0"
    }
  }
}

resource "local_file" "config" {
  content  = var.config_content
  filename = "${path.module}/output/config.txt"
}

resource "null_resource" "example" {
  triggers = {
    version = var.app_version
  }
}

variable "config_content" {
  description = "Content for the config file"
  type        = string
  default     = "Hello from Terraform!"
}

variable "app_version" {
  description = "Application version"
  type        = string
  default     = "1.0.0"
}

output "config_file_path" {
  value = local_file.config.filename
}
EOF

mkdir -p ~/drift-detection-test/sample-terraform/output
```

### Step 1.5: Initialize and Apply Terraform

```bash
cd ~/drift-detection-test/sample-terraform
terraform init
terraform apply -auto-approve
```

### Step 1.6: Test the Drift Detection Script

**Test with no drift:**
```bash
cd ~/drift-detection-test
terraform -chdir=sample-terraform plan -no-color > /tmp/plan-output.txt 2>&1
PLANFILE="/tmp/plan-output.txt" REPO_NAME="test-repo" PROJECT_NAME="test" ./scripts/drift-detection.sh
cat ~/drift-detection-test/textfile_collector/atlantis_drift_test.prom
```

**Simulate drift and test again:**
```bash
# Modify the file outside of Terraform (simulate drift)
echo "Manual change - DRIFT!" > ~/drift-detection-test/sample-terraform/output/config.txt

# Run plan
terraform -chdir=~/drift-detection-test/sample-terraform plan -no-color > /tmp/plan-output.txt 2>&1

# Run drift detection
cd ~/drift-detection-test
PLANFILE="/tmp/plan-output.txt" REPO_NAME="test-repo" PROJECT_NAME="test" ./scripts/drift-detection.sh

# Check metrics - should show drift_total = 1
cat ~/drift-detection-test/textfile_collector/atlantis_drift_test.prom
```

### Step 1.7: Start node_exporter (Terminal 1)

```bash
node_exporter --collector.textfile.directory="$HOME/drift-detection-test/textfile_collector" --web.listen-address=":9100"
```

Keep this running.

### Step 1.8: Create Prometheus Config

```bash
cat > ~/drift-detection-test/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
EOF

cat > ~/drift-detection-test/prometheus/alert_rules.yml << 'EOF'
groups:
  - name: terraform_drift
    rules:
      - alert: TerraformDriftDetected
        expr: linode_atlantis_drift_total > 0
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Terraform drift detected in {{ $labels.repo }}"
          description: "Drift in repo={{ $labels.repo }}, project={{ $labels.project }}"
EOF
```

### Step 1.9: Start Prometheus (Terminal 2)

```bash
prometheus --config.file="$HOME/drift-detection-test/prometheus/prometheus.yml" --storage.tsdb.path="$HOME/drift-detection-test/prometheus/data" --web.listen-address=":9090"
```

Keep this running.

### Step 1.10: Verify Local Setup

```bash
# Check node_exporter metrics
curl -s http://localhost:9100/metrics | grep linode_atlantis

# Check Prometheus targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'

# Check metrics in Prometheus
curl -s 'http://localhost:9090/api/v1/query?query=linode_atlantis_drift_total' | jq '.data.result'

# Check alerts
curl -s http://localhost:9090/api/v1/alerts | jq '.data.alerts'
```

**View in browser:**
- Prometheus UI: http://localhost:9090
- Alerts: http://localhost:9090/alerts
- Metrics query: http://localhost:9090/graph → query `linode_atlantis_drift_total`

---

## Part 2: Production-Like Setup on Linode

This section covers setting up a complete Atlantis + Prometheus + drift detection environment on a Linode instance.

### Prerequisites

- Personal Linode account
- Personal GitHub account
- SSH key configured

### Step 2.1: Create GitHub Test Repo

1. Go to https://github.com/new
2. Create repo named `terraform-drift-test`
3. Make it private
4. Initialize with README

Clone and set up:
```bash
cd ~
git clone https://github.com/YOUR_USERNAME/terraform-drift-test.git
cd terraform-drift-test

# Copy Terraform config
cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.0.0"

  backend "local" {
    path = "terraform.tfstate"
  }

  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
    null = {
      source  = "hashicorp/null"
      version = "~> 3.0"
    }
  }
}

resource "local_file" "config" {
  content  = var.config_content
  filename = "${path.module}/output/config.txt"
}

resource "null_resource" "example" {
  triggers = {
    version = var.app_version
  }
}

variable "config_content" {
  type    = string
  default = "Hello from Terraform!"
}

variable "app_version" {
  type    = string
  default = "1.0.0"
}

output "config_file_path" {
  value = local_file.config.filename
}
EOF

# Create atlantis.yaml
cat > atlantis.yaml << 'EOF'
version: 3
projects:
  - dir: .
    name: main
    workflow: default
    autoplan:
      enabled: true
      when_modified: ["*.tf", "*.tfvars"]

  - dir: .
    name: drift-check
    workflow: drift_detection
    autoplan:
      enabled: false
EOF

# Create .gitignore
cat > .gitignore << 'EOF'
.terraform/
*.tfstate
*.tfstate.*
.terraform.lock.hcl
output/
.drift-check-timestamp
EOF

# Create GitHub Actions workflow
mkdir -p .github/workflows
cat > .github/workflows/drift-detection.yml << 'EOF'
name: Terraform Drift Detection

on:
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:
    inputs:
      project:
        description: 'Atlantis project to check'
        required: false
        default: 'drift-check'

jobs:
  trigger-drift-check:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Get default branch
        id: default-branch
        run: |
          DEFAULT_BRANCH=$(gh api repos/${{ github.repository }} --jq '.default_branch')
          echo "branch=$DEFAULT_BRANCH" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Find or create drift-detection PR
        id: find-pr
        run: |
          PR_NUMBER=$(gh pr list --state open --head drift-detection --json number --jq '.[0].number // empty')
          if [ -z "$PR_NUMBER" ]; then
            echo "pr_exists=false" >> $GITHUB_OUTPUT
          else
            echo "pr_exists=true" >> $GITHUB_OUTPUT
            echo "pr_number=$PR_NUMBER" >> $GITHUB_OUTPUT
          fi
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Create drift-detection branch and PR
        if: steps.find-pr.outputs.pr_exists == 'false'
        id: create-pr
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git checkout -B drift-detection
          echo "Last drift check: $(date -u '+%Y-%m-%d %H:%M:%S UTC')" > .drift-check-timestamp
          git add .drift-check-timestamp
          git commit -m "chore: trigger drift detection check" || echo "No changes"
          git push -f origin drift-detection
          PR_URL=$(gh pr create \
            --title "[Automated] Terraform Drift Detection" \
            --body "Automated PR for drift detection. Do not merge." \
            --base ${{ steps.default-branch.outputs.branch }} \
            --head drift-detection \
            --draft)
          PR_NUMBER=$(echo "$PR_URL" | grep -oE '[0-9]+$')
          echo "pr_number=$PR_NUMBER" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Update existing PR
        if: steps.find-pr.outputs.pr_exists == 'true'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git fetch origin drift-detection || true
          git checkout drift-detection || git checkout -b drift-detection
          echo "Last drift check: $(date -u '+%Y-%m-%d %H:%M:%S UTC')" > .drift-check-timestamp
          git add .drift-check-timestamp
          git commit -m "chore: trigger drift detection check" || echo "No changes"
          git push origin drift-detection
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Trigger Atlantis
        run: |
          PR_NUMBER="${{ steps.find-pr.outputs.pr_number || steps.create-pr.outputs.pr_number }}"
          gh pr comment "$PR_NUMBER" --body "atlantis plan -p ${{ github.event.inputs.project || 'drift-check' }}"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
EOF

# Commit and push
git add .
git commit -m "Initial setup with drift detection"
git push origin main
```

### Step 2.2: Create Linode Instance

**Via Linode CLI:**
```bash
linode-cli linodes create \
  --label atlantis-drift-test \
  --region us-ord \
  --type g6-nanode-1 \
  --image linode/ubuntu24.04 \
  --root_pass "YOUR_SECURE_PASSWORD" \
  --authorized_keys "$(cat ~/.ssh/id_rsa.pub)"
```

**Or via Cloud Manager:**
1. https://cloud.linode.com/linodes/create
2. Ubuntu 24.04 LTS
3. Nanode 1GB ($5/month)
4. Add SSH key
5. Create

**Get the IP:**
```bash
linode-cli linodes list --label atlantis-drift-test
```

**SSH in:**
```bash
ssh root@YOUR_LINODE_IP
```

### Step 2.3: Install Software on Linode

Run these on the Linode:

```bash
# Update system
apt update && apt upgrade -y
apt install -y wget curl unzip jq git

# Install Terraform
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/hashicorp.list
apt update && apt install -y terraform

# Install Atlantis
cd /tmp
wget https://github.com/runatlantis/atlantis/releases/download/v0.28.5/atlantis_linux_amd64.zip
unzip atlantis_linux_amd64.zip
mv atlantis /usr/local/bin/
chmod +x /usr/local/bin/atlantis

# Install node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xzf node_exporter-1.8.2.linux-amd64.tar.gz
mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/

# Install Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.54.1/prometheus-2.54.1.linux-amd64.tar.gz
tar xzf prometheus-2.54.1.linux-amd64.tar.gz
mv prometheus-2.54.1.linux-amd64/prometheus /usr/local/bin/
mv prometheus-2.54.1.linux-amd64/promtool /usr/local/bin/

# Create directories
mkdir -p /etc/atlantis
mkdir -p /var/lib/prometheus-node-exporter
mkdir -p /etc/prometheus
mkdir -p /var/lib/prometheus
mkdir -p /opt/atlantis/scripts

# Verify
terraform version
atlantis version
node_exporter --version
prometheus --version
```

### Step 2.4: Create Drift Detection Script on Linode

```bash
cat > /opt/atlantis/scripts/drift-detection.sh << 'EOF'
#!/bin/bash
set -euo pipefail

TEXTFILE_DIR="${TEXTFILE_DIR:-/var/lib/prometheus-node-exporter}"
REPO_NAME="${REPO_NAME:-unknown}"
PROJECT_NAME="${PROJECT_NAME:-default}"
PLANFILE="${PLANFILE:-}"
DIR="${DIR:-.}"

METRICS_FILE="${TEXTFILE_DIR}/atlantis_drift_${PROJECT_NAME}.prom"

DRIFT_DETECTED=0
RESOURCES_TO_ADD=0
RESOURCES_TO_CHANGE=0
RESOURCES_TO_DESTROY=0

echo "=== Drift Detection Script ==="
echo "PLANFILE: ${PLANFILE:-not set}"
echo "REPO_NAME: $REPO_NAME"
echo "PROJECT_NAME: $PROJECT_NAME"
echo "DIR: $DIR"

if [[ -n "$PLANFILE" ]] && [[ -f "$PLANFILE" ]]; then
    echo "Converting binary plan to text..."

    cd "$DIR"
    PLAN_TEXT=$(terraform show -no-color "$PLANFILE" 2>/dev/null || cat "$PLANFILE" 2>/dev/null || echo "")

    echo "Plan output length: ${#PLAN_TEXT} chars"

    if echo "$PLAN_TEXT" | grep -q "No changes"; then
        echo "Result: No changes detected"
        DRIFT_DETECTED=0
    elif echo "$PLAN_TEXT" | grep -q "0 to add, 0 to change, 0 to destroy"; then
        echo "Result: No changes detected"
        DRIFT_DETECTED=0
    else
        PLAN_LINE=$(echo "$PLAN_TEXT" | grep -E "Plan: [0-9]+ to add" | head -1 || true)

        if [[ -n "$PLAN_LINE" ]]; then
            echo "Found: $PLAN_LINE"
            RESOURCES_TO_ADD=$(echo "$PLAN_LINE" | grep -oE '[0-9]+ to add' | grep -oE '[0-9]+' || echo "0")
            RESOURCES_TO_CHANGE=$(echo "$PLAN_LINE" | grep -oE '[0-9]+ to change' | grep -oE '[0-9]+' || echo "0")
            RESOURCES_TO_DESTROY=$(echo "$PLAN_LINE" | grep -oE '[0-9]+ to destroy' | grep -oE '[0-9]+' || echo "0")

            TOTAL=$((RESOURCES_TO_ADD + RESOURCES_TO_CHANGE + RESOURCES_TO_DESTROY))
            if [[ $TOTAL -gt 0 ]]; then
                DRIFT_DETECTED=1
                echo "Result: DRIFT DETECTED"
            fi
        else
            if echo "$PLAN_TEXT" | grep -qE "will be (created|destroyed|updated|replaced)"; then
                DRIFT_DETECTED=1
                RESOURCES_TO_ADD=$(echo "$PLAN_TEXT" | grep -c "will be created" || echo "0")
                RESOURCES_TO_CHANGE=$(echo "$PLAN_TEXT" | grep -c "will be updated" || echo "0")
                RESOURCES_TO_DESTROY=$(echo "$PLAN_TEXT" | grep -c "will be destroyed" || echo "0")
                echo "Result: DRIFT DETECTED (counted from resource lines)"
            fi
        fi
    fi
else
    echo "Warning: PLANFILE not set or not found"
fi

echo ""
echo "Drift: $DRIFT_DETECTED | Add: $RESOURCES_TO_ADD | Change: $RESOURCES_TO_CHANGE | Destroy: $RESOURCES_TO_DESTROY"

cat > "$METRICS_FILE" << METRICS
# HELP linode_atlantis_drift_total Whether drift was detected (1=drift, 0=no drift)
# TYPE linode_atlantis_drift_total gauge
linode_atlantis_drift_total{repo="${REPO_NAME}",project="${PROJECT_NAME}"} ${DRIFT_DETECTED}
# HELP linode_atlantis_resources_drift_added_total Resources to be added
# TYPE linode_atlantis_resources_drift_added_total gauge
linode_atlantis_resources_drift_added_total{repo="${REPO_NAME}",project="${PROJECT_NAME}"} ${RESOURCES_TO_ADD}
# HELP linode_atlantis_resources_drift_changed_total Resources to be changed
# TYPE linode_atlantis_resources_drift_changed_total gauge
linode_atlantis_resources_drift_changed_total{repo="${REPO_NAME}",project="${PROJECT_NAME}"} ${RESOURCES_TO_CHANGE}
# HELP linode_atlantis_resources_drift_destroyed_total Resources to be destroyed
# TYPE linode_atlantis_resources_drift_destroyed_total gauge
linode_atlantis_resources_drift_destroyed_total{repo="${REPO_NAME}",project="${PROJECT_NAME}"} ${RESOURCES_TO_DESTROY}
METRICS

echo "Metrics written to $METRICS_FILE"
EOF

chmod +x /opt/atlantis/scripts/drift-detection.sh
```

### Step 2.5: Create Systemd Services

**node_exporter:**
```bash
cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/node_exporter --collector.textfile.directory=/var/lib/prometheus-node-exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
```

**Prometheus:**
```bash
cat > /etc/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
EOF

cat > /etc/prometheus/alert_rules.yml << 'EOF'
groups:
  - name: terraform_drift
    rules:
      - alert: TerraformDriftDetected
        expr: linode_atlantis_drift_total > 0
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Terraform drift detected in {{ $labels.repo }}"
          description: "Drift in repo={{ $labels.repo }}, project={{ $labels.project }}"
EOF

cat > /etc/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=Prometheus
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
```

### Step 2.6: Create GitHub Credentials

**1. Create GitHub Personal Access Token:**
- Go to: https://github.com/settings/tokens?type=beta
- Generate new token with:
  - Repository access: Only select `terraform-drift-test`
  - Permissions: Contents (R/W), Issues (R/W), Pull requests (R/W), Webhooks (R/W)
- Save the token (starts with `github_pat_...`)

**2. Generate webhook secret:**
```bash
openssl rand -hex 20
# Save this output
```

### Step 2.7: Configure Atlantis

```bash
# Create repos.yaml
cat > /etc/atlantis/repos.yaml << 'EOF'
repos:
  - id: github.com/YOUR_GITHUB_USERNAME/terraform-drift-test
    allowed_overrides: [workflow, apply_requirements]
    allow_custom_workflows: true

workflows:
  drift_detection:
    plan:
      steps:
        - init
        - plan
        - run: |
            export TEXTFILE_DIR="/var/lib/prometheus-node-exporter"
            export REPO_NAME="$BASE_REPO_OWNER/$BASE_REPO_NAME"
            export PROJECT_NAME="${PROJECT_NAME:-drift-check}"
            export DIR="$REPO_REL_DIR"
            /opt/atlantis/scripts/drift-detection.sh
EOF

# Get your IP
MY_IP=$(curl -4 -s ifconfig.me)
echo "Your IP: $MY_IP"

# Create Atlantis service (replace YOUR_* values)
cat > /etc/systemd/system/atlantis.service << EOF
[Unit]
Description=Atlantis
After=network.target

[Service]
Type=simple
Environment="ATLANTIS_GH_USER=YOUR_GITHUB_USERNAME"
Environment="ATLANTIS_GH_TOKEN=YOUR_GITHUB_TOKEN"
Environment="ATLANTIS_GH_WEBHOOK_SECRET=YOUR_WEBHOOK_SECRET"
Environment="ATLANTIS_REPO_ALLOWLIST=github.com/YOUR_GITHUB_USERNAME/terraform-drift-test"
ExecStart=/usr/local/bin/atlantis server \
    --atlantis-url=http://${MY_IP}:4141 \
    --repo-config=/etc/atlantis/repos.yaml \
    --port=4141
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable atlantis
systemctl start atlantis
```

### Step 2.8: Configure GitHub Webhook

1. Go to: `https://github.com/YOUR_USERNAME/terraform-drift-test/settings/hooks`
2. Click "Add webhook"
3. Settings:
   - Payload URL: `http://YOUR_LINODE_IP:4141/events`
   - Content type: `application/json`
   - Secret: Your webhook secret from Step 2.6
   - Events: Issue comments, Pull requests, Pull request reviews, Pushes
4. Click "Add webhook"

### Step 2.9: Initialize Terraform State

```bash
cd /tmp
git clone https://YOUR_USERNAME:YOUR_TOKEN@github.com/YOUR_USERNAME/terraform-drift-test.git
cd terraform-drift-test
terraform init
terraform apply -auto-approve
```

### Step 2.10: Verify Everything

```bash
# Check all services
systemctl status node_exporter --no-pager
systemctl status prometheus --no-pager
systemctl status atlantis --no-pager

# Check Atlantis health
curl -s http://localhost:4141/healthz

# Check Prometheus targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

---

## Part 3: Demo Steps

Use these steps to demonstrate the drift detection system to your team.

### Quick Demo (5 minutes)

**Prerequisites:** Linode is running with all services up.

**1. Verify services are running:**
```bash
ssh root@YOUR_LINODE_IP
systemctl status atlantis prometheus node_exporter
```

**2. Create a test PR (on your Mac):**
```bash
cd ~/terraform-drift-test
git checkout -b demo-drift-$(date +%s)
echo "# Demo - $(date)" >> main.tf
git add . && git commit -m "Demo drift detection"
git push origin HEAD
```

**3. Create PR on GitHub**
- Go to the repo → Pull requests → New pull request
- Select your branch → Create pull request

**4. Watch Atlantis run (on Linode):**
```bash
journalctl -u atlantis -f
```

**5. Trigger drift detection:**
- On the PR, comment: `atlantis plan -p drift-check`

**6. Show the metrics:**
```bash
cat /var/lib/prometheus-node-exporter/atlantis_drift_drift-check.prom
curl -s 'http://localhost:9090/api/v1/query?query=linode_atlantis_drift_total' | jq '.data.result'
```

**7. Show Prometheus UI:**
- Open: `http://YOUR_LINODE_IP:9090/alerts`

### Starting From Scratch (if Linode was stopped)

**1. Start the Linode** (via Cloud Manager or CLI)

**2. SSH in and start services:**
```bash
ssh root@YOUR_LINODE_IP
systemctl start node_exporter prometheus atlantis
systemctl status node_exporter prometheus atlantis
```

**3. Verify webhook is still configured:**
- Check: `https://github.com/YOUR_USERNAME/terraform-drift-test/settings/hooks`
- Redeliver the ping if needed

**4. Continue with the Quick Demo steps above**

---

## All Configuration Files

### Drift Detection Script
**Location:** `/opt/atlantis/scripts/drift-detection.sh`

See Step 2.4 for full content.

### Atlantis repos.yaml
**Location:** `/etc/atlantis/repos.yaml`

```yaml
repos:
  - id: github.com/YOUR_USERNAME/terraform-drift-test
    allowed_overrides: [workflow, apply_requirements]
    allow_custom_workflows: true

workflows:
  drift_detection:
    plan:
      steps:
        - init
        - plan
        - run: |
            export TEXTFILE_DIR="/var/lib/prometheus-node-exporter"
            export REPO_NAME="$BASE_REPO_OWNER/$BASE_REPO_NAME"
            export PROJECT_NAME="${PROJECT_NAME:-drift-check}"
            export DIR="$REPO_REL_DIR"
            /opt/atlantis/scripts/drift-detection.sh
```

### Repo atlantis.yaml
**Location:** Root of terraform repo

```yaml
version: 3
projects:
  - dir: .
    name: main
    workflow: default
    autoplan:
      enabled: true
      when_modified: ["*.tf", "*.tfvars"]

  - dir: .
    name: drift-check
    workflow: drift_detection
    autoplan:
      enabled: false
```

### Prometheus Config
**Location:** `/etc/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

### Alert Rules
**Location:** `/etc/prometheus/alert_rules.yml`

```yaml
groups:
  - name: terraform_drift
    rules:
      - alert: TerraformDriftDetected
        expr: linode_atlantis_drift_total > 0
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Terraform drift detected in {{ $labels.repo }}"
          description: "Drift in repo={{ $labels.repo }}, project={{ $labels.project }}"
```

---

## Troubleshooting

### Atlantis not responding to webhooks

**Check webhook delivery:**
- Go to GitHub repo → Settings → Webhooks → Recent Deliveries
- Look for red X marks indicating failures

**Check Atlantis logs:**
```bash
journalctl -u atlantis -f
```

**Common issues:**
- Firewall blocking port 4141: `ufw allow 4141/tcp`
- Wrong webhook secret: Verify it matches in GitHub and Atlantis service
- Wrong webhook URL: Should be `http://IP:4141/events`

### Metrics not appearing in Prometheus

**Check node_exporter:**
```bash
curl -s http://localhost:9100/metrics | grep linode_atlantis
```

**Check the .prom file exists:**
```bash
ls -la /var/lib/prometheus-node-exporter/
cat /var/lib/prometheus-node-exporter/atlantis_drift_*.prom
```

**Check Prometheus targets:**
```bash
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets'
```

### Drift detection script not detecting drift

**Test manually:**
```bash
cd /tmp/terraform-drift-test
terraform plan -out=test.tfplan
PLANFILE="test.tfplan" REPO_NAME="test" PROJECT_NAME="test" /opt/atlantis/scripts/drift-detection.sh
```

**Check Atlantis is passing correct env vars:**
- Look in Atlantis logs for the script output
- Verify `$PLANFILE` and `$REPO_REL_DIR` are set

### Services not starting after reboot

**Enable services:**
```bash
systemctl enable node_exporter prometheus atlantis
```

**Check status:**
```bash
systemctl status node_exporter prometheus atlantis
```

---

## Production Implementation Notes

To implement this for `terraform-observability-team` in production:

### 1. atlantis-formula (Salt)

Add the drift detection script to the Atlantis formula:
- File: `atlantis-formula/atlantis/files/drift-detection.sh`
- Deploy to: `/opt/atlantis/scripts/drift-detection.sh`

### 2. Atlantis Server Config

Update the server's repos.yaml to include the `drift_detection` workflow.

### 3. t-o-t Repo

The atlantis.yaml already has a partial `drift_detection` workflow. Update it to:
- Call the drift detection script
- Ensure the `drift-check` project exists

### 4. Prometheus

Add alert rule to the existing Prometheus config:
```yaml
- alert: TerraformDriftDetected
  expr: linode_atlantis_drift_total > 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Terraform drift detected in {{ $labels.repo }}"
```

### 5. Alertmanager

Route the alert to `#notification-o11y` Slack channel.

---

## Quick Reference Commands

```bash
# Start all services
systemctl start node_exporter prometheus atlantis

# Check all services
systemctl status node_exporter prometheus atlantis

# View Atlantis logs
journalctl -u atlantis -f

# Check metrics
curl -s http://localhost:9100/metrics | grep linode_atlantis

# Query Prometheus
curl -s 'http://localhost:9090/api/v1/query?query=linode_atlantis_drift_total' | jq

# Check alerts
curl -s http://localhost:9090/api/v1/alerts | jq

# Trigger drift check on PR
# Comment on PR: atlantis plan -p drift-check
```

---

**Document Last Updated:** December 2024
