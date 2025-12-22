# Drift Detection - Production Implementation Changes

> **Ticket:** OY-2004
> **Author:** Shivanshu Gour
> **Date:** December 2024

---

## Executive Summary

This document details **all changes required** to implement drift detection for `terraform-observability-team` (and other TF repos). The good news is that **most infrastructure already exists** - we just need to connect the pieces.

### What Already Exists

| Component | Location | Status |
|-----------|----------|--------|
| Drift detection script | `atlantis-formula/atlantis/files/export-drift-metric.sh` | Exists, needs update |
| Salt state to deploy script | `atlantis-formula/atlantis/install.sls` | Exists, working |
| Metrics directory creation | `atlantis-formula/atlantis/install.sls` | Exists, working |
| `drift-check` project | `t-o-t/atlantis.yaml` | Exists, working |
| `drift_detection` workflow | `t-o-t/atlantis.yaml` | Exists, needs update |
| Prometheus scraping Atlantis | Already configured | Working |

### What Needs to Change

| Change | Repo | File | Priority |
|--------|------|------|----------|
| Fix script to handle binary .tfplan | atlantis-formula | `files/export-drift-metric.sh` | HIGH |
| Update workflow to call script | t-o-t | `atlantis.yaml` | HIGH |
| Add scheduled GitHub Action | t-o-t | `.github/workflows/drift-detection.yml` | HIGH |
| Add Prometheus alert rule | Prometheus config | Alert rules | MEDIUM |

---

## Change #1: Update Drift Detection Script

**Repo:** `atlantis-formula`
**File:** `atlantis/files/export-drift-metric.sh`
**Issue:** Script reads PLANFILE directly with `cat`, but Atlantis provides binary `.tfplan` files
**Fix:** Use `terraform show` to convert binary plan to text

### Current Code (Lines 52-54):

```bash
log "Analyzing plan file: ${PLANFILE}"
PLAN_OUTPUT="$(cat "${PLANFILE}")"
```

### Updated Code:

```bash
log "Analyzing plan file: ${PLANFILE}"

# Check if PLANFILE is binary (Atlantis .tfplan) or text
if file "${PLANFILE}" 2>/dev/null | grep -q "text"; then
  # Text file - read directly
  PLAN_OUTPUT="$(cat "${PLANFILE}")"
else
  # Binary .tfplan file - use terraform show to convert
  log "Converting binary plan file to text..."
  if command -v terraform &> /dev/null; then
    PLAN_OUTPUT="$(terraform show -no-color "${PLANFILE}" 2>/dev/null || cat "${PLANFILE}" 2>/dev/null || echo "")"
  else
    log "Warning: terraform not found, attempting to read file directly"
    PLAN_OUTPUT="$(cat "${PLANFILE}" 2>/dev/null || echo "")"
  fi
fi
```

### Full Updated Script:

Replace the entire content of `atlantis/files/export-drift-metric.sh` with:

