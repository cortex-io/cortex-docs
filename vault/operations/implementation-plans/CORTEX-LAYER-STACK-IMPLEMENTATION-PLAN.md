# Cortex Layer Stack Implementation Plan
**Version**: 1.0.0
**Date**: 2026-01-17
**Status**: READY FOR EXECUTION

---

## Executive Summary

**Goal**: Transform Cortex from centralized monolithic architecture to distributed, scale-to-zero layer-stack architecture where each MCP server gets its own isolated burst-capable stack that can learn and specialize.

**Current State**:
- 10 MCP servers operational (all with SSE endpoints)
- School pipeline deployed but blocked (youtube-ingestion has placeholder code)
- Cluster at 96-99% memory saturation
- Centralized MoE + Qdrant model not deployed (insufficient memory)

**Target State**:
- 10 layer-specific stacks (one per MCP domain)
- Each layer: MoE Router + Qdrant + MCP Server
- Scale-to-zero when idle (0 memory)
- Burst on demand (max 3.5Gi per layer)
- Autonomous distillation into specialized models
- School pipeline feeding layer improvements

**Timeline**: 5 phases, can be executed in parallel where noted

---

## Current Infrastructure Assessment

### What's Already Built and Running

#### 1. School Pipeline (cortex-school namespace) ✅ DEPLOYED
All 6 components running for 38 hours:

| Component | Status | Memory | Purpose |
|-----------|--------|--------|---------|
| school-coordinator | Running 1/1 | 256Mi req, 1Gi limit | Orchestrates improvement pipeline |
| moe-router | Running 1/1 | 128Mi req, 512Mi limit | Routes to 6 expert models |
| rag-validator | Running 1/1 | 256Mi req, 1Gi limit | Validates against infrastructure |
| implementation-workers | Running 1/1 | 128Mi req, 512Mi limit | Generates GitOps manifests |
| health-monitor | Running 1/1 | 256Mi req, 1Gi limit | Monitors deployments, rollbacks |
| qdrant | Running 1/1 | 128Mi req, 512Mi limit | Vector DB (20Gi emptyDir) |

**Total Memory**: ~1.1Gi requests, ~4.5Gi limits
**Location**: `/Users/ryandahlberg/Projects/cortex-gitops/apps/cortex-school/`
**Images**: All in local registry `10.43.170.72:5000/cortex-*`

#### 2. MCP Servers (cortex-system namespace) ✅ OPERATIONAL
All 10 servers with working SSE endpoints:

| Server | Domain | Status | Purpose |
|--------|--------|--------|---------|
| cloudflare-mcp | cloudflare-mcp.ry-ops.dev | Running | DNS, CDN management |
| cortex-mcp | cortex-mcp.ry-ops.dev | Running | Main orchestration |
| github-mcp | github-mcp.ry-ops.dev | Running | Git operations |
| github-security-mcp | github-security-mcp.ry-ops.dev | Running | Security scanning |
| kubernetes-mcp | kubernetes-mcp.ry-ops.dev | Running | K8s operations |
| langflow-chat-mcp | langflow-chat-mcp.ry-ops.dev | Running | Chat workflows |
| n8n-mcp | n8n-mcp.ry-ops.dev | Running | Automation |
| proxmox-mcp | proxmox-mcp.ry-ops.dev | Running | VM management |
| sandfly-mcp | sandfly-mcp.ry-ops.dev | Running | EDR/security |
| unifi-mcp | unifi-mcp.ry-ops.dev | Running | Network management |

#### 3. Layer Stack Helm Chart ✅ READY
**Location**: `/Users/ryandahlberg/Projects/cortex-layer-stack/`
**Status**: Complete Helm chart with:
- KEDA autoscaling templates
- MoE Router deployment
- Qdrant StatefulSet
- MCP Server deployment
- Distillation CronJob
- Example layer configs (infrastructure, security, networking)

**Missing**: Container images need to be built

### What's Broken/Missing

#### 1. YouTube Ingestion Service ❌ BROKEN
**Component**: youtube-ingestion (cortex namespace)
**Status**: CrashLoopBackOff (76 restarts)
**Problem**: ConfigMap `youtube-index-fixed` contains placeholder code
**Impact**: Blocks entire school pipeline (no improvements entering system)
**Fix Required**: Replace placeholder with actual youtube-ingestion code

#### 2. Centralized MoE + Qdrant ❌ NOT DEPLOYED
**Components**:
- cortex-knowledge/moe-router (scaled to 0)
- cortex-knowledge/qdrant (scaled to 0)
**Problem**: Insufficient cluster memory to run centralized services
**Solution**: Layer-stack architecture eliminates need for centralized version

