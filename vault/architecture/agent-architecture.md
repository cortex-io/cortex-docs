---
title: "Cortex Agent Architecture"
type: architecture
created: "2026-01-20"
updated: "2026-01-20"
tags:
  - agents
  - masters
  - workers
  - claude-api
  - mcp
---

# Cortex Agent Architecture

## Overview

Cortex uses a hierarchical Master-Worker architecture where:
- **Masters** orchestrate and route tasks (no Claude API calls)
- **Workers** execute tasks via Claude API conversations and MCP tools

```
                    ┌─────────────────────────┐
                    │   External Request      │
                    └───────────┬─────────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │   CoordinatorMaster     │
                    │   (Top-level router)    │
                    └───────────┬─────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │ SecurityMaster  │ │ [Future]        │ │ [Future]        │
    │ (Division GM)   │ │ DevMaster       │ │ InfraMaster     │
    └────────┬────────┘ └─────────────────┘ └─────────────────┘
             │
      ┌──────┴──────┐
      │             │
      ▼             ▼
┌───────────┐ ┌─────────────────┐
│ Sandfly   │ │ GitHubSecurity  │
│ Worker    │ │ Worker          │
└─────┬─────┘ └───────┬─────────┘
      │               │
      ▼               ▼
  Claude API      Claude API
  + MCP Tools     + MCP Tools
```

---

## Base Classes

### BaseMaster (`agents/base_master.py`)

Masters orchestrate workers but **never call Claude API directly**.

**Responsibilities:**
- Task routing and delegation
- Worker lifecycle management (spawn/terminate)
- Result aggregation
- Health monitoring

**Key Methods:**

| Method | Description |
|--------|-------------|
| `route_task(message)` | Determine which worker handles a task |
| `process_result(message)` | Handle results from workers |
| `spawn_worker(worker_id, worker_class, capabilities)` | Create new worker |
| `find_available_worker(capability)` | Find worker by capability |
| `send_message(recipient, task_type, payload)` | Send message to any agent |

**Lifecycle:**
```
start() → register in registry → create consumer group → start loops
         │
         ├── _heartbeat_loop() - periodic health signals
         ├── _consume_loop() - process incoming messages
         └── _monitor_loop() - check worker health

stop() → cancel tasks → cleanup workers → deregister → disconnect
```

### BaseWorker (`agents/base_worker.py`)

Workers execute tasks via Claude API conversations.

**Responsibilities:**
- Execute tasks using Claude API
- Use MCP tools for specialized operations
- Report results back to masters
- Manage conversation context

**Key Methods:**

| Method | Description |
|--------|-------------|
| `process_task(message)` | Main task processing (subclass implements) |
| `get_system_prompt()` | Claude system prompt for this worker |
| `get_mcp_tools()` | MCP tool definitions |
| `ask_claude(message)` | Simple Claude API call |
| `ask_claude_with_tools(message)` | Agentic loop with tool use |
| `process_tool_call(tool_name, input)` | Execute MCP tool |

**Agentic Loop:**
```
ask_claude_with_tools(message)
    │
    ├── Call Claude API with tools
    │
    ├── If Claude requests tool use:
    │   ├── Execute tool via process_tool_call()
    │   ├── Add tool result to messages
    │   └── Loop (up to max_iterations)
    │
    └── Return final response + tool calls made
```

---

## Implemented Masters

### CoordinatorMaster

**Location:** `masters/coordinator_master.py`
**Agent ID:** `coordinator-master`
**Name:** Cortex Coordinator

**Capabilities:**
- `task_routing`
- `load_balancing`
- `orchestration`
- `system_monitoring`

**Routing Logic:**
```python
# Security tasks → security-master or security workers
if "security" or "threat" or "vulnerability" or "sandfly" in task_type:
    return security_master or find_worker("security_operations")

# GitHub tasks → github workers
if "github" in task_type:
    return find_worker("github_security")

# Unknown → log warning, return None
```

**Spawning Methods:**
- `spawn_security_division()` - Spawns security-master
- `spawn_sandfly_worker()` - Spawns sandfly-worker-001

### SecurityMaster

**Location:** `masters/security_master.py`
**Agent ID:** `security-master`
**Name:** Security Division GM

**Capabilities:**
- `security_operations`
- `threat_management`
- `vulnerability_scanning`
- `incident_coordination`

**Routing Logic:**
```python
# Sandfly tasks
if task_type in ["scan_host", "analyze_threat", "list_findings"]:
    return find_worker("sandfly_api") or spawn_sandfly_worker()

# GitHub Security tasks
if task_type in ["scan_repository", "check_vulnerabilities", "list_alerts"]:
    return find_worker("github_security")
```

**Escalation:**
- Critical findings → `coordinator-master` with `MessagePriority.CRITICAL`
- High threats → `coordinator-master` with `MessagePriority.HIGH`

---

## Implemented Workers

### SandflyWorker

**Location:** `workers/sandfly_worker.py`
**Agent ID:** `sandfly-worker-{id}`
**Master ID:** `security-master`

**Capabilities:**
- `sandfly_api`
- `threat_analysis`
- `intrusion_detection`
- `linux_security`

**System Prompt:**
```
You are a security analyst specializing in Linux intrusion detection using Sandfly.

Your capabilities:
- Analyze Sandfly scan findings
- Identify threat patterns and indicators of compromise (IOCs)
- Assess threat severity and risk
- Generate remediation recommendations
- Correlate findings across multiple hosts
```

**Task Types:**

| Task | Description |
|------|-------------|
| `scan_host` | Analyze findings for a specific host |
| `analyze_threat` | Deep dive on specific threat (MITRE ATT&CK) |
| `list_findings` | Get all findings across hosts |