```bash
#!/bin/bash
# Atlantis Drift Detection Metric Exporter
# Exports Prometheus metrics based on Terraform plan output

set -e

# =====================
# Configuration
# =====================
REPO_NAME="${BASE_REPO_NAME:-unknown}"
REPO_OWNER="${BASE_REPO_OWNER:-unknown}"
WORKSPACE="${WORKSPACE:-default}"
DIRECTORY="${REPO_REL_DIR:-.}"
PROJECT_NAME="${PROJECT_NAME:-unknown}"

# Metric configuration
METRIC_DIR="${METRIC_DIR:-/var/lib/prometheus-node-exporter}"
METRIC_FILE_TMP="${METRIC_DIR}/atlantis_drift_${PROJECT_NAME}.prom.$$"
METRIC_FILE="${METRIC_DIR}/atlantis_drift_${PROJECT_NAME}.prom"

# =====================
# Functions
# =====================
log() {
  echo "[drift-detection] $1" >&2
}

error_exit() {
  log "ERROR: $1"
  exit 1
}

# =====================
# Main Logic
# =====================
log "Starting drift detection for ${REPO_OWNER}/${REPO_NAME}"
log "Workspace=${WORKSPACE}, Directory=${DIRECTORY}, Project=${PROJECT_NAME}"

# Initialize values
DRIFT_DETECTED=0
RESOURCES_TO_ADD=0
RESOURCES_TO_CHANGE=0
RESOURCES_TO_DESTROY=0

# Validate PLANFILE
if [ -z "${PLANFILE}" ] || [ ! -f "${PLANFILE}" ]; then
  log "Warning: PLANFILE not available or does not exist: ${PLANFILE}"
  log "Skipping drift detection"
  exit 0
fi

log "Analyzing plan file: ${PLANFILE}"

# Check if PLANFILE is binary (Atlantis .tfplan) or text
if file "${PLANFILE}" 2>/dev/null | grep -q "text"; then
  # Text file - read directly
  PLAN_OUTPUT="$(cat "${PLANFILE}")"
else
  # Binary .tfplan file - use terraform show to convert
  log "Converting binary plan file to text..."
  if command -v terraform &> /dev/null; then
    PLAN_OUTPUT="$(terraform show -no-color "${PLANFILE}" 2>/dev/null || cat "${PLANFILE}" 2>/dev/null || echo "")"
  else
    log "Warning: terraform not found, attempting to read file directly"
    PLAN_OUTPUT="$(cat "${PLANFILE}" 2>/dev/null || echo "")"
  fi
fi

log "Plan output length: ${#PLAN_OUTPUT} characters"

# =====================
# Parse Terraform plan
# =====================
if echo "${PLAN_OUTPUT}" | grep -q "Plan:"; then
  PLAN_SUMMARY="$(echo "${PLAN_OUTPUT}" | grep "Plan:" | head -1 | sed 's/^[[:space:]]*//')"
  log "Plan summary detected: ${PLAN_SUMMARY}"

  RESOURCES_TO_ADD="$(echo "${PLAN_SUMMARY}" | grep -o '[0-9]\+ to add' | grep -o '[0-9]\+' || echo 0)"
  RESOURCES_TO_CHANGE="$(echo "${PLAN_SUMMARY}" | grep -o '[0-9]\+ to change' | grep -o '[0-9]\+' || echo 0)"
  RESOURCES_TO_DESTROY="$(echo "${PLAN_SUMMARY}" | grep -o '[0-9]\+ to destroy' | grep -o '[0-9]\+' || echo 0)"

  # Handle empty values
  RESOURCES_TO_ADD="${RESOURCES_TO_ADD:-0}"
  RESOURCES_TO_CHANGE="${RESOURCES_TO_CHANGE:-0}"
  RESOURCES_TO_DESTROY="${RESOURCES_TO_DESTROY:-0}"

  TOTAL_CHANGES=$((RESOURCES_TO_ADD + RESOURCES_TO_CHANGE + RESOURCES_TO_DESTROY))

  if [ "${TOTAL_CHANGES}" -gt 0 ]; then
    DRIFT_DETECTED=1
    log "DRIFT DETECTED: ${TOTAL_CHANGES} total changes"
  else
    log "No drift detected"
  fi

elif echo "${PLAN_OUTPUT}" | grep -q "No changes"; then
  log "No changes — infrastructure matches configuration"
elif echo "${PLAN_OUTPUT}" | grep -qE "will be (created|destroyed|updated|replaced)"; then
  # Fallback: count resource changes from plan output
  DRIFT_DETECTED=1
  RESOURCES_TO_ADD="$(echo "${PLAN_OUTPUT}" | grep -c "will be created" || echo 0)"
  RESOURCES_TO_CHANGE="$(echo "${PLAN_OUTPUT}" | grep -c "will be updated" || echo 0)"
  RESOURCES_TO_DESTROY="$(echo "${PLAN_OUTPUT}" | grep -c "will be destroyed" || echo 0)"
  log "DRIFT DETECTED (parsed from resource lines)"
else
  log "Warning: Unable to parse Terraform plan output"
fi

# =====================
# Metric Export
# =====================
DIRECTORY_LABEL="$(echo "${DIRECTORY}" | tr '/' '_' | sed 's/^\.$/root/')"

log "Exporting metrics to ${METRIC_FILE}"
mkdir -p "${METRIC_DIR}" 2>/dev/null || true

cat > "${METRIC_FILE_TMP}" <<EOF
# HELP linode_atlantis_drift_total Indicates if terraform drift was detected (1) or not (0)
# TYPE linode_atlantis_drift_total gauge
linode_atlantis_drift_total{repo="${REPO_NAME}",repo_owner="${REPO_OWNER}",workspace="${WORKSPACE}",directory="${DIRECTORY_LABEL}",project="${PROJECT_NAME}"} ${DRIFT_DETECTED}

# HELP linode_atlantis_drift_resources_added_total Number of resources to add
# TYPE linode_atlantis_drift_resources_added_total gauge
linode_atlantis_drift_resources_added_total{repo="${REPO_NAME}",repo_owner="${REPO_OWNER}",workspace="${WORKSPACE}",directory="${DIRECTORY_LABEL}",project="${PROJECT_NAME}"} ${RESOURCES_TO_ADD}

# HELP linode_atlantis_drift_resources_changed_total Number of resources to change
# TYPE linode_atlantis_drift_resources_changed_total gauge
linode_atlantis_drift_resources_changed_total{repo="${REPO_NAME}",repo_owner="${REPO_OWNER}",workspace="${WORKSPACE}",directory="${DIRECTORY_LABEL}",project="${PROJECT_NAME}"} ${RESOURCES_TO_CHANGE}

# HELP linode_atlantis_drift_resources_destroyed_total Number of resources to destroy
# TYPE linode_atlantis_drift_resources_destroyed_total gauge
linode_atlantis_drift_resources_destroyed_total{repo="${REPO_NAME}",repo_owner="${REPO_OWNER}",workspace="${WORKSPACE}",directory="${DIRECTORY_LABEL}",project="${PROJECT_NAME}"} ${RESOURCES_TO_DESTROY}

# HELP linode_atlantis_drift_last_check_timestamp Unix timestamp of last drift check
# TYPE linode_atlantis_drift_last_check_timestamp gauge
linode_atlantis_drift_last_check_timestamp{repo="${REPO_NAME}",repo_owner="${REPO_OWNER}",workspace="${WORKSPACE}",directory="${DIRECTORY_LABEL}",project="${PROJECT_NAME}"} $(date +%s)
EOF

# Atomic write
mv "${METRIC_FILE_TMP}" "${METRIC_FILE}" || error_exit "Failed to write metrics file"

log "Metrics exported successfully: drift=${DRIFT_DETECTED}, add=${RESOURCES_TO_ADD}, change=${RESOURCES_TO_CHANGE}, destroy=${RESOURCES_TO_DESTROY}"

# =====================
# Console Summary
# =====================
echo ""
echo "═══════════════════════════════════════════════════════════════"
echo "                    DRIFT DETECTION SUMMARY                     "
echo "═══════════════════════════════════════════════════════════════"
echo ""
if [ "${DRIFT_DETECTED}" -eq 1 ]; then
  echo "  ⚠️  DRIFT DETECTED"
  echo ""
  echo "  Resources to add:     ${RESOURCES_TO_ADD}"
  echo "  Resources to change:  ${RESOURCES_TO_CHANGE}"
  echo "  Resources to destroy: ${RESOURCES_TO_DESTROY}"
else
  echo "  ✅ NO DRIFT DETECTED"
  echo ""
  echo "  Infrastructure matches Terraform configuration."
fi
echo ""
echo "═══════════════════════════════════════════════════════════════"
echo ""
```