#### 3. Layer Stack Images ❌ NOT BUILT
**Missing Images**:
- cortex-layer-moe-router
- cortex-layer-mcp-server (generic, tool-injected)
- cortex-distillation-pipeline

**Note**: Qdrant uses upstream image (qdrant/qdrant:v1.7.4)

---

## Phase 1: Fix Existing School Pipeline Infrastructure

**Goal**: Get the autonomous learning cycle operational with current centralized architecture
**Duration**: ~2-3 hours
**Blockers**: None
**Dependencies**: None

### Tasks

#### 1.1: Fix youtube-ingestion Service
**Priority**: CRITICAL - Blocks entire pipeline

**Steps**:
1. Locate actual youtube-ingestion source code in cortex-platform
2. Build proper container image with all dependencies
3. Push to local registry `10.43.170.72:5000/youtube-ingestion:latest`
4. Update deployment to remove ConfigMap volume mounts
5. Update manifest in cortex-gitops
6. Commit and push (ArgoCD auto-syncs)
7. Verify pod starts and health checks pass

**Acceptance Criteria**:
- youtube-ingestion pod Running 1/1
- Health endpoint responds 200 OK
- Redis queue `improvements:raw` starts receiving data
- Logs show successful transcript fetching

#### 1.2: Verify School Pipeline Flow
**Priority**: HIGH

**Steps**:
1. Monitor Redis queues:
   ```bash
   kubectl exec -n cortex redis-queue-0 -- redis-cli keys "improvements:*"
   ```
2. Check coordinator logs for improvement processing
3. Verify MoE router receives requests and calls Claude API
4. Confirm RAG validator queries Qdrant
5. Watch for implementation-worker Git commits
6. Monitor ArgoCD for auto-sync
7. Verify health-monitor detects deployments

**Acceptance Criteria**:
- At least 1 improvement flows through all 7 queues
- Git commit appears in cortex-gitops
- ArgoCD syncs the change
- Health monitor verifies deployment

#### 1.3: Test Auto-Approval and Rollback
**Priority**: MEDIUM

**Steps**:
1. Inject a low-quality improvement (relevance < 90%)
2. Verify it lands in `improvements:pending_review` (not approved)
3. Inject a high-quality improvement (relevance ≥ 90%)
4. Verify auto-approval to `improvements:approved`
5. Inject a broken manifest (intentional failure)
6. Verify health-monitor detects failure
7. Confirm Git revert is executed
8. Verify ArgoCD syncs rollback

**Acceptance Criteria**:
- Auto-approval works for quality improvements
- Manual review queue works for uncertain changes
- Rollback system detects failures and reverts

### Deliverables

- [ ] youtube-ingestion running and ingesting videos
- [ ] School pipeline processing improvements end-to-end
- [ ] Auto-approval functioning
- [ ] Rollback system tested and working
- [ ] Documentation updated with test results

---

## Phase 2: Build Layer Stack Images and Test KEDA Scaling

**Goal**: Create reusable container images for layer-stack components and prove scale-to-zero works
**Duration**: ~4-6 hours
**Blockers**: None (can run in parallel with Phase 1)
**Dependencies**: Docker, local registry access

### Tasks

#### 2.1: Install KEDA
**Priority**: HIGH

**Steps**:
1. Add KEDA Helm repo:
   ```bash
   helm repo add kedacore https://kedacore.github.io/charts
   helm repo update
   ```
2. Install KEDA in keda namespace:
   ```bash
   helm install keda kedacore/keda \
     --namespace keda \
     --create-namespace \
     --set resources.operator.limits.memory=256Mi \
     --set resources.operator.requests.memory=128Mi
   ```
3. Verify KEDA operator running:
   ```bash
   kubectl get pods -n keda
   ```
4. Create manifest in cortex-gitops:
   ```
   apps/keda/keda-installation.yaml
   ```

**Acceptance Criteria**:
- KEDA operator Running 1/1
- KEDA metrics server Running 1/1
- ScaledObject CRD available

#### 2.2: Build Layer-Stack MoE Router Image
**Priority**: HIGH

**Source Code Location**: TBD (need to create or locate in cortex-platform)

**Dockerfile** (`/tmp/layer-moe-router.Dockerfile`):
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
RUN pip install --no-cache-dir \
    anthropic==0.18.1 \
    fastapi==0.109.0 \
    uvicorn==0.27.0 \
    pydantic==2.6.0 \
    httpx==0.26.0 \
    redis==5.0.1

# Copy source code
COPY src/layer-moe-router/ /app/

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD python -c "import httpx; httpx.get('http://localhost:8080/health')"

# Expose HTTP port
EXPOSE 8080

# Run server
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**Features**:
- Routes to 6 expert models (Opus/Sonnet/Haiku)
- Telemetry capture for distillation
- Caching of routing decisions
- Prometheus metrics on /metrics

