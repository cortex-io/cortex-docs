---
title: "Infrastructure Fixes - 2026-01-21"
type: operations
created: "2026-01-21"
updated: "2026-01-21"
tags:
  - infrastructure
  - fixes
  - dns
  - registry
  - quotas
---

# Infrastructure Fixes - 2026-01-21

## Issues Identified

During the Layer Activator and Memory Service deployment, several recurring infrastructure issues were identified:

1. **DNS Resolution Failures** - Intermittent on k3s-worker03
2. **ImagePullBackOff** - Registry mirror not being used on all nodes
3. **ResourceQuota Exhaustion** - ConfigMaps limit reached
4. **Container Dependency Installation Failures** - pip install timing out

---

## 1. DNS Resolution on k3s-worker03

### Problem
Pods scheduled on k3s-worker03 experience intermittent DNS resolution failures:
```
Failed to establish a new connection: [Errno -3] Temporary failure in name resolution
```

### Diagnosis
- CoreDNS is running on k3s-master01
- DNS works from other worker nodes (01, 02, 04)
- k3s-worker03 specifically has connectivity issues to CoreDNS

### Root Cause
Likely causes:
1. Flannel/CNI issue on k3s-worker03
2. iptables rules not properly configured
3. kube-proxy not syncing properly

### Fix - SSH to k3s-worker03 and run:
```bash
# 1. Check DNS configuration
cat /etc/resolv.conf

# 2. Test DNS resolution
nslookup pypi.org 10.43.0.10

# 3. Check CNI interfaces
ip addr show flannel.1
ip addr show cni0

# 4. Restart k3s-agent to resync
sudo systemctl restart k3s-agent

# 5. If still failing, reboot the node
sudo reboot
```

### Workaround
Until fixed, add node anti-affinity to deployments:
```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: kubernetes.io/hostname
          operator: NotIn
          values:
          - k3s-worker03
```

---

## 2. Registry Mirror Configuration

### Problem
Image pulls fail with:
```
failed to do request: Head "https://registry-1.docker.io/v2/...": remote error: tls: handshake failure
```

Nodes are trying to pull from Docker Hub directly instead of using the registry-mirror.

### Root Cause
The k3s registries.yaml configuration is missing or not applied on some nodes.

### Fix - On each k3s node, create/update:

**File: `/etc/rancher/k3s/registries.yaml`**
```yaml
mirrors:
  docker.io:
    endpoint:
      - "http://registry-mirror.registry.svc.cluster.local:5000"
  registry-1.docker.io:
    endpoint:
      - "http://registry-mirror.registry.svc.cluster.local:5000"
```

Then restart k3s:
```bash
# On master nodes:
sudo systemctl restart k3s

# On worker nodes:
sudo systemctl restart k3s-agent
```

### Verify
```bash
# From any pod:
curl http://registry-mirror.registry.svc.cluster.local:5000/v2/_catalog
```

---

## 3. ResourceQuota Management

### Problem
ConfigMap creation failed:
```
exceeded quota: cortex-system-quota, requested: configmaps=1, used: configmaps=60, limited: configmaps=60
```

### Current Quotas
```
cortex-system-quota:
  configmaps: 80 (increased from 60)
  pods: 100
  services: 50
```

### Fix Applied
```bash
kubectl patch resourcequota cortex-system-quota -n cortex-system \
  --type='json' -p='[{"op": "replace", "path": "/spec/hard/configmaps", "value": "80"}]'
```

### Monitoring
Deployed quota-monitor service:
- Endpoint: `http://quota-monitor.cortex-system.svc.cluster.local:8080`
- Metrics: `/metrics` (Prometheus compatible)
- Alerts: `/alerts` (warning at 75%, critical at 90%)

---

## 4. Dependency Installation Pattern

### Problem
Container startup fails when pip install times out due to DNS issues.

### Solution - Init Container with Retry
```yaml
initContainers:
- name: install-deps
  image: python:3.11-slim
  command:
  - /bin/bash
  - -c
  - |
    set -e
    for i in 1 2 3 4 5; do
      if pip install --no-cache-dir --target=/deps fastapi uvicorn ...; then
        exit 0
      fi
      echo "Attempt $i failed, waiting 10s..."
      sleep 10
    done
    exit 1
  volumeMounts:
  - name: deps
    mountPath: /deps

# Main container adds:
env:
- name: PYTHONPATH
  value: /deps
volumeMounts:
- name: deps
  mountPath: /deps

volumes:
- name: deps
  emptyDir: {}
```

### Long-term Solution
Build pre-baked container images with dependencies:
- Location: `cortex-platform/build/`
- Base image: `cortex-python-base:3.11`
- Build script: `build-and-push.sh`

---

## Summary of Applied Fixes

| Issue | Status | Fix Applied |
|-------|--------|-------------|
| k3s-worker03 DNS | Workaround | Node anti-affinity |
| Registry mirror | Pending | Need to update registries.yaml on all nodes |
| ResourceQuota | Fixed | Increased to 80 ConfigMaps |
| pip install failures | Fixed | Init container with retry pattern |

---

## Action Items

1. [ ] SSH to each k3s node and configure registries.yaml
2. [ ] SSH to k3s-worker03 and diagnose/fix DNS
3. [ ] Build cortex-python-base image and push to registry
4. [ ] Deploy quota-monitor service

---

## Files Created

- `cortex-platform/infrastructure/init-container-template.yaml`
- `cortex-platform/infrastructure/k3s-worker03-repair.sh`
- `cortex-platform/build/Dockerfile.cortex-python-base`
- `cortex-platform/build/build-and-push.sh`
- `cortex-platform/services/quota-monitor/main.py`
- `cortex-gitops/apps/cortex-system/quota-monitor-deployment.yaml`
- `cortex-gitops/apps/registry/registry-mirror-deployment.yaml`