---

## Change #2: Update atlantis.yaml in t-o-t

**Repo:** `terraform-observability-team`
**File:** `atlantis.yaml`
**Issue:** The `drift_detection` workflow has an inline script that only prints to console
**Fix:** Replace inline script with call to the deployed drift detection script

### Current Code (Lines 105-127):

```yaml
        - run: |
            echo ""
            echo "       DRIFT DETECTION SUMMARY"
            echo ""

            if grep -q "No changes" "$PLANFILE" 2>/dev/null || \
               grep -q "0 to add, 0 to change, 0 to destroy" "$PLANFILE" 2>/dev/null; then
              echo "NO DRIFT DETECTED"
              echo ""
              echo "Infrastructure matches configuration."
            else
              PLAN_SUMMARY=$(grep "^Plan:" "$PLANFILE" 2>/dev/null | head -1 || true)
              if [ -n "$PLAN_SUMMARY" ]; then
                echo "DRIFT DETECTED"
                echo ""
                echo "$PLAN_SUMMARY"
                echo ""
                echo "Review the plan output above for details."
              else
                echo "Check plan output above for changes."
              fi
            fi
            echo ""
```

### Updated Code:

```yaml
        - run: /opt/atlantis/bin/export-drift-metric.sh
```

### Full Updated drift_detection Workflow (Lines 71-106):

