---
title: "Cortex Infrastructure State"
type: operations
severity: high
created: "2026-01-20"
updated: "2026-01-20"
tags:
  - infrastructure
  - status
  - mcp-servers
  - agents
  - layer-stack
---

# Cortex Infrastructure State
**Date**: 2026-01-20
**Purpose**: Single source of truth for current infrastructure state

---

## Executive Summary

| Metric | Value | Status |
|--------|-------|--------|
| Total Nodes | 7 (3 masters, 4 workers) | OK |
| Total Pods | 228 | Mixed |
| Running Pods | 127 (56%) | Warning |
| Problem Pods | 67 (29%) | Critical |
| Cortex Namespaces | 25 | OK |
| MCP Servers | 12 total (7 running, 5 broken) | Critical |
| ArgoCD Apps | 12 (5 Degraded, 3 Progressing) | Warning |
| Memory Usage | 33-65% per node | OK |
| CPU Usage | 2-8% per node | OK |

---

## Cluster Nodes

| Node | Role | IP | CPU | Memory | Status |
|------|------|-----|-----|--------|--------|
| k3s-master01 | control-plane,etcd,master | 10.88.145.190 | 4% | 41% | Ready |
| k3s-master02 | control-plane,etcd,master | 10.88.145.193 | 6% | 65% | Ready |
| k3s-master03 | control-plane,etcd,master | 10.88.145.196 | 8% | 62% | Ready |
| k3s-worker01 | worker | 10.88.145.191 | 2% | 35% | Ready |
| k3s-worker02 | worker | 10.88.145.192 | 2% | 47% | Ready |
| k3s-worker03 | worker | 10.88.145.194 | 2% | 33% | Ready |
| k3s-worker04 | worker | 10.88.145.195 | 3% | 42% | Ready |

**Cluster Version**: v1.33.6+k3s1
**OS**: Ubuntu 24.04.3 LTS
**Kernel**: 6.8.0-90-generic
**Runtime**: containerd 2.1.5-k3s1.33
**Age**: 31 days

---

## Pod Status Summary

| Status | Count | Percentage | Action Required |
|--------|-------|------------|-----------------|
| Running | 127 | 56% | None |
| Completed | 34 | 15% | Normal (jobs) |
| ContainerCreating | 23 | 10% | Investigate PVC/image issues |
| ImagePullBackOff | 18 | 8% | Fix registry/image references |
| CrashLoopBackOff | 10 | 4% | Fix application bugs |
| Error | 9 | 4% | Investigate failures |
| Unknown | 4 | 2% | Node communication issues |
| Pending | 1 | <1% | Resource constraints |
| Init failures | 2 | <1% | Init container issues |

---

## MCP Server Inventory

### Running MCP Servers (7)

| Server | Namespace | Restarts | Purpose |
|--------|-----------|----------|---------|
| cortex-mcp-server | cortex-system | 0 | Core Cortex operations |
| kubernetes-mcp-server | cortex-system | 1 | K8s cluster management |
| unifi-mcp-server | cortex-system | 0 | Network infrastructure |
| sandfly-mcp-server | cortex-system | 1 | Linux security scanning |
| cloudflare-mcp-server | cortex-system | 0 | DNS/CDN management |
| langflow-chat-mcp-server | cortex-system | 4 | Langflow integration |
| mcp-server (knowledge) | cortex-knowledge | 0 | Knowledge graph operations |

### Broken MCP Servers (5)

| Server | Namespace | Status | Restarts | Issue |
|--------|-----------|--------|----------|-------|
| github-mcp-server | cortex-system | Running | **709** | Crash loop - needs investigation |
| github-security-mcp-server | cortex-system | Init:ImagePullBackOff | 0 | Image pull failure |
| proxmox-mcp-server | cortex-system | Unknown | 0 | Node communication lost |
| n8n-mcp-server | cortex-system | Unknown | 0 | Node communication lost |
| cortex-desktop-mcp | cortex | ImagePullBackOff | 0 | Image pull failure |

---

## Cortex Namespaces (25)

