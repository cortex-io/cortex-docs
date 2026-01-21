---
title: "Layer Activator Design"
type: architecture
created: "2026-01-20"
updated: "2026-01-20"
tags:
  - layer-stack
  - serverless
  - keda
  - scale-to-zero
  - mcp
---

# Layer Activator Design

## Overview

The Layer Activator is the "special sauce" that enables Cortex to run 200+ pods and scale to 1000+ workers efficiently by:

1. **Activating Layer Stacks on-demand** when requests arrive
2. **Scaling to zero** when stacks are idle
3. **Managing cold start** with ~32s target latency
4. **Routing requests** to the correct stack

```
                        ┌─────────────────────────┐
                        │    Incoming Request     │
                        └───────────┬─────────────┘
                                    │
                                    ▼
                        ┌─────────────────────────┐
                        │    Layer Activator      │
                        │    (Always Running)     │
                        └───────────┬─────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
              ▼                     ▼                     ▼
    ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
    │ Security Stack  │   │ Knowledge Stack │   │ Dev Stack       │
    │ (Scale 0→1→0)   │   │ (Scale 0→1→0)   │   │ (Scale 0→1→0)   │
    └─────────────────┘   └─────────────────┘   └─────────────────┘
```

---

## Layer Stack Definition

Each Layer Stack contains:

```
┌─────────────────────────────────────────────────────┐
│                    Layer Stack                       │
│                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │ MoE Router  │→│   Qdrant    │→│ MCP Server  │ │
│  │ (Master)    │  │ (Vector DB) │  │ (Tool API)  │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│         │                                    │      │
│         ▼                                    ▼      │
│  ┌─────────────┐                     ┌───────────┐ │
│  │  Workers    │                     │ Telemetry │ │
│  │ (Claude API)│                     │ (Phoenix) │ │
│  └─────────────┘                     └───────────┘ │
└─────────────────────────────────────────────────────┘
```

### Stack Components

| Component | Purpose | Scale Behavior |
|-----------|---------|----------------|
| MoE Router | Route tasks to workers | Scales with stack |
| Qdrant | Vector storage for RAG | Persistent or scale-to-zero |
| MCP Server(s) | Domain-specific tools | Scales with stack |
| Workers | Claude API execution | Scales 0-N |
| Telemetry | Observability (Phoenix) | Shared or per-stack |

---

## Layer Activator Architecture

### Core Components

```python
class LayerActivator:
    """
    Central proxy that activates Layer Stacks on-demand.

    Always running. Receives all requests and:
    1. Determines target stack
    2. Activates stack if scaled to zero
    3. Forwards request
    4. Tracks activity for scale-down
    """

    def __init__(self):
        self.stack_registry: Dict[str, StackInfo]
        self.activity_tracker: Dict[str, datetime]
        self.keda_client: KEDAClient
        self.redis: Redis  # For queue-based activation

    async def route_request(self, request: Request) -> Response:
        """Route incoming request to appropriate stack."""
        stack_id = self.determine_stack(request)

        # Activate if needed
        if not await self.is_stack_active(stack_id):
            await self.activate_stack(stack_id)
            await self.wait_for_ready(stack_id)

        # Track activity
        self.activity_tracker[stack_id] = datetime.now()

        # Forward request
        return await self.forward_to_stack(stack_id, request)
```

### Stack Registry

```python
@dataclass
class StackInfo:
    stack_id: str
    name: str
    namespace: str
    capabilities: List[str]
    components: List[str]  # Deployment names

    # Scaling config
    min_replicas: int = 0
    max_replicas: int = 5
    scale_up_cooldown: int = 0  # seconds
    scale_down_cooldown: int = 300  # 5 minutes

    # Routing config
    route_patterns: List[str]  # Task types this stack handles
    priority: int = 0  # Higher = preferred when multiple match

    # State
    current_replicas: int = 0
    last_activity: datetime = None
    status: StackStatus = StackStatus.SCALED_DOWN
```

---

## Activation Flow

### 1. Request Arrives

```
Request: { task_type: "scan_host", target: "web-server-01" }
           │
           ▼
    Layer Activator receives request
           │
           ▼
    Determine stack: "scan_host" matches Security Stack
           │
           ▼
    Check stack status
```

### 2. Stack Activation (if scaled to zero)