```yaml
  drift_detection:
    plan:
      steps:
        - env:
            name: AWS_ACCESS_KEY_ID
            command: >
              vault read -format=json -field=data
              /infra/data/${PROJECT_NAME}/atlantis/envs/ops/terraform-observability-team
              | jq -r '.AWS_ACCESS_KEY_ID'

        - env:
            name: AWS_SECRET_ACCESS_KEY
            command: >
              vault read -format=json -field=data
              /infra/data/${PROJECT_NAME}/atlantis/envs/ops/terraform-observability-team
              | jq -r '.AWS_SECRET_ACCESS_KEY'

        - env:
            name: TF_VAR_atlantis_approle_role_id
            command: >
              vault read -format=json -field=data
              /infra/data/${PROJECT_NAME}/atlantis/envs/ops/terraform-observability-team
              | jq -r '.TF_VAR_atlantis_approle_role_id'

        - env:
            name: TF_VAR_atlantis_approle_secret_id
            command: >
              vault read -format=json -field=data
              /infra/data/${PROJECT_NAME}/atlantis/envs/ops/terraform-observability-team
              | jq -r '.TF_VAR_atlantis_approle_secret_id'

        - init
        - plan
        - run: /opt/atlantis/bin/export-drift-metric.sh
```

---

## Change #3: Add GitHub Actions Workflow

**Repo:** `terraform-observability-team`
**File:** `.github/workflows/drift-detection.yml` (NEW FILE)
**Purpose:** Trigger drift detection on a schedule (nightly) and on-demand

### New File Content:

```yaml
name: Terraform Drift Detection

on:
  # Run nightly at 2 AM UTC
  schedule:
    - cron: '0 2 * * *'

  # Allow manual trigger for testing
  workflow_dispatch:
    inputs:
      project:
        description: 'Atlantis project to check (default: drift-check)'
        required: false
        default: 'drift-check'

jobs:
  trigger-drift-check:
    runs-on: ["self-hosted"]
    container:
      image: ubuntu:24.04
    permissions:
      pull-requests: write
      contents: write
      issues: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install GitHub CLI
        run: |
          apt-get update && apt-get install -y curl
          curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
          echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list > /dev/null
          apt-get update && apt-get install -y gh

      - name: Find or create drift-detection PR
        id: find-pr
        env:
          GH_TOKEN: ${{ secrets.OPS_BOT_TOKEN }}
        run: |
          # Look for existing drift-detection PR
          PR_NUMBER=$(gh pr list --state open --head drift-detection --json number --jq '.[0].number // empty')

          if [ -z "$PR_NUMBER" ]; then
            echo "No existing drift-detection PR found"
            echo "pr_exists=false" >> $GITHUB_OUTPUT
          else
            echo "Found existing PR: #$PR_NUMBER"
            echo "pr_exists=true" >> $GITHUB_OUTPUT
            echo "pr_number=$PR_NUMBER" >> $GITHUB_OUTPUT
          fi

      - name: Create drift-detection branch and PR
        if: steps.find-pr.outputs.pr_exists == 'false'
        id: create-pr
        env:
          GH_TOKEN: ${{ secrets.OPS_BOT_TOKEN }}
        run: |
          # Configure git
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          # Get default branch
          DEFAULT_BRANCH=$(gh api repos/${{ github.repository }} --jq '.default_branch')

          # Create or update drift-detection branch
          git checkout -B drift-detection

          # Create a timestamp file to trigger Atlantis
          echo "Last drift check: $(date -u '+%Y-%m-%d %H:%M:%S UTC')" > .drift-check-timestamp

          git add .drift-check-timestamp
          git commit -m "chore: trigger drift detection check" || echo "No changes to commit"
          git push -f origin drift-detection

          # Create PR
          PR_URL=$(gh pr create \
            --title "[Automated] Terraform Drift Detection" \
            --body "This PR is automatically created to trigger Atlantis drift detection.

          **Do not merge this PR.** It will be automatically updated nightly.

          To manually check for drift, comment: \`atlantis plan -p drift-check\`" \
            --base "$DEFAULT_BRANCH" \
            --head drift-detection \
            --draft)

          PR_NUMBER=$(echo "$PR_URL" | grep -oE '[0-9]+$')
          echo "pr_number=$PR_NUMBER" >> $GITHUB_OUTPUT
          echo "Created PR #$PR_NUMBER"

      - name: Update existing PR branch
        if: steps.find-pr.outputs.pr_exists == 'true'
        env:
          GH_TOKEN: ${{ secrets.OPS_BOT_TOKEN }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          git fetch origin drift-detection || true
          git checkout drift-detection || git checkout -b drift-detection

          # Update timestamp to trigger new Atlantis plan
          echo "Last drift check: $(date -u '+%Y-%m-%d %H:%M:%S UTC')" > .drift-check-timestamp

          git add .drift-check-timestamp
          git commit -m "chore: trigger drift detection check" || echo "No changes"
          git push origin drift-detection

      - name: Trigger Atlantis drift check
        env:
          GH_TOKEN: ${{ secrets.OPS_BOT_TOKEN }}
        run: |
          PR_NUMBER="${{ steps.find-pr.outputs.pr_number || steps.create-pr.outputs.pr_number }}"
          PROJECT="${{ github.event.inputs.project || 'drift-check' }}"

          echo "Triggering Atlantis plan on PR #$PR_NUMBER for project: $PROJECT"

          gh pr comment "$PR_NUMBER" --body "atlantis plan -p $PROJECT"

      - name: Summary
        run: |
          PR_NUMBER="${{ steps.find-pr.outputs.pr_number || steps.create-pr.outputs.pr_number }}"
          echo "## Drift Detection Triggered" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "- PR: #$PR_NUMBER" >> $GITHUB_STEP_SUMMARY
          echo "- Project: ${{ github.event.inputs.project || 'drift-check' }}" >> $GITHUB_STEP_SUMMARY
          echo "- Time: $(date -u '+%Y-%m-%d %H:%M:%S UTC')" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "Atlantis will run the drift detection workflow and export metrics." >> $GITHUB_STEP_SUMMARY
```

---

## Change #4: Add Prometheus Alert Rule

**Location:** Prometheus configuration (likely managed via Salt or Terraform)
**Purpose:** Alert when drift is detected

### Alert Rule:

```yaml
groups:
  - name: terraform_drift
    rules:
      - alert: TerraformDriftDetected
        expr: linode_atlantis_drift_total > 0
        for: 5m
        labels:
          severity: warning
          team: o11y
        annotations:
          summary: "Terraform drift detected in {{ $labels.repo }}"
          description: |
            Drift detected in {{ $labels.repo_owner }}/{{ $labels.repo }}
            Project: {{ $labels.project }}
            Directory: {{ $labels.directory }}

            Resources to add: {{ with printf "linode_atlantis_drift_resources_added_total{repo='%s',project='%s'}" $labels.repo $labels.project | query }}{{ . | first | value }}{{ end }}
            Resources to change: {{ with printf "linode_atlantis_drift_resources_changed_total{repo='%s',project='%s'}" $labels.repo $labels.project | query }}{{ . | first | value }}{{ end }}
            Resources to destroy: {{ with printf "linode_atlantis_drift_resources_destroyed_total{repo='%s',project='%s'}" $labels.repo $labels.project | query }}{{ . | first | value }}{{ end }}
          runbook_url: "https://bits.linode.com/{{ $labels.repo_owner }}/{{ $labels.repo }}"

      - alert: TerraformDriftCheckStale
        expr: time() - linode_atlantis_drift_last_check_timestamp > 172800
        for: 1h
        labels:
          severity: warning
          team: o11y
        annotations:
          summary: "Terraform drift check is stale for {{ $labels.repo }}"
          description: |
            No drift check has run for {{ $labels.repo_owner }}/{{ $labels.repo }} in over 48 hours.
            Last check: {{ $value | humanizeTimestamp }}
```

