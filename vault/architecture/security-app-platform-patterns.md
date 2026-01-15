---
title: "Security App Platform Patterns"
type: architecture
created: "2026-01-14"
status: draft
tags:
  - architecture
  - security
  - automation
  - sola-security
  - langflow
  - app-gallery
---

# Security App Platform Patterns

> Learnings from Sola Security for building a security-first app platform in Cortex

**Source**: https://sola.security (analyzed 2026-01-14)
**Related**: [[knowledge/langflow-chat-integration-design|Langflow Chat Integration Design]]

---

## Overview

Sola Security built an "AI platform for security teams" that enables prompt-based app creation, template galleries, and multi-platform orchestration. Their core value proposition: **"Give it a prompt, connect your data sources and get a working app in minutes."**

Cortex has similar foundational pieces (Langflow workflows, MCP servers, chat interface) but can learn from Sola's productization strategy.

### Sola's Key Differentiators

1. **Graph Research**: Advanced reasoning mode that maps relationships between resources, controls, and risks across connected data
2. **Read-Only Data Access**: Default read-only connections with limited write only for agentic workflows with guardrails
3. **Security Domain Expertise**: Built-in security knowledge (not general-purpose LLM)
4. **Collaborative Canvas**: Share findings as links, Google Slides, PDFs
5. **Trust-First Architecture**: ISO 27001, SOC 2, GDPR certified with transparency (public Trust Center)

---

## Cortex ↔ Sola Mapping

| Sola Component | Cortex Equivalent | Maturity Gap |
|----------------|-------------------|--------------|
| Prompt → App creation | Chat → Workflow generation | 🟡 Partial |
| App Gallery (templates) | Langflow workflows (10 created) | 🟡 Exists, not discoverable |
| Multi-platform integrations | MCP servers (5 deployed) | 🟢 Strong foundation |
| Security-first design | Security namespace, policies | 🟡 Needs formalization |
| Dashboards & alerts | Grafana + AlertManager | 🟢 Production-ready |
| Compliance (SOC 2, ISO 27001) | Manual checklists | 🔴 Not automated |

**Key Insight**: Cortex has the **infrastructure** but lacks the **user experience layer** that makes security accessible to non-technical teams.

---

## 7 Key Learnings for Cortex

### 1. Prompt-Based Security App Creation

**Sola's Approach**: "Solving security challenges shouldn't take weeks in 2025. With security expertise and advanced reasoning baked in, even the most complex of tasks are just a prompt away from being done."

**Example Prompts**:
- "Do we have publicly accessed S3 buckets?"
- "Show me all admin users across AWS, Azure, and GCP"
- "Detect CVE-2025-30066 in GitHub Actions workflows"

**Cortex Implementation**:

Currently, users must:
1. Open Langflow
2. Import workflow JSON
3. Configure MCP server endpoints
4. Click play button

**Better UX**:
```
User: "Show me all pods without resource limits in cortex-system"

Cortex Chat:
→ Detects intent: Kubernetes security audit
→ Auto-generates Langflow workflow:
  - MCP: kubernetes-mcp-server.k8s_list_pods()
  - Claude: Filter pods missing requests/limits
  - Output: Formatted table with recommendations
→ Saves workflow as "Pod Resource Audit"
→ Executes immediately
→ Shows results in chat
→ Offers: "Run this daily?" → Creates CronJob
```

**Architecture**:
```yaml
# New service: cortex-workflow-generator
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workflow-generator
  namespace: cortex-system
spec:
  containers:
  - name: generator
    image: cortex/workflow-generator:latest
    env:
    - name: LANGFLOW_API_URL
      value: "http://langflow.cortex-system.svc.cluster.local:7860/api/v1"
    - name: CLAUDE_API_KEY
      valueFrom:
        secretKeyRef:
          name: anthropic-api-key
          key: key
```

**Workflow**:
1. User sends natural language prompt to cortex-chat
2. Chat analyzes intent → determines required MCP servers
3. Calls workflow-generator API with requirements
4. Generator creates Langflow JSON dynamically
5. POSTs to Langflow API to create flow
6. Executes flow and streams results back to chat
7. Saves flow to gallery with auto-generated name

**Benefits**:
- Zero-code security automation
- Security expertise embedded in prompt templates
- Instant results (no manual workflow creation)

---

### 2. Curated Template Gallery