**Build**:
```bash
docker build -f /tmp/layer-moe-router.Dockerfile \
  -t 10.43.170.72:5000/cortex-layer-moe-router:latest \
  /Users/ryandahlberg/Projects/cortex-platform/services/moe-router/

docker push 10.43.170.72:5000/cortex-layer-moe-router:latest
```

**Acceptance Criteria**:
- Image builds successfully
- Starts and responds to /health
- Routes requests to Claude API
- Captures telemetry to Qdrant

#### 2.3: Build Layer-Stack MCP Server Image
**Priority**: HIGH

**Source Code**: Generic MCP server with tool injection

**Dockerfile** (`/tmp/layer-mcp-server.Dockerfile`):
```dockerfile
FROM node:20-slim

WORKDIR /app

# Install dependencies
RUN npm install --production \
    @modelcontextprotocol/sdk@1.0.0 \
    express@4.18.2 \
    sse-channel@4.2.0

# Copy generic MCP server
COPY src/layer-mcp-server/ /app/

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:8080/health')"

# Expose HTTP port
EXPOSE 8080

# Run server
CMD ["node", "server.js"]
```

**Features**:
- Tool auto-discovery from environment variable
- SSE transport
- Telemetry capture (invocations, outcomes)
- Prometheus metrics

**Build**:
```bash
docker build -f /tmp/layer-mcp-server.Dockerfile \
  -t 10.43.170.72:5000/cortex-layer-mcp-server:latest \
  /Users/ryandahlberg/Projects/cortex-platform/services/layer-mcp-server/

docker push 10.43.170.72:5000/cortex-layer-mcp-server:latest
```

**Acceptance Criteria**:
- Image builds successfully
- Accepts tool list via env var
- Serves SSE endpoint
- Captures telemetry

#### 2.4: Build Distillation Pipeline Image
**Priority**: MEDIUM (needed for Phase 5)

**Dockerfile** (`/tmp/distillation-pipeline.Dockerfile`):
```dockerfile
FROM python:3.11

WORKDIR /workspace

# Install training dependencies
RUN pip install --no-cache-dir \
    torch==2.1.2 \
    transformers==4.37.0 \
    peft==0.8.2 \
    datasets==2.16.1 \
    accelerate==0.26.1 \
    bitsandbytes==0.42.0 \
    qdrant-client==1.7.0

# Copy distillation code
COPY src/distillation-pipeline/ /app/

# Run pipeline
CMD ["python", "/app/distill.py"]
```

**Features**:
- Collects telemetry from Qdrant
- Quality filtering (confidence, success rate)
- LoRA adapter training
- Model publishing to registry

**Build**:
```bash
docker build -f /tmp/distillation-pipeline.Dockerfile \
  -t 10.43.170.72:5000/cortex-distillation-pipeline:latest \
  /Users/ryandahlberg/Projects/cortex-platform/services/distillation-pipeline/

docker push 10.43.170.72:5000/cortex-distillation-pipeline:latest
```

**Acceptance Criteria**:
- Image builds successfully
- Can connect to Qdrant
- Trains LoRA adapters
- Publishes to registry

#### 2.5: Test Scale-to-Zero with KEDA
**Priority**: HIGH

**Steps**:
1. Deploy a test layer stack:
   ```bash
   helm install test-layer ./cortex-layer-stack \
     -f examples/infrastructure-layer.yaml \
     -n cortex-test \
     --create-namespace
   ```
2. Verify initial deployment (should scale to 0):
   ```bash
   kubectl get pods -n cortex-test
   # Should show 0 pods after idleTimeout
   ```
3. Send HTTP request to trigger scale-up:
   ```bash
   curl https://test-layer-mcp.ry-ops.dev/health
   ```
4. Watch pods scale up:
   ```bash
   kubectl get pods -n cortex-test -w
   ```
5. Verify all 3 components running:
   - test-layer-moe-router
   - test-layer-qdrant
   - test-layer-mcp-server
6. Wait for idle timeout (300s default)
7. Verify scale-down to 0

**Acceptance Criteria**:
- Pods scale to 0 when idle
- HTTP request triggers scale-up within 10s
- All components burst successfully
- Pods scale back to 0 after idle timeout
- Cold start latency < 30s

### Deliverables

- [ ] KEDA installed and operational
- [ ] cortex-layer-moe-router:latest image built and tested
- [ ] cortex-layer-mcp-server:latest image built and tested
- [ ] cortex-distillation-pipeline:latest image built
- [ ] Scale-to-zero proven with test layer
- [ ] Cold start performance documented

---

## Phase 3: Migrate 10 MCP Servers to Layer-Stack Architecture

