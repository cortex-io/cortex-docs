---
title: "Memory System Design"
type: architecture
created: "2026-01-20"
updated: "2026-01-20"
tags:
  - memory
  - session-continuity
  - debugging
  - infrastructure-state
  - timeline
---

# Cortex Memory System Design

## Overview

The Memory System enables Cortex to maintain continuity across sessions, debug historical issues, and maintain infrastructure visibility. It addresses three core pain points:

1. **Session Continuity** - Pick up where we left off
2. **Debugging Historical Issues** - Find what broke and when
3. **Infrastructure Visibility** - Know what's running and its state

```
┌─────────────────────────────────────────────────────────────┐
│                     Memory System                            │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐│
│  │ Session Memory  │  │ Infrastructure  │  │  Historical  ││
│  │ (Short-term)    │  │ State (Live)    │  │  Timeline    ││
│  └────────┬────────┘  └────────┬────────┘  └──────┬───────┘│
│           │                    │                   │        │
│           └────────────────────┼───────────────────┘        │
│                                │                            │
│                    ┌───────────▼───────────┐                │
│                    │    Memory Service     │                │
│                    │    (Query Interface)  │                │
│                    └───────────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

---

## Three Memory Layers

### Layer 1: Session Memory (Short-term)

**Purpose:** Remember what we were working on, decisions made, context gathered.

**Retention:** Current session + recent sessions (7 days)

**Contents:**
- Current task context
- Decisions made and why
- Files read/modified
- Commands executed
- Conversation summaries

```python
@dataclass
class SessionMemory:
    session_id: str
    started_at: datetime
    ended_at: Optional[datetime]

    # Context
    working_directory: str
    active_files: List[str]
    current_task: Optional[str]

    # History
    decisions: List[Decision]
    actions: List[Action]
    conversation_summary: str

    # State
    todo_list: List[TodoItem]
    blockers: List[str]
    next_steps: List[str]
```

**Storage:** Redis (hot) + PostgreSQL (warm)

### Layer 2: Infrastructure State (Live)

**Purpose:** Know what's deployed, what's healthy, what's broken.

**Retention:** Always current (continuously updated)

**Contents:**
- Cluster topology
- Pod/deployment status
- MCP server inventory
- Service health
- Resource utilization

```python
@dataclass
class InfrastructureState:
    timestamp: datetime

    # Cluster
    nodes: List[NodeInfo]
    namespaces: List[NamespaceInfo]

    # Workloads
    deployments: List[DeploymentInfo]
    pods: List[PodInfo]
    services: List[ServiceInfo]

    # Cortex-specific
    mcp_servers: List[MCPServerInfo]
    layer_stacks: List[LayerStackInfo]
    agents: List[AgentInfo]

    # Health
    alerts: List[Alert]
    degraded_components: List[str]
```

**Storage:** Redis (current state) + InfluxDB/TimescaleDB (time series)

### Layer 3: Historical Timeline (Long-term)

**Purpose:** Track changes over time, debug what broke when.

**Retention:** 90 days detailed, 1 year summarized

**Contents:**
- Git commits and their effects
- Deployment events
- Incidents and resolutions
- Configuration changes
- Scaling events

```python
@dataclass
class TimelineEvent:
    event_id: str
    timestamp: datetime
    event_type: EventType
    source: str  # git, argocd, k8s, cortex, etc.

    # Event data
    description: str
    details: Dict[str, Any]

    # Relationships
    caused_by: Optional[str]  # Previous event ID
    caused: List[str]  # Subsequent event IDs

    # Impact
    affected_components: List[str]
    severity: Severity
```

**Storage:** PostgreSQL (queryable) + S3 (archive)

---

## Memory Service API

### Session Memory Endpoints

```
POST /memory/sessions
  Create new session
  Response: { session_id: "sess-abc123", started_at: "..." }

PUT /memory/sessions/{session_id}
  Update session context
  Request: { current_task: "Fix MCP servers", todo_list: [...] }

GET /memory/sessions/current
  Get current or most recent session
  Response: { session_id: "...", context: {...}, decisions: [...] }