**Sola's App Gallery**: Pre-built apps for common use cases:
- Google Workspace: Security and Access Insights
- AWS Cloud Security: Startup Essentials
- GitHub Security: Dependabot Alert Insights
- Multi-Platform Access Control: Admin Users

**Cortex Workflows** (already created):
1. ✅ Kubernetes Health Monitor
2. ✅ MCP Orchestration Symphony
3. ✅ GitHub Security Scanner
4. ✅ Multi-Agent Chat Router
5. ✅ Infrastructure Alert Analyzer
6. ✅ Cost Tracker Reporter
7. ✅ Deployment Validator
8. ✅ Log Pattern Analyzer
9. ✅ Service Health Dashboard
10. ✅ Auto Documentation Generator

**Gap**: No discoverable interface for browsing/installing workflows

**Solution**: Create "Cortex App Gallery" web UI

**Implementation**:
```yaml
# New service: cortex-app-gallery
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-gallery
  namespace: cortex-system
spec:
  containers:
  - name: gallery
    image: cortex/app-gallery:latest
    ports:
    - containerPort: 3000
      name: http
    volumeMounts:
    - name: workflows
      mountPath: /workflows
      readOnly: true
  volumes:
  - name: workflows
    configMap:
      name: langflow-workflows
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-gallery
  namespace: cortex-system
spec:
  rules:
  - host: gallery.cortex.ry-ops.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-gallery
            port:
              number: 3000
```

**Features**:
- Browse workflows by category (Security, Infrastructure, Cost, Monitoring)
- Preview workflow DAG (nodes/edges visualization)
- One-click "Install to Langflow" button
- Search by tags (kubernetes, github, security, cost)
- Version history (track workflow updates)
- Usage statistics (most popular workflows)

**Gallery Metadata** (add to ConfigMap):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: langflow-workflow-metadata
data:
  01-k8s-health-monitor.yaml: |
    name: "Kubernetes Health Monitor"
    description: "Real-time cluster health monitoring"
    category: "Infrastructure"
    tags: ["kubernetes", "monitoring", "health"]
    author: "Cortex System"
    version: "1.0.0"
    installs: 0
    rating: 0.0
    mcp_servers: ["kubernetes"]
    estimated_runtime: "30s"
```

---

### 3. Multi-Platform Integration Strategy

**Sola's Integrations**: 30+ tools across:
- **Cloud Providers**: AWS, Azure, GCP
- **Identity**: Okta, Google Workspace, Azure AD
- **Security**: Wiz, Datadog, GitHub
- **Databases**: MongoDB, PostgreSQL
- **SaaS**: Slack, PagerDuty

**Cortex MCP Servers** (currently 5):
1. ✅ Kubernetes (K8s cluster)
2. ✅ GitHub Security (repos, Dependabot)
3. ✅ Proxmox (VMs, infrastructure)
4. ✅ UniFi (network)
5. ✅ Sandfly (security scanning)

**Expansion Roadmap**:

**Phase 1: Infrastructure** (Weeks 1-2)
- [ ] **Traefik MCP Server**: Ingress inspection, middleware config
- [ ] **ArgoCD MCP Server**: Application sync status, deployment history
- [ ] **Cert-Manager MCP Server**: Certificate status, renewal tracking

**Phase 2: Observability** (Weeks 3-4)
- [ ] **Prometheus MCP Server**: Metrics queries, alert rules
- [ ] **Grafana MCP Server**: Dashboard management, annotations
- [ ] **Loki MCP Server**: Log aggregation queries

**Phase 3: Cloud** (Weeks 5-6)
- [ ] **AWS MCP Server**: EC2, S3, IAM, CloudTrail
- [ ] **DigitalOcean MCP Server**: Droplets, volumes, networking

**Phase 4: Security & Compliance** (Weeks 7-8)
- [ ] **Trivy MCP Server**: Container image scanning
- [ ] **OPA Gatekeeper MCP Server**: Policy violations, constraint reports
- [ ] **Vault MCP Server**: Secrets audit, rotation status

**MCP Server Template** (standardize creation):
```typescript
// mcp-server-template/
interface MCPServer {
  name: string;
  port: number;
  methods: Method[];
  authentication: AuthConfig;
  rateLimits: RateLimit;
}

interface Method {
  name: string;
  description: string;
  parameters: Parameter[];
  returns: Schema;
  example: string;
}