**Goal**: Convert each existing MCP server into its own isolated layer stack
**Duration**: ~6-8 hours (can parallelize)
**Blockers**: Phase 2 completion (need images)
**Dependencies**: KEDA, layer-stack images

### Architecture Mapping

Each MCP server becomes a layer with specialized configuration:

| Current MCP Server | Layer Name | Domain | Tools | Qdrant Size | Auto-Approve |
|-------------------|------------|--------|-------|-------------|--------------|
| cloudflare-mcp | cloudflare-layer | DNS/CDN | cloudflare_* | 10Gi | Yes (0.90) |
| github-mcp | github-layer | Git Ops | github_*, git_* | 15Gi | Yes (0.85) |
| github-security-mcp | security-layer | Security | trivy_*, falco_*, sandfly_* | 30Gi | No (manual) |
| kubernetes-mcp | kubernetes-layer | K8s Ops | kubectl_*, helm_* | 20Gi | Yes (0.90) |
| proxmox-mcp | infrastructure-layer | VM Mgmt | proxmox_* | 20Gi | Yes (0.88) |
| unifi-mcp | networking-layer | Network | unifi_*, dns_*, ping_* | 15Gi | Yes (0.90) |
| sandfly-mcp | edr-layer | Threat Detect | sandfly_* | 25Gi | No (manual) |
| n8n-mcp | automation-layer | Workflows | n8n_* | 10Gi | Yes (0.85) |
| langflow-chat-mcp | chat-layer | Conversations | langflow_* | 10Gi | Yes (0.85) |
| cortex-mcp | orchestration-layer | Main Coord | cortex_* | 25Gi | Yes (0.92) |

### Tasks

#### 3.1: Create Layer Value Files
**Priority**: HIGH

For each layer, create `values-{layer}.yaml`:

**Example: cloudflare-layer**
```yaml
# values-cloudflare-layer.yaml
layer:
  name: cloudflare
  description: "Cloudflare DNS and CDN management"
  domain: "cloudflare.cortex.local"

autoscaling:
  enabled: true
  idleTimeout: 300
  minReplicas: 0
  maxReplicas: 2

moeRouter:
  enabled: true
  routing:
    topK: 2
    strategy: "affinity"
  telemetry:
    enabled: true
    sampleRate: 1.0

qdrant:
  enabled: true
  storage:
    enabled: true
    size: "10Gi"

mcpServer:
  enabled: true
  tools:
    explicit:
      - cloudflare_list_zones
      - cloudflare_get_dns_records
      - cloudflare_update_dns
      - cloudflare_purge_cache

distillation:
  enabled: true
  collection:
    minSamples: 5000
    qualityThreshold: 0.90
  output:
    type: "lora"
    baseModel: "codellama/CodeLlama-7b-Instruct-hf"
```

**Location**: `/Users/ryandahlberg/Projects/cortex-gitops/layers/values-*.yaml`

**Deliverable**: 10 layer value files created

#### 3.2: Deploy Layer Stacks via Helm
**Priority**: HIGH

**Steps for each layer**:
1. Deploy using Helm:
   ```bash
   helm install cloudflare-layer ./cortex-layer-stack \
     -f layers/values-cloudflare-layer.yaml \
     -n cortex-cloudflare \
     --create-namespace
   ```
2. Verify KEDA ScaledObjects created:
   ```bash
   kubectl get scaledobjects -n cortex-cloudflare
   ```
3. Verify initial scale to zero
4. Test scale-up with HTTP request
5. Repeat for all 10 layers

**Deliverable**: 10 layer stacks deployed, all scaling correctly

#### 3.3: Create ArgoCD Applications for Layers
**Priority**: HIGH

**Template**: `argocd-apps/layer-template.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {layer-name}-layer
  namespace: argocd
spec:
  project: cortex
  source:
    repoURL: https://github.com/ry-ops/cortex-gitops.git
    targetRevision: main
    path: layers/{layer-name}
    helm:
      valueFiles:
        - values-{layer-name}-layer.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: cortex-{layer-name}
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Deliverable**: 10 ArgoCD Applications managing layer stacks

#### 3.4: Update Ingresses for Layer Endpoints
**Priority**: MEDIUM

Each layer needs Ingress for external access:

**Template**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {layer-name}-layer-ingress
  namespace: cortex-{layer-name}
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod-dns
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  rules:
  - host: {layer-name}.ry-ops.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {layer-name}-layer-mcp-server
            port:
              number: 8080
  tls:
  - hosts:
    - {layer-name}.ry-ops.dev
    secretName: {layer-name}-layer-tls
```

**Deliverable**: 10 Ingresses created, all endpoints accessible

#### 3.5: Deprecate Old MCP Server Deployments
**Priority**: LOW (after validation)

**Steps**:
1. Verify layer-stack versions working
2. Scale old deployments to 0:
   ```bash
   kubectl scale deployment cloudflare-mcp-server -n cortex-system --replicas=0
   ```