```
Stack is at 0 replicas
           │
           ▼
    Trigger KEDA ScaledObject
           │
           ├── Scale MoE Router: 0 → 1
           ├── Scale MCP Server: 0 → 1
           └── Scale Worker Pool: 0 → 1
           │
           ▼
    Wait for Ready (poll health endpoints)
           │
           ▼
    ~32 seconds later: Stack ready
```

### 3. Request Forwarding

```
Stack is ready
           │
           ▼
    Forward request to MoE Router
           │
           ▼
    MoE Router → Worker → Claude API → MCP Tools
           │
           ▼
    Response returns through chain
           │
           ▼
    Update activity timestamp
```

### 4. Scale Down (after idle period)

```
No activity for 5 minutes
           │
           ▼
    Layer Activator triggers scale-down
           │
           ├── Scale Workers: N → 0
           ├── Scale MCP Server: 1 → 0
           └── Scale MoE Router: 1 → 0
           │
           ▼
    Stack at 0 replicas (no cost)
```

---

## KEDA Integration

### ScaledObject per Stack

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: security-stack-scaler
  namespace: cortex-security
spec:
  scaleTargetRef:
    name: security-moe-router
  minReplicaCount: 0
  maxReplicaCount: 5
  cooldownPeriod: 300
  triggers:
  - type: redis
    metadata:
      address: redis.cortex-system.svc.cluster.local:6379
      listName: security-stack-queue
      listLength: "1"
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: security_stack_active_requests
      threshold: "1"
```

### Queue-Based Activation

```python
async def activate_via_queue(self, stack_id: str):
    """Activate stack by pushing to its Redis queue."""
    queue_name = f"{stack_id}-queue"

    # Push activation signal
    await self.redis.lpush(queue_name, "activate")

    # KEDA sees queue length > 0, scales up
    # Stack processes "activate" message and discards
```

### Metrics-Based Activation

```python
async def activate_via_metrics(self, stack_id: str):
    """Activate stack by incrementing Prometheus metric."""
    # Increment active_requests metric
    self.metrics.inc(f"{stack_id}_active_requests")

    # KEDA sees metric > threshold, scales up
```

---

## Stack Definitions

### Security Stack

```yaml
stack_id: security-stack
name: Security Operations
namespace: cortex-security
capabilities:
  - sandfly_api
  - threat_analysis
  - vulnerability_scanning
  - github_security
components:
  - security-master
  - sandfly-worker
  - github-security-worker
  - sandfly-mcp-server
  - github-security-mcp-server
route_patterns:
  - scan_host
  - analyze_threat
  - scan_repository
  - check_vulnerabilities
scale_down_cooldown: 300
```

### Knowledge Stack

```yaml
stack_id: knowledge-stack
name: Knowledge Operations
namespace: cortex-knowledge
capabilities:
  - knowledge_extraction
  - rag_queries
  - documentation
components:
  - knowledge-master
  - knowledge-worker
  - qdrant
  - mcp-server
route_patterns:
  - extract_knowledge
  - query_docs
  - search_codebase
scale_down_cooldown: 600  # Longer due to Qdrant startup
```

### School Stack (Always On)

```yaml
stack_id: school-stack
name: Cortex Online School
namespace: cortex-school
capabilities:
  - youtube_learning
  - knowledge_ingestion
components:
  - school-master
  - youtube-worker
  - implementation-worker
  - qdrant
min_replicas: 1  # Never scale to zero
scale_down_cooldown: 0
```

### Infrastructure Stack

```yaml
stack_id: infra-stack
name: Infrastructure Operations
namespace: cortex-infra
capabilities:
  - proxmox_api
  - unifi_api
  - cloudflare_api
  - kubernetes_api
components:
  - infra-master
  - proxmox-mcp-server
  - unifi-mcp-server
  - cloudflare-mcp-server
  - kubernetes-mcp-server
route_patterns:
  - manage_vm
  - network_config
  - dns_update
  - cluster_operation
scale_down_cooldown: 300
```

---

## Layer Activator API

### Endpoints

```
POST /activate
  Request: { stack_id: "security-stack" }
  Response: { status: "activating", estimated_ready: 32 }

GET /stacks
  Response: [
    { stack_id: "security-stack", status: "scaled_down", replicas: 0 },
    { stack_id: "school-stack", status: "active", replicas: 1 },
    ...
  ]