// Auto-generate:
// - Kubernetes Deployment manifest
// - Service definition
// - Ingress rule
// - Langflow MCP Client node template
// - Documentation
```

**Benefits**:
- Consistent MCP server development
- Faster time-to-integration
- Unified authentication/authorization
- Standardized error handling

---

### 4. Security-First Architecture

**Sola's Claims**:
- "We don't train models on your data"
- SOC 2, ISO 27001, GDPR compliance
- Dedicated CISO oversight
- Regular penetration testing

**Cortex Current State**:
- ✅ Security namespace (`cortex-security`)
- ✅ Security compliance checklists (manual)
- ✅ GitHub Actions security workflows
- ⚠️ No formal compliance certification
- ⚠️ Limited data governance policies

**Cortex Security Hardening Checklist**:

**Infrastructure Security**:
- [ ] All services run as non-root
- [ ] SecurityContext enforced (readOnlyRootFilesystem: true)
- [ ] Network policies isolate namespaces
- [ ] PodDisruptionBudgets prevent total outages
- [ ] Resource limits prevent resource exhaustion
- [ ] Secrets stored in Sealed Secrets (not plaintext ConfigMaps)
- [ ] TLS for all inter-service communication

**Data Governance**:
- [ ] No PII stored in logs
- [ ] Sensitive data encrypted at rest
- [ ] 90-day log retention (compliance requirement)
- [ ] Audit trail for all API access
- [ ] Data deletion procedures documented

**Access Control**:
- [ ] RBAC enforced (least privilege)
- [ ] ServiceAccounts scoped per namespace
- [ ] No cluster-admin permissions in production
- [ ] MFA required for admin access
- [ ] Regular access reviews (quarterly)

**Monitoring & Alerting**:
- [ ] Security events logged to centralized system
- [ ] Alert on privilege escalation attempts
- [ ] Detect unauthorized API access
- [ ] Track configuration changes
- [ ] Monitor for CVEs in running containers

**Compliance Documentation**:
```yaml
# New ConfigMap: security-policies
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-policies
  namespace: cortex-security
data:
  data-retention.md: |
    # Data Retention Policy
    - Logs: 90 days (SOC 2 requirement)
    - Audit trails: 1 year
    - Metrics: 30 days
    - Backups: 30 days

  access-control.md: |
    # Access Control Policy
    - MFA required for production access
    - ServiceAccounts use least privilege
    - No shared credentials
    - Access reviews quarterly

  incident-response.md: |
    # Incident Response Plan
    1. Detect: Alerts trigger
    2. Contain: Isolate affected services
    3. Investigate: Collect logs/metrics
    4. Remediate: Apply fixes
    5. Document: Post-mortem required
```

---

### 5. Unified Dashboards & Alerts

**Sola's Value Prop**: "Share security insights, dashboards, alerts and workflows with your team"

**Cortex Implementation**:

**Current**:
- Grafana dashboards (ad-hoc)
- AlertManager rules (scattered)
- Langflow workflows (not discoverable)

**Enhancement**: Create "Cortex Command Center"

**Architecture**:
```
┌─────────────────────────────────────────────────┐
│          Cortex Command Center                   │
│                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐│
│  │ Dashboards  │  │   Alerts    │  │ Workflows││
│  │             │  │             │  │          ││
│  │ • Security  │  │ • Critical  │  │ • Active ││
│  │ • Cost      │  │ • Warning   │  │ • Saved  ││
│  │ • Health    │  │ • Info      │  │ • Gallery││
│  └─────────────┘  └─────────────┘  └──────────┘│
│                                                  │
│  ┌──────────────────────────────────────────┐  │
│  │     Recent Activity Feed                 │  │
│  │                                          │  │
│  │  🔴 [2m ago] Pod crash in cortex-chat   │  │
│  │  🟡 [5m ago] High memory in langflow     │  │
│  │  🟢 [8m ago] Deployment synced           │  │
│  └──────────────────────────────────────────┘  │
│                                                  │
│  ┌──────────────────────────────────────────┐  │
│  │     Quick Actions                        │  │
│  │                                          │  │
│  │  • Run health check                      │  │
│  │  • Generate cost report                  │  │
│  │  • Scan for vulnerabilities              │  │
│  │  • Create new workflow                   │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

**Implementation**:
```yaml
# New service: cortex-command-center
apiVersion: apps/v1
kind: Deployment
metadata:
  name: command-center
  namespace: cortex-system
spec:
  containers:
  - name: frontend
    image: cortex/command-center-ui:latest
    ports:
    - containerPort: 3000
  - name: backend
    image: cortex/command-center-api:latest
    ports:
    - containerPort: 8080
    env:
    - name: GRAFANA_URL
      value: "http://grafana.cortex-system.svc.cluster.local:3000"
    - name: ALERTMANAGER_URL
      value: "http://alertmanager.cortex-system.svc.cluster.local:9093"
    - name: LANGFLOW_URL
      value: "http://langflow.cortex-system.svc.cluster.local:7860"
```