GET /memory/sessions/{session_id}/summary
  Get session summary for handoff
  Response: { summary: "Working on Layer Activator...", key_decisions: [...] }

POST /memory/sessions/{session_id}/decision
  Record a decision
  Request: { decision: "Use init container", rationale: "...", alternatives: [...] }
```

### Infrastructure State Endpoints

```
GET /memory/infrastructure/current
  Get current infrastructure state
  Response: { nodes: [...], pods: [...], mcp_servers: [...] }

GET /memory/infrastructure/component/{name}
  Get specific component state
  Response: { name: "github-mcp-server", status: "Running", restarts: 0 }

GET /memory/infrastructure/diff
  Compare current state to previous
  Query: ?since=2026-01-20T00:00:00Z
  Response: { added: [...], removed: [...], changed: [...] }

GET /memory/infrastructure/health
  Get health summary
  Response: { healthy: 127, degraded: 15, critical: 5 }
```

### Timeline Endpoints

```
GET /memory/timeline
  Query timeline events
  Query: ?start=2026-01-19&end=2026-01-20&type=deployment
  Response: { events: [...] }

GET /memory/timeline/event/{event_id}
  Get specific event with relationships
  Response: { event: {...}, caused_by: {...}, caused: [...] }

GET /memory/timeline/component/{name}
  Get event history for component
  Response: { component: "github-mcp-server", events: [...] }

POST /memory/timeline/correlate
  Find related events
  Request: { symptom: "github-mcp crashing", timeframe: "24h" }
  Response: { probable_cause: {...}, related_events: [...] }
```

---

## Session Continuity Flow

### Starting a New Session

```
1. Claude Code starts
           │
           ▼
2. Memory Service: GET /sessions/current
           │
           ├── If recent session exists (< 24h)
           │   └── Return session summary for handoff
           │
           └── If no recent session
               └── Create new session
           │
           ▼
3. Load context into conversation
   - Previous task status
   - Key decisions made
   - Current blockers
   - Next steps
           │
           ▼
4. "Here's where we left off..."
```

### During a Session

```
User: "Fix the MCP servers"
           │
           ▼
Claude: Records decision to investigate
        POST /sessions/{id}/decision
           │
           ▼
Claude: Reads infrastructure state
        GET /infrastructure/current
           │
           ▼
Claude: Identifies issues, takes actions
        Records each action
        PUT /sessions/{id} (update context)
           │
           ▼
Claude: Commits fix, pushes to Git
        Timeline event auto-recorded
```

### Session Handoff

```
1. Session ends (context limit, user stops)
           │
           ▼
2. Generate session summary
   - What was accomplished
   - What's still pending
   - Key decisions and rationale
   - Blockers identified
           │
           ▼
3. Store summary
   POST /sessions/{id}/summary
           │
           ▼
4. Next session starts
   GET /sessions/current
           │
           ▼
5. Load summary into new context
   "Continuing from previous session..."
```

---

## Infrastructure State Collection

### Collectors

```python
class InfrastructureCollector:
    """Continuously collect infrastructure state."""

    async def collect_kubernetes(self):
        """Collect K8s state every 30s."""
        nodes = await self.k8s.list_nodes()
        pods = await self.k8s.list_pods(all_namespaces=True)
        deployments = await self.k8s.list_deployments(all_namespaces=True)

        # Enrich with Cortex-specific info
        mcp_servers = self.identify_mcp_servers(pods)
        layer_stacks = self.identify_layer_stacks(deployments)

        return InfrastructureState(
            timestamp=datetime.now(),
            nodes=nodes,
            pods=pods,
            mcp_servers=mcp_servers,
            layer_stacks=layer_stacks,
        )

    async def collect_argocd(self):
        """Collect ArgoCD application state."""
        apps = await self.argocd.list_applications()
        return [AppState(
            name=app.name,
            sync_status=app.status.sync.status,
            health_status=app.status.health.status,
        ) for app in apps]

    async def collect_metrics(self):
        """Collect resource metrics."""
        return await self.metrics_server.get_node_metrics()