| Namespace | Purpose | Health |
|-----------|---------|--------|
| cortex | Main orchestration | Degraded |
| cortex-ai-infra | AI infrastructure (Langflow, Phoenix) | Healthy |
| cortex-autonomous | Autonomous operations | Unknown |
| cortex-change-mgmt | Change management | Unknown |
| cortex-chat | Chat interfaces | Degraded |
| cortex-cicd | CI/CD pipelines (Tekton) | Progressing |
| cortex-control-plane | Control plane services | Unknown |
| cortex-csaf | CSAF vulnerability feeds | Progressing |
| cortex-dev | Development tools | Degraded |
| cortex-governance | Governance services | Unknown |
| cortex-itil | ITIL processes | Unknown |
| cortex-itil-stream2 | ITIL stream 2 | Unknown |
| cortex-knowledge | Knowledge graph, Phoenix | Degraded |
| cortex-lifecycle | Lifecycle management | Stuck |
| cortex-live | Live services | Unknown |
| cortex-metrics | Metrics collection | Unknown |
| cortex-orchestration | Orchestration layer | Unknown |
| cortex-school | Learning pipeline (Layer Stack) | Progressing |
| cortex-security | Security tools (Falco, Trivy) | Progressing |
| cortex-service-desk | Service desk | Unknown |
| cortex-standards | Standards enforcement | Unknown |
| cortex-system | Core system services | Degraded |
| cortex-test | Testing environment | Unknown |
| cortex-tui | Terminal UI | Unknown |
| cortex-youtube | YouTube ingestion | Unknown |

---

## ArgoCD Application Status

| Application | Sync | Health | Issue |
|-------------|------|--------|-------|
| checkmk | Unknown | Healthy | - |
| cortex-ai-infra | Unknown | Healthy | - |
| cortex-chat | Unknown | **Degraded** | ImagePullBackOff on registry |
| cortex-cicd | Unknown | Progressing | - |
| cortex-core | Unknown | **Degraded** | Multiple failures |
| cortex-csaf | Unknown | Progressing | CrashLoopBackOff |
| cortex-dev | Unknown | **Degraded** | Unknown |
| cortex-knowledge | Unknown | **Degraded** | CrashLoopBackOff on graph-api |
| cortex-school | Unknown | Progressing | CrashLoopBackOff on workers |
| cortex-security | Unknown | Progressing | - |
| cortex-system | Unknown | **Degraded** | Multiple MCP failures |
| default-apps | Unknown | Healthy | - |

---

## Critical Problem Pods

### Highest Priority (Fix First)

| Pod | Namespace | Status | Restarts | Root Cause |
|-----|-----------|--------|----------|------------|
| github-mcp-server | cortex-system | Running | 709 | App bug - constant restarts |
| cortex-resource-manager | cortex-system | Unknown | 1971 | Node issue + app bug |
| implementation-workers | cortex-school | CrashLoopBackOff | 877 | App configuration |
| docling-service | cortex-system | CrashLoopBackOff | 838 | App bug |
| csaf-registry (x2) | cortex-csaf | CrashLoopBackOff | 750+ | App bug |
| knowledge-graph-api | cortex-knowledge | CrashLoopBackOff | 677 | App bug |

### Image Pull Failures

| Pod | Namespace | Cause |
|-----|-----------|-------|
| github-security-mcp-server | cortex-system | Missing image |
| auto-fix-daemon | cortex-system | Missing image |
| cortex-live-cli (x3) | cortex-system | Missing image |
| cortex-desktop-mcp | cortex | Missing image |
| docker-registry | cortex-chat | Missing image |
| postgres-backup | cortex-system | Missing image |
| postgres-exporter | cortex-system | Missing image |

### Stuck in ContainerCreating

| Pod | Namespace | Duration | Likely Cause |
|-----|-----------|----------|--------------|
| lifecycle-auditor (x10) | cortex-lifecycle | 2+ days | PVC or image issue |
| redis | cortex-chat | 114m | PVC issue |

---

## Agent Architecture (cortex-platform)

### Masters

| Master | ID | Capabilities | Status |
|--------|-----|--------------|--------|
| CoordinatorMaster | coordinator-master | task_routing, load_balancing, orchestration, system_monitoring | Deployed |
| SecurityMaster | security-master | security_operations, threat_management, vulnerability_scanning, incident_coordination | Deployed |

### Workers

| Worker | Capabilities | MCP Tools |
|--------|--------------|-----------|
| SandflyWorker | sandfly_api, threat_analysis, intrusion_detection, linux_security | sandfly_get_findings, sandfly_get_host_info, sandfly_list_hosts |
| GitHubSecurityWorker | github_security, dependabot_analysis, code_scanning, vulnerability_assessment | github_list_dependabot_alerts, github_list_code_scanning_alerts, github_get_alert_details |

