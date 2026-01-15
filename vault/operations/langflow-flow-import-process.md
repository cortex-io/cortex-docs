---
title: "Langflow Flow Import Process"
type: operations
category: langflow
status: active
created: "2026-01-15"
updated: "2026-01-15"
---

# Langflow Flow Import Process

## Overview

This document describes the process for adding new workflows (flows) to Langflow. Langflow stores flows in PostgreSQL but can also read them from mounted JSON files. However, **mounted files are NOT automatically imported** into the database.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│         GitOps Repository (cortex-gitops)            │
│                                                       │
│  apps/cortex-system/langflow-workflows/              │
│  ├── 01-k8s-health-monitor.json                     │
│  ├── 02-mcp-orchestration-symphony.json             │
│  ├── ...                                             │
│  └── 11-cortex-daily-brief.json                     │
└─────────────────────────────────────────────────────┘
                     ↓
         (bundled into ConfigMap)
                     ↓
┌─────────────────────────────────────────────────────┐
│      Kubernetes ConfigMap (langflow-workflows)       │
│                                                       │
│  Mounted to: /app/flows/ in Langflow pod            │
└─────────────────────────────────────────────────────┘
                     ↓
         (manual import required)
                     ↓
┌─────────────────────────────────────────────────────┐
│        Langflow PostgreSQL Database                  │
│                                                       │
│  Tables: flow, user, folder                         │
│  Flows must be imported via API or UI               │
└─────────────────────────────────────────────────────┘
```

---

## Two-Step Process

When adding new flows to Langflow, there are **two required steps**:

### Step 1: Add Flow JSON to ConfigMap

The flow JSON files must be added to the `langflow-workflows` ConfigMap in the `cortex-system` namespace.

#### Option A: Edit ConfigMap Directly (Not Recommended)

```bash
kubectl edit configmap langflow-workflows -n cortex-system
```

Add the new flow JSON file(s) to the ConfigMap `data:` section.

**Warning**: Manual ConfigMap edits will be overwritten by ArgoCD on next sync.

#### Option B: Add to GitOps Repository (Recommended)

```bash
cd ~/Projects/cortex-gitops

# Add new flow JSON file
cat > apps/cortex-system/langflow-workflows/12-new-workflow.json << 'EOF'
{
  "name": "New Workflow",
  "description": "Description here",
  "data": {
    "nodes": [...],
    "edges": [...]
  }
}
EOF

# Regenerate ConfigMap
cd apps/cortex-system/langflow-workflows
cat > ../langflow-workflows-configmap.yaml << 'CONFIGMAP_EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: langflow-workflows
  namespace: cortex-system
  labels:
    app: langflow
    component: workflows
data:
CONFIGMAP_EOF

for f in *.json; do
  echo "  $f: |" >> ../langflow-workflows-configmap.yaml
  sed 's/^/    /' "$f" >> ../langflow-workflows-configmap.yaml
done

# Commit and push
cd ~/Projects/cortex-gitops
git add apps/cortex-system/langflow-workflows/
git commit -m "Add new Langflow workflow: 12-new-workflow"
git push origin main

# ArgoCD will sync automatically
```

**Result**: Flow files are mounted at `/app/flows/` in the Langflow pod.

**Verification**:
```bash
# List mounted flow files
kubectl exec -n cortex-system deploy/langflow -- ls -la /app/flows/

# Should show all workflow JSON files
```

---

### Step 2: Import Flows into Database

**CRITICAL**: Flows mounted at `/app/flows/` are **NOT automatically imported** into Langflow's PostgreSQL database. They must be manually imported.

#### Required Information

Before importing, you need two IDs from the Langflow PostgreSQL database:

1. **Folder ID**: The folder where flows will be imported
2. **User ID**: The user who will own the flows

##### Finding IDs

Connect to Langflow's PostgreSQL database:

```bash
# Option 1: Via psql from Langflow pod
kubectl exec -n cortex-system deploy/langflow -- psql -U langflow -d langflow -c "
SELECT
  u.id as user_id,
  u.username,
  f.id as folder_id,
  f.name as folder_name