```

### Change Detection

```python
class ChangeDetector:
    """Detect and record infrastructure changes."""

    def __init__(self, memory_service: MemoryService):
        self.memory = memory_service
        self.previous_state: Optional[InfrastructureState] = None

    async def detect_changes(self, current: InfrastructureState):
        if self.previous_state is None:
            self.previous_state = current
            return

        changes = self.diff(self.previous_state, current)

        for change in changes:
            await self.memory.record_timeline_event(
                event_type=EventType.INFRASTRUCTURE_CHANGE,
                source="collector",
                description=change.description,
                details=change.details,
                affected_components=change.components,
            )

        self.previous_state = current
```

---

## Timeline Event Sources

### Automatic Event Sources

| Source | Events Captured |
|--------|-----------------|
| Git/GitHub | Commits, PRs, merges |
| ArgoCD | Sync events, health changes |
| Kubernetes | Pod lifecycle, scaling, failures |
| KEDA | Scale events (up/down) |
| Layer Activator | Stack activations |
| Cortex Agents | Task completions, errors |
| Phoenix | Trace completions, anomalies |

### Event Correlation

```python
class EventCorrelator:
    """Correlate events to find root causes."""

    async def find_cause(
        self,
        symptom: str,
        timeframe: timedelta,
    ) -> Optional[TimelineEvent]:
        """Find probable cause for a symptom."""

        # Get recent events
        events = await self.memory.get_timeline(
            end=datetime.now(),
            start=datetime.now() - timeframe,
        )

        # Score events by relevance
        scored = []
        for event in events:
            score = self.score_relevance(event, symptom)
            if score > 0:
                scored.append((score, event))

        # Return highest scoring
        if scored:
            scored.sort(reverse=True)
            return scored[0][1]
        return None

    def score_relevance(self, event: TimelineEvent, symptom: str) -> float:
        """Score how likely an event caused the symptom."""
        score = 0.0

        # Time proximity (more recent = higher)
        time_delta = (datetime.now() - event.timestamp).total_seconds()
        score += max(0, 1 - time_delta / 3600)  # Decay over 1 hour

        # Component overlap
        symptom_components = self.extract_components(symptom)
        overlap = set(event.affected_components) & set(symptom_components)
        score += len(overlap) * 0.5

        # Event type relevance
        if event.event_type in [EventType.DEPLOYMENT, EventType.CONFIG_CHANGE]:
            score += 0.3

        return score
```

---

## Storage Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Storage Layer                            │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │    Redis    │  │  PostgreSQL │  │   S3/Minio  │         │
│  │  (Hot Data) │  │ (Warm Data) │  │  (Archive)  │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                 │
│         ├────────────────┼────────────────┤                 │
│         │                │                │                 │
│  Current State     Historical Data    Old Archives          │
│  Session Cache     Timeline Events    Session Logs          │
│  Health Status     Decision Records   Metrics History       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

```
New Data → Redis (immediate access)
              │
              │ (after 1 hour)
              ▼
        PostgreSQL (queryable history)
              │
              │ (after 90 days)
              ▼
           S3/Minio (compressed archive)
```

---

## Memory Service Implementation

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-service
  namespace: cortex-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: memory-service
  template:
    metadata:
      labels:
        app: memory-service
    spec:
      containers:
      - name: memory
        image: cortex/memory-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: REDIS_URL
          value: redis://redis.cortex-system.svc:6379
        - name: POSTGRES_URL
          valueFrom:
            secretKeyRef:
              name: memory-db-credentials
              key: url
        - name: S3_ENDPOINT
          value: http://minio.velero.svc:9000
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: memory-service
  namespace: cortex-system
spec:
  selector:
    app: memory-service
  ports:
  - port: 8080
    targetPort: 8080
```

### Database Schema