**Features**:
1. **Unified Navigation**: Single entry point for all Cortex tools
2. **Real-time Activity Feed**: WebSocket stream of events
3. **Quick Actions**: One-click workflow execution
4. **Role-Based Views**: CISO sees compliance, DevOps sees infra
5. **Shareable Links**: `/dashboard/security` → specific view

---

### 6. Collaboration & Sharing

**Sola's Approach**: Teams share apps, insights, and workflows

**Cortex Gap**: Workflows stored in ConfigMap, not shareable

**Enhancement**: Workflow versioning + sharing

**Implementation**:
```yaml
# Workflow with metadata
apiVersion: cortex.ry-ops.dev/v1alpha1
kind: Workflow
metadata:
  name: kubernetes-health-monitor
  namespace: cortex-system
  annotations:
    cortex.io/author: "ryan@ry-ops.dev"
    cortex.io/version: "1.2.0"
    cortex.io/created: "2026-01-14"
    cortex.io/public: "true"
spec:
  description: "Real-time cluster health monitoring"
  category: "Infrastructure"
  tags: ["kubernetes", "monitoring", "health"]
  mcpServers:
    - kubernetes-mcp-server
  langflowJSON: |
    {
      "name": "Kubernetes Health Monitor",
      "data": { ... }
    }
  changelog:
    - version: "1.2.0"
      date: "2026-01-14"
      changes: "Added resource utilization metrics"
    - version: "1.1.0"
      date: "2026-01-13"
      changes: "Initial release"
```

**CRD Benefits**:
- Kubernetes-native workflow storage
- Version control built-in
- RBAC integration (who can view/edit)
- GitOps compatible (stored in cortex-gitops repo)
- API-accessible (kubectl get workflows)

**Sharing Flow**:
1. User creates workflow in Langflow
2. Clicks "Save to Gallery"
3. Workflow-generator converts to CRD YAML
4. User reviews metadata (name, description, tags)
5. Commits to cortex-gitops
6. ArgoCD syncs → Workflow appears in gallery
7. Other users "Install" → Langflow imports JSON

---

### 7. Compliance as Code

**Sola's Positioning**: SOC 2, ISO 27001, GDPR as foundational (not bolted-on)

**Cortex Opportunity**: Automate compliance checks

**Implementation**: OPA Gatekeeper + Custom Policies

**Example Policies**:
```yaml
# Constraint: Require resource limits
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireResourceLimits
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet"]
    namespaces:
      - cortex
      - cortex-chat
      - cortex-security
  parameters:
    cpu: "2000m"
    memory: "4Gi"

# Constraint: No privileged containers
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivileged
metadata:
  name: deny-privileged
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - cert-manager

# Constraint: Require NetworkPolicies
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireNetworkPolicy
metadata:
  name: require-network-policy
spec:
  match:
    namespaces:
      - cortex
      - cortex-security
```

**Compliance Reporting Workflow**:
```json
{
  "name": "SOC 2 Compliance Report",
  "data": {
    "nodes": [
      {
        "id": "gatekeeper-violations",
        "type": "MCPClient",
        "data": {
          "server_url": "http://gatekeeper-mcp-server.cortex-security:3000",
          "method": "list_constraint_violations",
          "params": "{}"
        }
      },
      {
        "id": "trivy-scans",
        "type": "MCPClient",
        "data": {
          "server_url": "http://trivy-mcp-server.cortex-security:3000",
          "method": "list_vulnerabilities",
          "params": "{\"severity\": \"HIGH,CRITICAL\"}"
        }
      },
      {
        "id": "compliance-report",
        "type": "ChatAnthropic",
        "data": {
          "input_value": "Generate SOC 2 compliance report:\n\nPolicy Violations:\n{gatekeeper-violations.output}\n\nVulnerabilities:\n{trivy-scans.output}\n\nFormat as executive summary with pass/fail status."
        }
      }
    ]
  }
}
```

**Automated Compliance**:
- CronJob runs daily at 6 AM
- Generates compliance report
- Posts to Slack #security channel
- Creates Jira ticket for violations
- Tracks remediation progress