FROM \"user\" u
LEFT JOIN folder f ON f.user_id = u.id
WHERE u.username = 'langflow';
"

# Option 2: Via pgAdmin UI
# Connect to PostgreSQL service and run:
SELECT id FROM "user" WHERE username = 'langflow';
SELECT id FROM folder WHERE user_id = '<user_id_from_above>';
```

**Example Output**:
```
 user_id                              | username | folder_id                            | folder_name
--------------------------------------+----------+--------------------------------------+-------------
 167cbefd-5e87-4968-99df-25586ef0bd29 | langflow | 05a21d71-dbba-4609-925b-e5b7e8d6aae7 | My Flows
```

#### Import Script

Once you have the `FOLDER_ID` and `USER_ID`, run this import script:

```bash
kubectl exec -n cortex-system deploy/langflow -- python3 -c "
import json
import os
import requests

# IMPORTANT: Update these IDs from your database query above
FOLDER_ID = '05a21d71-dbba-4609-925b-e5b7e8d6aae7'  # Update this!
USER_ID = '167cbefd-5e87-4968-99df-25586ef0bd29'    # Update this!

# Loop through all flow files in /app/flows/
for filename in sorted(os.listdir('/app/flows')):
    if filename.endswith('.json'):
        # Read flow JSON
        with open(f'/app/flows/{filename}', 'r') as f:
            flow = json.load(f)

        # Add user_id and folder_id
        flow['user_id'] = USER_ID
        flow['folder_id'] = FOLDER_ID

        # POST to Langflow API
        resp = requests.post('http://localhost:7860/api/v1/flows/', json=flow)
        print(f'{filename}: {resp.status_code}')
"
```

**Expected Output**:
```
01-k8s-health-monitor.json: 201
02-mcp-orchestration-symphony.json: 201
03-github-security-scanner.json: 201
...
11-cortex-daily-brief.json: 201
```

**Status Codes**:
- `201`: Successfully created
- `200`: Updated existing flow
- `400`: Bad request (check flow JSON format)
- `409`: Conflict (flow already exists with same name)
- `500`: Server error (check Langflow logs)

#### Verification

Check that flows were imported:

```bash
# Via Langflow API
kubectl exec -n cortex-system deploy/langflow -- curl -s http://localhost:7860/api/v1/flows/ | jq '.[] | {name: .name, id: .id}'

# Or via PostgreSQL
kubectl exec -n cortex-system deploy/langflow -- psql -U langflow -d langflow -c "
SELECT name, description, created_at
FROM flow
ORDER BY created_at DESC
LIMIT 10;
"
```

---

## Important Notes

### Duplicate Prevention

**Warning**: Running the import script multiple times will create duplicate flows.

#### Option 1: Check Before Import

Modify the script to check if flow exists:

```python
# Get existing flows
existing = requests.get('http://localhost:7860/api/v1/flows/').json()
existing_names = {f['name'] for f in existing}

for filename in sorted(os.listdir('/app/flows')):
    if filename.endswith('.json'):
        with open(f'/app/flows/{filename}', 'r') as f:
            flow = json.load(f)

        # Skip if already exists
        if flow['name'] in existing_names:
            print(f'{filename}: SKIPPED (already exists)')
            continue

        flow['user_id'] = USER_ID
        flow['folder_id'] = FOLDER_ID
        resp = requests.post('http://localhost:7860/api/v1/flows/', json=flow)
        print(f'{filename}: {resp.status_code}')
```

#### Option 2: Delete Existing Flows First

```bash
# Delete all flows (WARNING: This removes ALL flows)
kubectl exec -n cortex-system deploy/langflow -- psql -U langflow -d langflow -c "DELETE FROM flow;"

# Then run import script
```

#### Option 3: Delete Specific Flow

```bash
# Find flow ID
kubectl exec -n cortex-system deploy/langflow -- psql -U langflow -d langflow -c "
SELECT id, name FROM flow WHERE name = 'Workflow Name';
"