```sql
-- Sessions
CREATE TABLE sessions (
    session_id UUID PRIMARY KEY,
    started_at TIMESTAMP NOT NULL,
    ended_at TIMESTAMP,
    working_directory TEXT,
    current_task TEXT,
    summary TEXT,
    metadata JSONB
);

-- Decisions
CREATE TABLE decisions (
    decision_id UUID PRIMARY KEY,
    session_id UUID REFERENCES sessions(session_id),
    timestamp TIMESTAMP NOT NULL,
    decision TEXT NOT NULL,
    rationale TEXT,
    alternatives JSONB,
    outcome TEXT
);

-- Timeline Events
CREATE TABLE timeline_events (
    event_id UUID PRIMARY KEY,
    timestamp TIMESTAMP NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    source VARCHAR(50) NOT NULL,
    description TEXT,
    details JSONB,
    caused_by UUID REFERENCES timeline_events(event_id),
    affected_components TEXT[],
    severity VARCHAR(20)
);

-- Infrastructure Snapshots
CREATE TABLE infrastructure_snapshots (
    snapshot_id UUID PRIMARY KEY,
    timestamp TIMESTAMP NOT NULL,
    state JSONB NOT NULL,
    diff_from_previous JSONB
);

-- Indexes
CREATE INDEX idx_timeline_timestamp ON timeline_events(timestamp);
CREATE INDEX idx_timeline_type ON timeline_events(event_type);
CREATE INDEX idx_timeline_components ON timeline_events USING GIN(affected_components);
CREATE INDEX idx_sessions_started ON sessions(started_at);
```

---

## Integration Points

### With Layer Activator

```python
# Layer Activator records activation events
await memory.record_timeline_event(
    event_type=EventType.STACK_ACTIVATION,
    source="layer-activator",
    description=f"Activated {stack_id}",
    details={"cold_start": True, "latency_ms": 32000},
    affected_components=[stack_id],
)
```

### With Claude Code

```python
# At session start
session = await memory.get_current_session()
if session:
    context = f"""
    Continuing from previous session:
    - Task: {session.current_task}
    - Progress: {session.summary}
    - Blockers: {session.blockers}
    - Next steps: {session.next_steps}
    """
```

### With Git/ArgoCD

```python
# Webhook handler for Git pushes
@app.post("/webhooks/git")
async def handle_git_push(payload: GitPushPayload):
    await memory.record_timeline_event(
        event_type=EventType.GIT_COMMIT,
        source="github",
        description=payload.head_commit.message,
        details={
            "sha": payload.head_commit.id,
            "author": payload.head_commit.author.name,
            "files_changed": payload.head_commit.modified,
        },
        affected_components=infer_components(payload.head_commit.modified),
    )
```

---

## Usage Examples

### "What broke the MCP server?"

```python
# Query timeline for recent changes affecting github-mcp-server
events = await memory.get_timeline(
    component="github-mcp-server",
    start=datetime.now() - timedelta(days=1),
)

# Find correlation
cause = await memory.correlate(
    symptom="github-mcp-server crash looping",
    timeframe=timedelta(hours=24),
)

# Result: "Commit abc123 changed pip install logic 2 days ago"
```

### "What did we decide about the registry?"

```python
# Query decisions mentioning registry
decisions = await memory.search_decisions(
    query="registry",
    session_id=None,  # All sessions
)

# Result: "Decision on 2026-01-20: Scale cortex-desktop-mcp to 0
#          because private registry is unavailable"
```

### "Resume from last session"

```python
# Get last session summary
session = await memory.get_current_session()

# Load into context
print(f"Last session ended: {session.ended_at}")
print(f"Task in progress: {session.current_task}")
print(f"Summary: {session.summary}")
print(f"Next steps: {session.next_steps}")
```

---

## Implementation Plan

### Phase 1: Core Memory Service

1. Create memory-service deployment
2. Implement session memory (Redis + PostgreSQL)
3. Basic API endpoints
4. Session start/end tracking

### Phase 2: Infrastructure State

1. Kubernetes state collector
2. Change detection
3. State diff API
4. Health summary

### Phase 3: Timeline

1. Event recording API
2. Git/ArgoCD webhooks
3. Kubernetes event watcher
4. Event correlation

### Phase 4: Claude Integration

1. Session handoff protocol
2. Decision recording from Claude
3. Automatic context loading
4. MCP server for memory queries

---

## Related Documentation

- [[agent-architecture]] - Agent memory and context
- [[layer-activator]] - Stack activation tracking
- [[observability]] - Phoenix tracing integration
- [[gitops]] - Git event capture