GET /stacks/{stack_id}/status
  Response: {
    stack_id: "security-stack",
    status: "active",
    replicas: 2,
    last_activity: "2026-01-20T10:30:00Z",
    components: [
      { name: "security-master", ready: true },
      { name: "sandfly-mcp-server", ready: true }
    ]
  }

POST /route
  Request: { task_type: "scan_host", payload: {...} }
  Response: { stack_id: "security-stack", response: {...} }
```

### Health Endpoint

```
GET /health
  Response: {
    status: "healthy",
    active_stacks: 2,
    total_stacks: 5,
    pending_activations: 0
  }
```

---

## Implementation Plan

### Phase 1: Core Activator

1. Create `LayerActivator` service
2. Implement stack registry
3. Implement basic activation (kubectl scale)
4. Implement request routing
5. Deploy to `cortex-system`

### Phase 2: KEDA Integration

1. Create ScaledObject for each stack
2. Implement queue-based activation
3. Implement metrics-based activation
4. Test cold start latency

### Phase 3: Stack Templates

1. Create Helm chart for Layer Stack
2. Define stack configurations in Git
3. ArgoCD ApplicationSet for stacks
4. Automated stack deployment

### Phase 4: Observability

1. Add Phoenix tracing to activator
2. Stack activation metrics
3. Cold start latency tracking
4. Cost attribution per stack

---

## Kubernetes Manifests

### Layer Activator Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: layer-activator
  namespace: cortex-system
spec:
  replicas: 2  # HA - always running
  selector:
    matchLabels:
      app: layer-activator
  template:
    metadata:
      labels:
        app: layer-activator
    spec:
      serviceAccountName: layer-activator
      containers:
      - name: activator
        image: cortex/layer-activator:latest
        ports:
        - containerPort: 8080
        env:
        - name: REDIS_URL
          value: redis://redis.cortex-system.svc:6379
        - name: STACK_CONFIG
          value: /config/stacks.yaml
        volumeMounts:
        - name: config
          mountPath: /config
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
      volumes:
      - name: config
        configMap:
          name: layer-activator-config
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: layer-activator
  namespace: cortex-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: layer-activator
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "deployments/scale"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["keda.sh"]
  resources: ["scaledobjects"]
  verbs: ["get", "list", "watch", "create", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: layer-activator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: layer-activator
subjects:
- kind: ServiceAccount
  name: layer-activator
  namespace: cortex-system
```

### Stack Configuration

```yaml
# configmap: layer-activator-config
apiVersion: v1
kind: ConfigMap
metadata:
  name: layer-activator-config
  namespace: cortex-system
data:
  stacks.yaml: |
    stacks:
      - id: security-stack
        namespace: cortex-security
        min_replicas: 0
        max_replicas: 5
        cooldown: 300
        components:
          - deployment: security-master
          - deployment: sandfly-mcp-server
          - deployment: github-security-mcp-server
        routes:
          - pattern: "security.*"
          - pattern: "scan_.*"
          - pattern: "threat.*"

      - id: school-stack
        namespace: cortex-school
        min_replicas: 1  # Always on
        components:
          - deployment: school-master
          - deployment: youtube-worker
        routes:
          - pattern: "learn.*"
          - pattern: "youtube.*"

      - id: infra-stack
        namespace: cortex-infra
        min_replicas: 0
        cooldown: 300
        components:
          - deployment: proxmox-mcp-server
          - deployment: unifi-mcp-server
          - deployment: cloudflare-mcp-server
          - deployment: kubernetes-mcp-server
        routes:
          - pattern: "infra.*"
          - pattern: "vm.*"
          - pattern: "network.*"
```

---

## Metrics

### Key Metrics to Track

| Metric | Description |
|--------|-------------|
| `layer_activator_activations_total` | Total stack activations |
| `layer_activator_activation_duration_seconds` | Time to activate stack |
| `layer_activator_active_stacks` | Currently active stacks |
| `layer_activator_requests_total` | Total requests routed |
| `layer_activator_cold_starts_total` | Requests that triggered cold start |
| `stack_idle_seconds` | Time since last activity per stack |
| `stack_replicas` | Current replicas per stack |

---

## Related Documentation

- [[agent-architecture]] - Master/Worker agent details
- [[keda-configuration]] - KEDA setup and tuning
- [[memory-system]] - Session continuity across activations
- [[stack-templates]] - Helm charts for Layer Stacks