---

## Change #5: Add .gitignore Entry (Optional)

**Repo:** `terraform-observability-team`
**File:** `.gitignore`
**Purpose:** Ignore the drift check timestamp file

### Add this line:

```
.drift-check-timestamp
```

---

## Implementation Order

### Phase 1: Deploy Script Update (atlantis-formula)

1. **Create PR** in `atlantis-formula`:
   - Update `atlantis/files/export-drift-metric.sh` with the new content
   - PR Title: `fix: Handle binary .tfplan files in drift detection script`

2. **After merge**, Salt will deploy the updated script to Atlantis server on next highstate

### Phase 2: Update t-o-t Repo

3. **Create PR** in `terraform-observability-team`:
   - Update `atlantis.yaml` - change the run step to call the script
   - Add `.github/workflows/drift-detection.yml`
   - Update `.gitignore` (optional)
   - PR Title: `feat: Add automated drift detection with Prometheus metrics`

### Phase 3: Add Alerting

4. **Add Prometheus alert rule** (wherever alerts are managed)
   - Add the `TerraformDriftDetected` alert
   - Add the `TerraformDriftCheckStale` alert

5. **Configure Alertmanager routing** to send to `#notification-o11y` or PagerDuty

---

## File Change Summary

| Repo | File | Action | Lines Changed |
|------|------|--------|---------------|
| atlantis-formula | `atlantis/files/export-drift-metric.sh` | UPDATE | ~50 lines added |
| t-o-t | `atlantis.yaml` | UPDATE | Lines 105-127 → 1 line |
| t-o-t | `.github/workflows/drift-detection.yml` | NEW | ~100 lines |
| t-o-t | `.gitignore` | UPDATE | 1 line added |
| Prometheus | Alert rules | NEW | ~30 lines |

---

## Testing Plan

### After atlantis-formula PR is merged:

1. SSH to Atlantis server
2. Verify script is deployed: `cat /opt/atlantis/bin/export-drift-metric.sh`
3. Verify script is executable: `ls -la /opt/atlantis/bin/export-drift-metric.sh`

### After t-o-t PR is merged:

1. Manually trigger the GitHub Action:
   - Go to Actions → Terraform Drift Detection → Run workflow
2. Check the PR created/updated
3. Verify Atlantis comment appears
4. Check metrics on Atlantis server:
   ```bash
   cat /var/lib/prometheus-node-exporter/atlantis_drift_drift-check.prom
   ```
5. Query Prometheus:
   ```
   linode_atlantis_drift_total{repo="terraform-observability-team"}
   ```

---

## Metrics Available After Implementation

| Metric | Description | Labels |
|--------|-------------|--------|
| `linode_atlantis_drift_total` | 1 if drift detected, 0 otherwise | repo, repo_owner, workspace, directory, project |
| `linode_atlantis_drift_resources_added_total` | Count of resources to add | same |
| `linode_atlantis_drift_resources_changed_total` | Count of resources to change | same |
| `linode_atlantis_drift_resources_destroyed_total` | Count of resources to destroy | same |
| `linode_atlantis_drift_last_check_timestamp` | Unix timestamp of last check | same |

---

## Rollout to Other Repos

Once working for t-o-t, add to other repos by:

1. Add `drift-check` project to their `atlantis.yaml`:
   ```yaml
   - dir: '.'
     name: drift-check
     workflow: drift_detection
     autoplan:
       enabled: false
   ```

2. Add the `drift_detection` workflow (copy from t-o-t)

3. Add the GitHub Actions workflow (copy from t-o-t)

Repos to consider:
- `terraform-grafana-config`
- `terraform-module-infra` (has workspaces - may need adjustment)

---

## Questions for Team

1. **Alert routing**: Should drift alerts go to Slack only, or also PagerDuty (warning severity)?

2. **Schedule**: Is 2 AM UTC good, or should it run at a different time?

3. **Stale check alert**: Should we alert if drift check hasn't run in 48 hours?

4. **Other repos**: Which other Terraform repos should get drift detection next?