# Delete by ID
kubectl exec -n cortex-system deploy/langflow -- curl -X DELETE http://localhost:7860/api/v1/flows/<flow-id>
```

---

## Alternative: Manual UI Import

Users can also upload flows directly via the Langflow UI:

### Step 1: Export Flow JSON from GitOps

```bash
# From local machine
cd ~/Projects/cortex-gitops/apps/cortex-system/langflow-workflows
cat 11-cortex-daily-brief.json
```

Copy the JSON content.

### Step 2: Import via Langflow UI

1. Open Langflow UI: `https://langflow.ry-ops.dev`
2. Click **"Import"** button (top-right or sidebar)
3. Select **"Upload JSON"**
4. Paste the flow JSON
5. Click **"Import"**

**Pros**:
- No command-line access needed
- Visual confirmation of import
- Can edit flow immediately after import

**Cons**:
- Manual process (not automated)
- Must repeat for each flow
- No version control after import

---

## Automation with GitOps

### Future Enhancement: Auto-Import Job

To fully automate flow imports, create a Kubernetes Job that runs after ConfigMap updates:

**File**: `apps/cortex-system/langflow-flow-import-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: langflow-flow-import
  namespace: cortex-system
  labels:
    app: langflow
    job-type: flow-import
spec:
  ttlSecondsAfterFinished: 3600
  backoffLimit: 3
  template:
    metadata:
      labels:
        app: langflow
        job-type: flow-import
    spec:
      restartPolicy: OnFailure
      containers:
      - name: flow-importer
        image: python:3.11-slim
        imagePullPolicy: IfNotPresent
        command:
        - /bin/bash
        - -c
        - |
          pip install requests
          python3 << 'PYTHON_EOF'
          import json
          import os
          import requests

          FOLDER_ID = '05a21d71-dbba-4609-925b-e5b7e8d6aae7'
          USER_ID = '167cbefd-5e87-4968-99df-25586ef0bd29'

          # Get existing flows
          existing = requests.get('http://langflow:7860/api/v1/flows/').json()
          existing_names = {f['name'] for f in existing}

          for filename in sorted(os.listdir('/app/flows')):
              if filename.endswith('.json'):
                  with open(f'/app/flows/{filename}', 'r') as f:
                      flow = json.load(f)

                  if flow['name'] in existing_names:
                      print(f'{filename}: SKIPPED (exists)')
                      continue

                  flow['user_id'] = USER_ID
                  flow['folder_id'] = FOLDER_ID
                  resp = requests.post('http://langflow:7860/api/v1/flows/', json=flow)
                  print(f'{filename}: {resp.status_code}')
          PYTHON_EOF
        volumeMounts:
        - name: flows
          mountPath: /app/flows
          readOnly: true
      volumes:
      - name: flows
        configMap:
          name: langflow-workflows
```

**Trigger**: Run manually or via ArgoCD hook after ConfigMap updates.

---

## Troubleshooting

### Issue: Import Script Returns 500 Error

**Symptom**:
```
11-cortex-daily-brief.json: 500
```

**Diagnosis**:
```bash
# Check Langflow logs
kubectl logs -n cortex-system deploy/langflow --tail=50

# Check if Langflow is healthy
kubectl exec -n cortex-system deploy/langflow -- curl -s http://localhost:7860/api/v1/health
```

**Common Causes**:
- Langflow not fully started
- PostgreSQL connection issues
- Invalid JSON in flow file
- Missing required fields in flow JSON

**Solution**:
Wait 30 seconds and retry, or fix the flow JSON format.

---

### Issue: Flows Not Appearing in UI

**Symptom**: Import script returns 201, but flows don't show in Langflow UI

**Diagnosis**:
```bash
# Check database directly
kubectl exec -n cortex-system deploy/langflow -- psql -U langflow -d langflow -c "
SELECT id, name, user_id, folder_id FROM flow;
"

# Verify user and folder IDs match
```