3. Monitor for issues (keep old deployments as backup)
4. After 1 week of stability, delete old manifests
5. Update ArgoCD to prune old resources

**Deliverable**: Old MCP servers deprecated, cluster memory freed

### Deliverables

- [ ] 10 layer value files created
- [ ] 10 layer stacks deployed and scaling
- [ ] 10 ArgoCD Applications managing layers
- [ ] 10 Ingresses configured
- [ ] Old MCP servers deprecated
- [ ] Memory usage reduced by ~3-4Gi

---

## Phase 4: Deploy School Pipeline as Infrastructure-Layer

**Goal**: Integrate the autonomous learning pipeline as the "infrastructure-layer" that manages cluster improvements
**Duration**: ~3-4 hours
**Blockers**: Phase 1 completion (need working school pipeline)
**Dependencies**: Phase 2 images, Phase 3 layer-stack proven

### Architecture

The school pipeline becomes a specialized layer:

```
┌─────────────────────────────────────────────────────────┐
│              Infrastructure Layer                        │
│  ┌────────────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │ MoE Router     │  │  Qdrant     │  │ MCP Server   │  │
│  │ (6 experts)    │  │  (25Gi)     │  │ (kubectl,    │  │
│  │                │  │             │  │  proxmox,    │  │
│  │                │  │             │  │  cloudflare) │  │
│  └────────────────┘  └─────────────┘  └──────────────┘  │
│           │                 │                 │          │
│           └─────────────────┴─────────────────┘          │
│                          │                               │
│                ┌─────────▼──────────┐                   │
│                │ School Coordinator │                    │
│                │ (orchestrates)     │                    │
│                └─────────┬──────────┘                   │
│                          │                               │
│        ┌─────────────────┼──────────────────┐           │
│        ▼                 ▼                   ▼           │
│  ┌───────────┐  ┌─────────────────┐  ┌──────────────┐  │
│  │ YouTube   │  │ Implementation  │  │ Health       │  │
│  │ Ingestion │  │ Workers         │  │ Monitor      │  │
│  └───────────┘  └─────────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Tasks

#### 4.1: Create infrastructure-layer Values
**Priority**: HIGH

**File**: `layers/values-infrastructure-layer.yaml`
```yaml
layer:
  name: infrastructure
  description: "Kubernetes, Proxmox, and cloud infrastructure operations - Cortex Online School"
  domain: "infra.cortex.local"

autoscaling:
  enabled: true
  idleTimeout: 600  # 10 minutes - infra tasks are sporadic
  minReplicas: 0
  maxReplicas: 3

moeRouter:
  enabled: true
  routing:
    topK: 2
    strategy: "affinity"
  # Use the existing school pipeline MoE Router
  experts:
    - name: architecture
      model: claude-opus-4-5
    - name: integration
      model: claude-sonnet-4-5
    - name: security
      model: claude-opus-4-5
    - name: database
      model: claude-sonnet-4-5
    - name: networking
      model: claude-sonnet-4-5
    - name: monitoring
      model: claude-haiku-4
  telemetry:
    enabled: true
    captureDecisions: true
    sampleRate: 1.0

qdrant:
  enabled: true
  storage:
    enabled: true
    size: "25Gi"  # Infrastructure docs, runbooks, past incidents
    snapshots:
      enabled: true
      schedule: "0 2 * * *"
      retention: 30
  collections:
    vectorSize: 768  # Larger for technical documentation

mcpServer:
  enabled: true
  tools:
    explicit:
      - kubectl_get
      - kubectl_apply
      - kubectl_delete
      - proxmox_list_vms
      - proxmox_vm_status
      - cloudflare_list_zones
      - cloudflare_get_dns_records
      - argocd_sync
      - argocd_get_app
    timeout: 60

# Distillation for infrastructure expertise
distillation:
  enabled: true
  collection:
    minSamples: 10000
    qualityThreshold: 0.90
  output:
    type: "lora"
    baseModel: "codellama/CodeLlama-13b-Instruct-hf"
    loraRank: 64  # Higher rank for complex patterns
  automation:
    autoTrigger: false
    schedule: "0 4 * * 0"  # Weekly Sunday 4AM

# Additional school pipeline components
schoolPipeline:
  enabled: true

  youtubeIngestion:
    enabled: true
    image: 10.43.170.72:5000/youtube-ingestion:latest
    schedule: "0 */1 * * *"  # Hourly

  coordinator:
    enabled: true
    image: 10.43.170.72:5000/cortex-school-coordinator:latest
    autoApprovalThreshold: 0.90

  implementationWorkers:
    enabled: true
    replicas: 2
    image: 10.43.170.72:5000/cortex-implementation-worker:latest

  healthMonitor:
    enabled: true
    image: 10.43.170.72:5000/cortex-health-monitor:latest
    healthCheckDuration: 300  # 5 minutes