---

## Implementation Priority Matrix

| Initiative | Impact | Effort | Priority |
|------------|--------|--------|----------|
| Prompt → Workflow generation | 🔴 High | 🟡 Medium | **P0** |
| App Gallery UI | 🔴 High | 🟢 Low | **P0** |
| OPA Gatekeeper policies | 🔴 High | 🟡 Medium | **P1** |
| Command Center dashboard | 🟡 Medium | 🟡 Medium | **P1** |
| Workflow CRD + versioning | 🟡 Medium | 🔴 High | **P2** |
| 10+ new MCP servers | 🟢 Low | 🔴 High | **P2** |
| Compliance automation | 🔴 High | 🔴 High | **P3** |

**Quick Wins** (P0 - Next 2 weeks):
1. App Gallery UI (reuse Langflow UI, add metadata display)
2. Prompt → Workflow prototype (Claude + Langflow API)

**High Impact** (P1 - Weeks 3-4):
3. OPA Gatekeeper deployment + base policies
4. Command Center MVP (aggregated view)

**Long-term** (P2/P3 - Months 2-3):
5. Workflow CRD architecture
6. MCP server expansion
7. Full compliance automation

---

## Success Metrics

**User Experience**:
- Time to create workflow: From 15 minutes → 30 seconds
- Workflow discovery: From "hidden in ConfigMap" → "browse gallery"
- Security insights: From "run kubectl commands" → "ask in chat"

**Security Posture**:
- Policy violations: Track reduction over time
- Vulnerability remediation time: From days → hours
- Compliance audit prep: From weeks → 1 day (auto-generated reports)

**Platform Adoption**:
- Active workflows: Track usage per workflow
- Gallery installs: Most popular workflows
- MCP server utilization: Which integrations are most valuable

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    CORTEX PLATFORM                           │
│                                                              │
│  ┌──────────────┐      ┌──────────────┐                    │
│  │ Cortex Chat  │◄────►│ Workflow Gen │                    │
│  │              │      │              │                    │
│  │ Natural      │      │ Prompt →     │                    │
│  │ Language UI  │      │ Langflow JSON│                    │
│  └──────────────┘      └──────┬───────┘                    │
│         │                     │                             │
│         │                     ▼                             │
│         │              ┌──────────────┐                    │
│         └─────────────►│   Langflow   │                    │
│                        │              │                    │
│                        │ Workflow     │                    │
│                        │ Execution    │                    │
│                        └──────┬───────┘                    │
│                               │                             │
│                               ▼                             │
│         ┌──────────────────────────────────────┐           │
│         │         MCP Server Layer             │           │
│         │                                      │           │
│         │  • Kubernetes  • GitHub  • Proxmox  │           │
│         │  • UniFi       • Sandfly • Traefik  │           │
│         │  • ArgoCD      • Trivy   • Vault    │           │
│         └──────────────────────────────────────┘           │
│                               │                             │
│         ┌─────────────────────┴─────────────────┐          │
│         │                                        │          │
│         ▼                                        ▼          │
│  ┌──────────────┐                      ┌─────────────┐    │
│  │ App Gallery  │                      │ Command     │    │
│  │              │                      │ Center      │    │
│  │ Browse/      │                      │             │    │
│  │ Install      │                      │ Dashboards/ │    │
│  │ Workflows    │                      │ Alerts      │    │
│  └──────────────┘                      └─────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Additional Sola Security Insights

### Product Roadmap Analysis

**Strategic Themes from Roadmap**:
1. **AI Integration**: Unified AI interface, Slack/Jira AI connectors
2. **Enterprise Integrations**: Tenable, Orca, Snyk, Microsoft 365, Google Workspace
3. **Collaboration**: Share Canvas, export to Slides/PDF, enhanced filtering

**What This Reveals**:
- Sola sees itself as **security orchestration layer** (not point solution)
- Focus on **collaborative workflows** (not isolated analyst tools)
- Embedding security into **existing workflows** (Slack, Jira) vs. standalone tools

**Cortex Implication**: Build integrations that meet users where they are (Slack, email) rather than forcing them to open another UI.

---

### Trust & Security Architecture

**Sola's Security Practices** (from Trust Center):

**Data Protection**:
- AES-256 encryption at rest
- Unique encryption keys per customer with rotation
- TLS 1.2+ for all transit
- VPC isolation on AWS

**Access Controls**:
- RBAC + MFA + SSO enforced
- "As-needed basis" data access
- Device locking requirements
- Endpoint detection & response (EDR)