**Common Causes**:
- Logged in as different user in UI
- Flows imported to wrong folder
- Browser cache showing old state

**Solution**:
1. Refresh browser (Ctrl+Shift+R)
2. Log out and log back in
3. Verify `USER_ID` and `FOLDER_ID` are correct
4. Check folder filter in Langflow UI sidebar

---

### Issue: Duplicate Flows Created

**Symptom**: Multiple copies of same flow in Langflow UI

**Diagnosis**:
```bash
# Count duplicates
kubectl exec -n cortex-system deploy/langflow -- psql -U langflow -d langflow -c "
SELECT name, COUNT(*)
FROM flow
GROUP BY name
HAVING COUNT(*) > 1;
"
```

**Solution**:
```bash
# Delete all flows and re-import
kubectl exec -n cortex-system deploy/langflow -- psql -U langflow -d langflow -c "DELETE FROM flow;"

# Then run import script once
```

---

### Issue: ConfigMap Changes Not Reflected in Pod

**Symptom**: New flow files not visible in `/app/flows/`

**Diagnosis**:
```bash
# Check ConfigMap has new flows
kubectl get configmap langflow-workflows -n cortex-system -o yaml | grep "11-cortex-daily-brief.json"

# Check what's mounted in pod
kubectl exec -n cortex-system deploy/langflow -- ls -la /app/flows/
```

**Common Causes**:
- ConfigMap not updated in cluster
- Pod needs restart to pick up ConfigMap changes
- ArgoCD hasn't synced yet

**Solution**:
```bash
# Force ArgoCD sync
kubectl patch application cortex-system -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# Or restart Langflow pod
kubectl rollout restart deployment/langflow -n cortex-system
```

---

## Best Practices

### 1. Use GitOps for Flow Management

**Always** add flow JSON files to the GitOps repository first, then import to database.

**Benefits**:
- Version control for flow changes
- Audit trail of who changed what
- Easy rollback to previous versions
- Disaster recovery (can rebuild from Git)

### 2. Document Flow Purpose

Add clear `name` and `description` fields to each flow:

```json
{
  "name": "K8s Health Monitor",
  "description": "Real-time cluster health monitoring with alerts",
  "data": { ... }
}
```

### 3. Use Semantic Versioning in Names

For major flow changes, update the flow name:

```
01-k8s-health-monitor.json       (v1.0)
01-k8s-health-monitor-v2.json    (v2.0 - breaking changes)
```

### 4. Test Flows Before Committing

1. Create flow in Langflow UI
2. Test thoroughly with real data
3. Export JSON from UI
4. Clean up and format JSON
5. Add to GitOps repository
6. Import script will recreate in database

### 5. Backup Flows Regularly

```bash
# Export all flows from database
kubectl exec -n cortex-system deploy/langflow -- python3 -c "
import requests
import json

flows = requests.get('http://localhost:7860/api/v1/flows/').json()
with open('/tmp/flows-backup.json', 'w') as f:
    json.dump(flows, f, indent=2)
"

# Copy backup to local machine
kubectl cp cortex-system/langflow-xxx:/tmp/flows-backup.json ./flows-backup.json
```

---

## Related Documentation

- [Langflow Workflows](../architecture/langflow-workflows.md)
- [YouTube Learning System](../architecture/youtube-learning-system.md)
- [MCP Server Architecture](../architecture/mcp-architecture.md)
- [ArgoCD GitOps Workflow](../operations/argocd-gitops-workflow.md)

---

## Summary

**Two-Step Process**:
1. Add flow JSON to ConfigMap (via GitOps or `kubectl edit`)
2. Import flows to database (via import script or UI)

**Key Point**: Mounted files at `/app/flows/` are **NOT** automatically imported.

**Recommendation**: Use GitOps workflow for version control and automation.

---

**Status**: Active
**Last Updated**: 2026-01-15
**Maintained By**: Cortex Platform Team