```

**Deliverable**: Infrastructure layer values file created

#### 4.2: Migrate School Components to Infrastructure Layer
**Priority**: HIGH

**Steps**:
1. Update school component images to use layer-stack patterns
2. Add school-specific deployments to Helm chart:
   - `templates/school-coordinator.yaml`
   - `templates/youtube-ingestion-cronjob.yaml`
   - `templates/implementation-workers.yaml`
   - `templates/health-monitor.yaml`
3. Wire components to layer's MoE Router and Qdrant
4. Deploy infrastructure-layer:
   ```bash
   helm install infrastructure-layer ./cortex-layer-stack \
     -f layers/values-infrastructure-layer.yaml \
     -n cortex-infrastructure \
     --create-namespace
   ```
5. Verify all components start
6. Test improvement flow end-to-end

**Deliverable**: School pipeline running as infrastructure-layer

#### 4.3: Configure Redis Queues for Infrastructure Layer
**Priority**: HIGH

**Queue Structure**:
```
infra:improvements:raw
infra:improvements:categorized
infra:improvements:validated
infra:improvements:approved
infra:improvements:implemented
infra:improvements:deployed
infra:improvements:verified
```

**ConfigMap**: `infrastructure-layer-queues.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: infrastructure-layer-queues
  namespace: cortex-infrastructure
data:
  queue-prefix: "infra:improvements"
  auto-approval-threshold: "0.90"
  health-check-duration: "300"
```

**Deliverable**: Redis queues configured for infrastructure layer

#### 4.4: Create ArgoCD Application for Infrastructure Layer
**Priority**: HIGH

**File**: `argocd-apps/infrastructure-layer.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: infrastructure-layer
  namespace: argocd
spec:
  project: cortex
  source:
    repoURL: https://github.com/ry-ops/cortex-gitops.git
    targetRevision: main
    path: layers/infrastructure
    helm:
      valueFiles:
        - values-infrastructure-layer.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: cortex-infrastructure
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Deliverable**: ArgoCD managing infrastructure-layer

#### 4.5: Deprecate Old cortex-school Namespace
**Priority**: LOW

**Steps**:
1. Verify infrastructure-layer working for 1 week
2. Scale old school deployments to 0
3. Delete cortex-school ArgoCD Application
4. Delete cortex-school namespace
5. Remove manifests from cortex-gitops/apps/cortex-school/

**Deliverable**: cortex-school namespace removed, freed ~4.5Gi

### Deliverables

- [ ] infrastructure-layer values file created
- [ ] School components migrated to layer-stack
- [ ] Redis queues configured
- [ ] ArgoCD managing infrastructure-layer
- [ ] End-to-end improvement flow tested
- [ ] Old cortex-school namespace deprecated

---

## Phase 5: Enable Full Autonomous Learning Cycle

**Goal**: Activate complete autonomous cycle from YouTube videos to deployed improvements with distillation
**Duration**: ~2-3 hours
**Blockers**: All previous phases complete
**Dependencies**: Working pipeline, layer stacks operational

### Tasks

#### 5.1: Enable Distillation for All Layers
**Priority**: MEDIUM

**Steps**:
1. Update each layer's values to enable distillation:
   ```yaml
   distillation:
     enabled: true
     automation:
       autoTrigger: true  # Enable automatic distillation
       schedule: "0 4 * * 0"  # Weekly
   ```
2. Create S3 bucket or PVC for training data storage
3. Update distillation-pipeline image with layer-specific logic
4. Deploy distillation CronJobs for each layer

**Acceptance Criteria**:
- Distillation CronJobs created for 10 layers
- Training data collection working
- Weekly distillation runs successfully

#### 5.2: Configure YouTube Channel Subscriptions
**Priority**: HIGH

**Channels to Monitor**:
- Kubernetes tutorials
- Infrastructure as Code
- Security best practices
- Networking deep dives
- GitOps methodologies

**ConfigMap**: `youtube-channel-subscriptions.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: youtube-channel-subscriptions
  namespace: cortex-infrastructure
data:
  channels.json: |
    [
      {
        "id": "UCa6eh7gCkpPo5XXUDfygQQA",
        "name": "IBMTechnology",
        "category": "infrastructure",
        "priority": "high"
      },
      {
        "id": "UCdngmbVKX1Tgre699-XLlUA",
        "name": "NetworkChuck",
        "category": "networking",
        "priority": "high"
      }
    ]
```

**Deliverable**: Channel subscriptions configured

#### 5.3: Test Complete Autonomous Cycle
**Priority**: HIGH