### Message Flow

```
External Request
       │
       ▼
┌─────────────────────┐
│  CoordinatorMaster  │  Routes by task_type
└──────────┬──────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
┌────────────┐  [Future Division Masters]
│ Security   │
│ Master     │
└─────┬──────┘
      │
   ┌──┴───┐
   ▼      ▼
Sandfly  GitHub
Worker   Security
   │     Worker
   ▼      │
Claude   ▼
API    Claude
       API
```

---

## Layer Stack Architecture

### Concept

Each domain gets its own "Layer Stack":
- MoE Router → Qdrant → MCP Server(s) → Telemetry
- Layer Activator spins up stacks on-demand
- Scale to zero when idle (~32s cold start via KEDA)

### Existing Layer Stacks

| Stack | Namespace | Components | Status |
|-------|-----------|------------|--------|
| School Stack | cortex-school | MoE Router, Qdrant, YouTube workers | Progressing (workers crashing) |
| Security Stack | cortex-security | Falco, Trivy, Redis | Progressing |
| Knowledge Stack | cortex-knowledge | Graph API, Elasticsearch, Phoenix | Degraded |

### Planned Layer Stacks

| Stack | Purpose | MCP Servers |
|-------|---------|-------------|
| Infrastructure Stack | Proxmox, UniFi, Cloudflare | proxmox-mcp, unifi-mcp, cloudflare-mcp |
| Development Stack | GitHub, CI/CD | github-mcp, github-security-mcp |
| Observability Stack | Monitoring, Tracing | Phoenix, metrics collectors |

---

## GitOps Structure

**Repository**: cortex-gitops

```
cortex-gitops/
├── apps/                    # Application manifests by namespace
│   ├── cortex/
│   ├── cortex-chat/
│   ├── cortex-cicd/
│   ├── cortex-dev/
│   ├── cortex-knowledge/
│   ├── cortex-security/
│   └── cortex-system/
├── argocd-apps/            # ArgoCD Application definitions
│   ├── cortex-chat.yaml
│   ├── cortex-cicd.yaml
│   ├── cortex-core.yaml
│   └── ...
├── csaf/                   # CSAF-specific manifests
├── build/                  # Build configurations
└── docs/                   # Documentation
```

---

## Immediate Action Items

### Priority 1: Stop the Bleeding

1. **github-mcp-server** (709 restarts)
   - Check logs: `kubectl logs -n cortex-system github-mcp-server-xxx --previous`
   - Likely: API rate limiting, auth issue, or infinite loop

2. **Unknown state pods** (proxmox-mcp, n8n-mcp)
   - Check node status where they're scheduled
   - May need pod deletion to reschedule

3. **lifecycle-auditor stuck** (10 pods)
   - Check PVC status
   - May need CronJob suspension

### Priority 2: Fix Image Issues

1. Identify missing images:
   ```bash
   kubectl get pods -A | grep ImagePullBackOff | awk '{print $2}' | \
     xargs -I {} kubectl describe pod {} -n cortex-system | grep "Image:"
   ```

2. Either:
   - Push images to registry
   - Update deployments with correct image references
   - Delete deployments that are no longer needed

### Priority 3: Application Bugs

1. **docling-service** - Check Python dependencies
2. **csaf-registry** - Check database connectivity
3. **knowledge-graph-api** - Check MongoDB connection
4. **implementation-workers** - Check configuration

---

## Systems That Need Awareness

| System A | Should Know About | Current State |
|----------|-------------------|---------------|
| CoordinatorMaster | All division masters | Hardcoded security-master |
| SecurityMaster | All security workers | Spawns on-demand |
| Layer Activator | All Layer Stacks | Not implemented |
| Memory System | All sessions, infrastructure state | Not implemented |
| ArgoCD | All deployed resources | Configured |
| Phoenix | All Claude API calls | Partially configured |

---

## Next Steps

1. [ ] Fix github-mcp-server crash loop
2. [ ] Resolve Unknown state pods
3. [ ] Clean up stuck ContainerCreating pods
4. [ ] Fix ImagePullBackOff issues
5. [ ] Implement Layer Activator
6. [ ] Implement Memory System
7. [ ] Create system discovery mechanism

---

**Document Owner**: Cortex Operations
**Review Frequency**: Weekly or after major changes
**Last Verified**: 2026-01-20