**MCP Tools:**
- `sandfly_get_findings` - Get findings for a host
- `sandfly_get_host_info` - Get host information
- `sandfly_list_hosts` - List all monitored hosts

### GitHubSecurityWorker

**Location:** `workers/github_security_worker.py`
**Agent ID:** `github-security-worker-{id}`
**Master ID:** `security-master`

**Capabilities:**
- `github_security`
- `dependabot_analysis`
- `code_scanning`
- `vulnerability_assessment`

**System Prompt:**
```
You are a security analyst specializing in GitHub code and dependency security.

Your capabilities:
- Analyze GitHub Security alerts (Dependabot, Code Scanning, Secret Scanning)
- Assess vulnerability severity and exploitability
- Review vulnerable dependencies and suggest upgrades
- Analyze code patterns that lead to vulnerabilities
- Generate remediation recommendations
```

**Task Types:**

| Task | Description |
|------|-------------|
| `scan_repository` | Analyze all security alerts for a repo |
| `check_vulnerabilities` | Check specific vulnerability |
| `list_alerts` | Get alerts across repos |

**MCP Tools:**
- `github_list_dependabot_alerts` - List Dependabot alerts
- `github_list_code_scanning_alerts` - List code scanning alerts
- `github_get_alert_details` - Get detailed alert info

---

## Messaging

### Message Flow

All agents communicate via Redis Streams.

```
┌─────────────┐    Redis Stream    ┌─────────────┐
│   Sender    │ ───────────────► │  Recipient  │
│             │   AgentMessage     │             │
└─────────────┘                    └─────────────┘
```

### AgentMessage Structure

```python
@dataclass
class AgentMessage:
    stream: str              # Target Redis stream
    sender: str              # Sender agent ID
    recipient: str           # Recipient agent ID
    task_type: str           # Type of task
    payload: Dict[str, Any]  # Task data
    priority: MessagePriority
    metadata: Dict[str, Any] # Additional context
    message_id: str          # Redis message ID (auto-generated)
    timestamp: datetime      # Message timestamp
```

### Priority Levels

```python
class MessagePriority(IntEnum):
    LOW = 1
    NORMAL = 2
    HIGH = 3
    CRITICAL = 4
```

---

## Agent Registry

All agents register in the shared registry (Redis).

### AgentInfo

```python
@dataclass
class AgentInfo:
    agent_id: str
    agent_type: AgentType    # MASTER or WORKER
    name: str
    status: AgentStatus
    capabilities: List[str]
    stream: str              # Agent's task stream
    task_count: int          # Tasks processed
    last_heartbeat: datetime
    metadata: Dict[str, Any]
```

### Agent Status

```python
class AgentStatus(str, Enum):
    STARTING = "starting"
    READY = "ready"
    BUSY = "busy"
    IDLE = "idle"
    STOPPED = "stopped"
    ERROR = "error"
```

### Registry Operations

```python
# Register agent
await registry.register(agent_info)

# Update status
await registry.update_status(agent_id, AgentStatus.BUSY)

# Send heartbeat
await registry.heartbeat(agent_id)

# Find workers by capability
workers = await registry.find_workers_by_capability("sandfly_api")

# Cleanup stale agents
await registry.cleanup_stale_agents()
```

---

## Creating New Agents

### New Master

1. Create file in `masters/`
2. Extend `BaseMaster`
3. Implement required methods:

```python
class MyMaster(BaseMaster):
    def __init__(self, **kwargs):
        super().__init__(
            agent_id="my-master",
            name="My Master",
            **kwargs
        )

    def get_capabilities(self) -> List[str]:
        return ["my_capability"]

    async def route_task(self, message: AgentMessage) -> Optional[str]:
        # Return worker_id or None
        pass

    async def process_result(self, message: AgentMessage) -> None:
        # Handle worker results
        pass
```

### New Worker

1. Create file in `workers/`
2. Extend `BaseWorker`
3. Implement required methods:

```python
class MyWorker(BaseWorker):
    def __init__(self, **kwargs):
        agent_id = os.getenv("AGENT_ID", "my-worker-001")
        master_id = os.getenv("MASTER_ID", "my-master")
        super().__init__(
            agent_id=agent_id,
            name=f"My Worker ({agent_id})",
            master_id=master_id,
            **kwargs
        )

    def get_capabilities(self) -> List[str]:
        return ["my_capability"]

    def get_system_prompt(self) -> str:
        return "You are a specialist in..."

    def get_mcp_tools(self) -> List[Dict]:
        return [
            {
                "name": "my_tool",
                "description": "Does something",
                "input_schema": {...}
            }
        ]

    async def process_task(self, message: AgentMessage) -> Dict:
        # Use self.ask_claude() or self.ask_claude_with_tools()
        result = await self.ask_claude_with_tools(message.payload["query"])
        return result

    async def process_tool_call(self, tool_name: str, tool_input: Dict) -> Any:
        # Execute MCP tool
        if tool_name == "my_tool":
            return await self._execute_my_tool(tool_input)
```

---

## Integration with Layer Stacks

The agent architecture maps to Layer Stacks:

| Agent Concept | Layer Stack Equivalent |
|--------------|----------------------|
| Master | MoE Router |
| Worker | MCP Server + Claude |
| Capability | Stack specialization |
| Task routing | Stack activation |

### Future: Layer Activator Integration

```
Request → Layer Activator → Activate Stack → Route to Master → Worker → Claude
                ↓
         Scale to Zero when idle
```

---

## Related Documentation

- [[messaging]] - Redis Streams messaging details
- [[registry]] - Agent registry implementation
- [[lifecycle]] - Agent spawn/terminate lifecycle
- [[layer-activator]] - Serverless stack management (planned)
- [[memory-system]] - Session continuity (planned)