**End-to-End Flow**:
1. YouTube ingestion runs (hourly CronJob)
2. Fetches new video transcripts
3. Claude analyzes and generates 25 improvements
4. Improvements pushed to `infra:improvements:raw`
5. School coordinator picks up improvements
6. MoE Router routes to appropriate expert
7. Expert evaluates (feasibility, impact, priority)
8. RAG Validator checks for conflicts/duplicates
9. Auto-approval gate (if relevance ≥ 90%)
10. Implementation worker generates manifest
11. Git commit to cortex-gitops
12. ArgoCD syncs within 3 minutes
13. Health monitor watches for 5 minutes
14. If healthy: marked verified ✅
15. If unhealthy: Git revert rollback

**Test**:
```bash
# Trigger manual ingestion
kubectl create job --from=cronjob/youtube-ingestion \
  -n cortex-infrastructure manual-test-1

# Monitor queues
watch -n 5 'kubectl exec -n cortex redis-queue-0 -- redis-cli --scan --pattern "infra:*"'

# Watch for Git commits
watch -n 10 'git log --oneline -5'

# Monitor ArgoCD
kubectl get applications -n argocd -w
```

**Acceptance Criteria**:
- At least 3 improvements flow through entire cycle
- Git commits appear in cortex-gitops
- ArgoCD syncs automatically
- Deployments verified healthy
- Rollback tested with intentional failure

#### 5.4: Enable Telemetry Collection
**Priority**: MEDIUM

**Components**:
- MoE Router → Qdrant (routing decisions)
- MCP Server → Qdrant (tool invocations)
- RAG Validator → Qdrant (query patterns)

**Verification**:
```bash
# Check Qdrant collections
kubectl exec -n cortex-infrastructure qdrant-0 -- \
  curl -s http://localhost:6333/collections

# Expected collections:
# - infrastructure_routing
# - infrastructure_tools
# - infrastructure_queries
```

**Acceptance Criteria**:
- Telemetry flowing to Qdrant
- Collections growing with operational data
- Quality filters working (min confidence)

#### 5.5: Monitor Resource Usage and Scaling
**Priority**: HIGH

**Metrics to Track**:
- Cluster memory usage (should be < 80%)
- Layer scale-to-zero frequency
- Cold start latency
- Burst capacity usage
- Distillation job completion rate

**Dashboard**: Create Grafana dashboard
```yaml
# grafana-dashboard-layers.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cortex-layers-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "title": "Cortex Layer Stacks",
      "panels": [
        {
          "title": "Layer Pod Count",
          "query": "sum(kube_deployment_status_replicas{namespace=~\"cortex-.*\"}) by (namespace)"
        },
        {
          "title": "Memory Usage by Layer",
          "query": "sum(container_memory_usage_bytes{namespace=~\"cortex-.*\"}) by (namespace)"
        },
        {
          "title": "Scale Events",
          "query": "rate(keda_scaler_activity[5m])"
        }
      ]
    }
```

**Acceptance Criteria**:
- Memory usage < 80%
- Layers scaling to 0 when idle
- Burst capacity working
- Metrics visible in Grafana

### Deliverables

- [ ] Distillation enabled for all layers
- [ ] YouTube channels configured
- [ ] Complete autonomous cycle tested
- [ ] Telemetry collection verified
- [ ] Resource monitoring dashboard created
- [ ] System running autonomously for 24 hours

---

## Success Criteria

### Technical Metrics

**Memory Efficiency**:
- Cluster memory usage < 80% (down from 96-99%)
- Idle layers consuming 0 memory
- Active layers bursting to max 3.5Gi each

**Scaling Performance**:
- Cold start latency < 30s
- Scale-to-zero after idle timeout (5-10 min)
- Burst capacity supports 3 concurrent layers active

**Pipeline Throughput**:
- YouTube ingestion: 10+ videos/hour
- Improvement generation: 250+ improvements/hour
- Auto-approval rate: 60-70% (high quality)
- Deployment rate: 5-10 improvements/day
- Rollback rate: < 5% (quality filtering works)

**Distillation Progress**:
- Telemetry collection: 1000+ samples/week per layer
- Training eligibility: 1-2 layers after 4 weeks
- Model graduation: 1 layer every 2 months

### Operational Metrics

**Reliability**:
- School pipeline uptime: > 99%
- Layer availability: > 95% (accounting for scale-to-zero)
- Rollback success rate: 100%
- Data loss: 0 (Redis persistence, Qdrant snapshots)

**Autonomy**:
- Manual interventions: < 5/week
- Auto-approval working: > 90% accuracy
- False positive rate: < 10%
- Pipeline self-healing: Working

**Cost Efficiency**:
- Compute usage: -40% (scale-to-zero savings)
- Storage usage: Stable (Qdrant snapshots)
- API costs: Tracked per layer