**Monitoring & Testing**:
- Continuous security assessments
- Regular external penetration testing (audited by Deloitte)
- DNS filtering + WAF
- Anti-malware on all endpoints

**Compliance**:
- ISO/IEC 27001:2022 ✅
- SOC 2 (Deloitte audit) ✅
- GDPR ✅
- Public Trust Center with security questionnaires

**Cortex Gap Analysis**:

| Security Practice | Sola | Cortex | Priority |
|-------------------|------|--------|----------|
| Encryption at rest | ✅ AES-256 | ⚠️ K8s default | P1 |
| Key rotation | ✅ Per-customer | ❌ Not implemented | P1 |
| RBAC/MFA | ✅ Enforced | ⚠️ K8s RBAC only | P0 |
| Penetration testing | ✅ Regular | ❌ Never done | P2 |
| Public Trust Center | ✅ trust.sola.security | ❌ None | P3 |
| Compliance certs | ✅ ISO/SOC2/GDPR | ❌ None | P3 |
| WAF | ✅ Deployed | ⚠️ Traefik only | P2 |
| EDR | ✅ All endpoints | ❌ Not deployed | P2 |

**Immediate Actions** (P0/P1):
1. Enable Kubernetes secrets encryption at rest
2. Implement SSO/MFA for all Cortex UIs (Langflow, Grafana, etc.)
3. Document data flows (create Cortex Trust documentation)
4. Enable encryption key rotation for Sealed Secrets

---

### Graph Research Pattern

**Sola's Innovation**: "Maps relationships between resources, controls, and risks across your connected data"

**Example Investigation**:
```
Question: "Are there publicly accessible S3 buckets?"

Sola's Graph Research:
1. Query AWS MCP → List all S3 buckets
2. Query IAM policies → Identify public access grants
3. Query CloudTrail → Find who created these policies
4. Map relationships:
   - Bucket → Policy → User → Team
5. Risk assessment:
   - Sensitivity of data in bucket (scan for PII)
   - Compliance impact (GDPR violation if EU customer data)
   - Blast radius (how many other resources accessible by same user)
6. Generate visual graph showing relationships
7. Provide remediation steps with context
```

**Cortex Implementation**:

Currently, Cortex workflows query single MCP servers in isolation. Graph Research requires **cross-MCP correlation**.

**New Langflow Workflow**: "12-graph-investigation.json"

```yaml
nodes:
  - id: k8s-query
    type: MCPClient
    data:
      server_url: http://kubernetes-mcp-server:3001
      method: k8s_list_services
      params: {"type": "LoadBalancer"}

  - id: traefik-query
    type: MCPClient
    data:
      server_url: http://traefik-mcp-server:3002
      method: list_ingresses
      params: {}

  - id: cert-query
    type: MCPClient
    data:
      server_url: http://cert-manager-mcp-server:3004
      method: list_certificates
      params: {}

  - id: graph-analyzer
    type: ChatAnthropic
    data:
      input_value: |
        Analyze these infrastructure components and map their relationships:

        LoadBalancer Services: {k8s-query.output}
        Ingresses: {traefik-query.output}
        TLS Certificates: {cert-query.output}

        Create a graph showing:
        1. Which services are externally accessible
        2. Which domains they expose
        3. Certificate validity status
        4. Security risks (e.g., expired certs, no TLS)
        5. Relationships (Service → Ingress → Certificate)

        Format as visual ASCII diagram with risk assessment.
```

**Benefits**:
- Understand blast radius of changes
- Identify security gaps across systems
- Trace impact of incidents
- Visualize dependencies

---

## Related Documentation

- [[projects/langflow-workflows-2026-01-13|Langflow Workflows]] - 10 existing workflows
- [[knowledge/langflow-chat-integration-design|Langflow Chat Integration Design]] - Three-phase integration plan
- [[architecture/automation-architecture|Automation Architecture]] - Self-healing systems
- [[operations/security/security-compliance-checklist|Security Compliance Checklist]] - Manual checklists

---

## Next Steps

1. **Prototype**: Build minimal App Gallery UI this week
2. **Demo**: Show prompt → workflow generation to stakeholders
3. **Prioritize**: Choose P0/P1 initiatives based on feedback
4. **Execute**: 2-week sprint on quick wins

---

*Created: 2026-01-14*
*Last Updated: 2026-01-14*
*Status: Draft - Ready for review*