---

## Risk Management

### Critical Risks

#### 1. Memory Saturation During Burst
**Risk**: All 10 layers burst simultaneously, exceeding cluster capacity
**Probability**: Medium
**Impact**: High
**Mitigation**:
- KEDA priority classes (infrastructure > others)
- Layer quotas (max 2 concurrent bursts)
- Cluster expansion (+1 worker node on standby)
- Prometheus alerts at 85% memory

#### 2. YouTube Ingestion API Rate Limits
**Risk**: YouTube API blocks requests, starving pipeline
**Probability**: Low
**Impact**: High
**Mitigation**:
- Respect API quotas (10,000 units/day)
- Implement exponential backoff
- Cache transcripts (don't re-fetch)
- Rotate API keys if needed

#### 3. Distillation Job OOM Kills
**Risk**: Training jobs exceed memory limits
**Probability**: Medium
**Impact**: Medium
**Mitigation**:
- Job resource limits: 16Gi max
- Use LoRA adapters (not full fine-tuning)
- Batch size tuning
- Fail gracefully, retry with smaller batch

#### 4. ArgoCD Sync Conflicts
**Risk**: Multiple layers committing simultaneously causes merge conflicts
**Probability**: Low
**Impact**: Medium
**Mitigation**:
- Layer-specific directories in cortex-gitops
- Git commit queuing (one at a time)
- Conflict detection in implementation-worker
- Automatic rebase on conflict

#### 5. Rollback Cascade
**Risk**: Rollback of improvement A breaks improvement B
**Probability**: Low
**Impact**: High
**Mitigation**:
- Health monitor checks dependencies
- Rollback in reverse order
- Test rollbacks in staging namespace
- Manual approval for risky changes

### Medium Risks

#### 6. Cold Start Latency
**Risk**: 30s cold start unacceptable for urgent requests
**Probability**: Medium
**Impact**: Low
**Mitigation**:
- Pre-warming for critical layers
- Reduce idle timeout (3 min instead of 10 min)
- Cache container images on all nodes
- Optimize image size

#### 7. Qdrant Data Loss
**Risk**: emptyDir volumes lost on pod restart
**Probability**: Low
**Impact**: Medium
**Mitigation**:
- Use PVCs instead of emptyDir
- Daily Qdrant snapshots
- Backup to S3
- Collection export before major changes

---

## Rollback Plan

If layer-stack architecture causes issues:

### Phase 1 Rollback
**Issue**: youtube-ingestion still broken
**Action**: Revert to placeholder, manual improvements testing

### Phase 2 Rollback
**Issue**: KEDA not working
**Action**: Uninstall KEDA, keep static deployments

### Phase 3 Rollback
**Issue**: Layer stacks unstable
**Action**: Re-enable old MCP server deployments in cortex-system

### Phase 4 Rollback
**Issue**: Infrastructure layer not working
**Action**: Restore cortex-school namespace, revert ArgoCD

### Phase 5 Rollback
**Issue**: Autonomous cycle causing issues
**Action**: Disable auto-approval, manual review only

---

## Timeline Estimate

### Sequential Execution (Conservative)
- Phase 1: 3 hours
- Phase 2: 6 hours (blocked on Phase 1)
- Phase 3: 8 hours (blocked on Phase 2)
- Phase 4: 4 hours (blocked on Phase 1 & 3)
- Phase 5: 3 hours (blocked on all)
**Total**: ~24 hours (3 days @ 8hrs/day)

### Parallel Execution (Aggressive)
- Day 1: Phase 1 + Phase 2 (parallel) = 6 hours
- Day 2: Phase 3 + Phase 4 (partial parallel) = 10 hours
- Day 3: Phase 5 = 3 hours
**Total**: ~19 hours (2.5 days)

### Recommended Approach
- Week 1: Phase 1 + 2 (stabilize, build images)
- Week 2: Phase 3 (migrate MCP servers)
- Week 3: Phase 4 (infrastructure layer)
- Week 4: Phase 5 (autonomous cycle)
**Total**: 4 weeks with testing and validation

---

## Next Steps

**Immediate Actions**:
1. Get approval for this plan
2. Fix youtube-ingestion (Phase 1.1)
3. Install KEDA (Phase 2.1)
4. Start building layer images (Phase 2.2-2.4)

**Questions to Resolve**:
1. Where is youtube-ingestion source code in cortex-platform?
2. Where is moe-router source code for layer-stack version?
3. Do we have S3 or PVC for distillation training data?
4. What's the priority order for layer migration (Phase 3)?

---

**Status**: READY FOR EXECUTION
**Author**: Claude Sonnet 4.5 + Human
**Date**: 2026-01-17
**Version**: 1.0.0
