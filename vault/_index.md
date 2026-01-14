# Cortex Knowledge Index

> The Map of Content (MOC) for navigating Cortex documentation.

## Quick Links

- [[meta/curation-workflow|Curation Workflow]] - How docs get organized
- [[_inbox/|Inbox]] - Unsorted new content

## By Domain

### Architecture
System design decisions, infrastructure patterns, and architectural documentation.

**Decision Records (ADRs)**:
- [[architecture/adrs/001-use-moe-architecture|ADR-001: Use MoE Architecture]] - Multi-agent routing strategy
- [[architecture/adrs/002-pattern-based-learning|ADR-002: Pattern-Based Learning]] - Learning from historical patterns
- [[architecture/adrs/003-worker-pool-management|ADR-003: Worker Pool Management]] - Scaling strategy

**Core Architecture**:
- [[architecture/core-principles|Core Principles]] - Foundational architectural principles
- [[architecture/mixture-of-experts|Mixture of Experts]] - MoE routing architecture
- [[architecture/master-worker-pattern|Master-Worker Pattern]] - Coordination pattern
- [[architecture/ml-ai-system|ML/AI System Architecture]] - Machine learning integration
- [[architecture/event-driven|Event-Driven Architecture]] - Event processing patterns
- [[architecture/hybrid-routing|Hybrid Routing Architecture]] - Multi-layer routing
- [[architecture/medallion-architecture|Medallion Architecture]] - Data layer organization
- [[architecture/automation-architecture|Automation Architecture]] - Self-healing systems
- [[architecture/api-versioning|API Versioning Strategy]] - Semantic versioning approach
- [[architecture/self-healing-deployment|Self-Healing Deployment]] - Deployment guide

### Components
Per-service and per-component documentation.
- *Start adding links as docs are created*

### Operations
Runbooks, standard operating procedures, and incident response.

**Runbooks**:
- [[operations/runbook-daily-operations|Daily Operations]] - Daily maintenance checklist
- [[operations/runbook-emergency-recovery|Emergency Recovery]] - Disaster recovery procedures
- [[operations/runbook-worker-failure|Worker Failure Handling]] - Worker recovery procedures
- [[operations/preflight-playbook|Preflight Playbook]] - Pre-deployment checks

**Troubleshooting**:
- [[operations/circuit-breaker-tripped|Circuit Breaker Recovery]] - Breaker reset procedures
- [[operations/moe-router-issues|MoE Router Troubleshooting]] - Router diagnostics
- [[operations/performance-troubleshooting|Performance Troubleshooting]] - Performance debugging
- [[operations/observability-debugging|Observability Debugging]] - Monitoring issues
- [[operations/troubleshooting-common-issues|Common Issues]] - Quick reference guide

**Lifecycle Management**:
- [[operations/worker-lifecycle|Worker Lifecycle]] - Worker management
- [[operations/daemon-management|Daemon Management]] - Daemon operations
- [[operations/daemon-failure|Daemon Failure Recovery]] - Daemon recovery
- [[operations/daemon-monitoring-guide|Daemon Monitoring]] - Health monitoring
- [[operations/task-queue-management|Task Queue Management]] - Queue operations
- [[operations/token-budget-exhaustion|Token Budget Management]] - Token tracking
- [[operations/data-quality-issues|Data Quality Issues]] - Data validation
- [[operations/governance-operations|Governance Operations]] - Policy enforcement
- [[operations/self-healing-system|Self-Healing System]] - Auto-recovery operations

**Security**:
- [[operations/security/security-compliance-checklist|Security Compliance Checklist]] - SOC2/GDPR/ISO27001
- [[operations/security/vulnerability-remediation-workflow|Vulnerability Remediation]] - Security patching
- [[operations/security/github-actions-security-workflows|GitHub Actions Security]] - CVE scanning
- [[operations/security/achievement-automation-security|Achievement Automation Security]] - Token management
- [[operations/security/security-workflow-quick-reference|Security Workflow Reference]] - Quick guide

**Reference**:
- [[operations/asset-catalog|Asset Catalog]] - Component inventory
- [[operations/ports-registry|Ports Registry]] - Port assignments
- [[operations/environment-separation|Environment Separation]] - Dev/staging/prod
- [[operations/github-actions-reports|GitHub Actions Reports]] - Automated reporting
- [[operations/manual-setup-steps|Manual Setup Steps]] - Configuration steps
- [[operations/pre-deployment-testing|Pre-Deployment Testing]] - Test framework

**Status Reports**:
- [[operations/deployment-status-2026-01-13-0513|Deployment Status (2026-01-13)]] - Agent deployment verification
- [[operations/cortex-status-2026-01-13-0707|Cortex Status (2026-01-13)]] - Outline removal validation
- [[operations/outline-blocker-2026-01-13|Outline Blocker]] - Network connectivity issue

### Knowledge
Extracted insights, learnings, and tribal knowledge made explicit.

**Getting Started**:
- [[knowledge/quickstart-guide|Quick Start Guide]] - Get up and running
- [[knowledge/developer-guide|Developer Guide]] - Development workflows
- [[knowledge/command-cheatsheet|Command Cheatsheet]] - Common commands

**Frameworks**:
- [[knowledge/evaluation-framework|Evaluation Framework]] - Testing and validation
- [[knowledge/change-management-framework|Change Management Framework]] - ITIL4-based changes
- [[knowledge/automation-daemons-framework|Automation Daemons Framework]] - Self-healing daemons
- [[knowledge/event-archival-framework|Event Archival Framework]] - Log management
- [[knowledge/ab-testing-framework|A/B Testing Framework]] - Experimentation
- [[knowledge/itil4-improvements|ITIL4 Improvements]] - Best practices adoption

**System Design**:
- [[knowledge/self-healing-system|Self-Healing System]] - Auto-recovery design
- [[knowledge/auto-learning-system|Auto-Learning System]] - Pattern learning
- [[knowledge/coordination-protocol|Coordination Protocol]] - Agent coordination
- [[knowledge/rag-system|RAG System]] - Retrieval-augmented generation
- [[knowledge/event-replay-guide|Event Replay Guide]] - Debugging with replay

**Integration Guides**:
- [[knowledge/ml-ai-quickstart|ML/AI Quickstart]] - Machine learning setup
- [[knowledge/ml-deployment-guide|ML Deployment Guide]] - Model deployment
- [[knowledge/webhook-integration-guide|Webhook Integration]] - Webhook setup
- [[knowledge/moe-router-integration|MoE Router Integration]] - Router setup
- [[knowledge/testing-strategy|Testing Strategy]] - Test pyramid approach

**Troubleshooting Knowledge**:
- [[knowledge/pod-issues-resolution|Pod Issues Resolution]] - K8s pod debugging
- [[knowledge/proxmox-network-issue-resolution|Proxmox Network Issues]] - Network troubleshooting
- [[knowledge/routing-performance-analysis|Routing Performance Analysis]] - Performance tuning

**Best Practices**:
- [[knowledge/prompting-best-practices|Prompting Best Practices]] - Effective prompts
- [[knowledge/langflow-chat-integration-design|Langflow Chat Integration Design]] - Three-phase integration plan
- [[knowledge/langflow-mcp-server-implementation|Langflow MCP Server]] - Pattern-based routing implementation

### Projects
Bounded efforts with defined scope, timeline, and outcomes.

**Active/Recent Projects**:
- [[projects/documentation-system-2026-01-14|Documentation System]] - Self-managing Obsidian vault with curator
- [[projects/docs-migration-from-chaos-to-clarity-2026-01-14|From Chaos to Clarity]] - Documentation migration success story
- [[projects/documentation-migration-progress-2026-01-14|Documentation Migration Progress]] - Phase 1 & 2 complete (87 files)
- [[projects/langflow-integration-2026-01-13|Langflow Integration]] - 12 workflows with MCP integration
- [[projects/langflow-workflows-2026-01-13|Langflow Workflows]] - 10 production-ready workflow definitions

**Major Initiatives**:
- [[projects/itil-implementation-journey|ITIL Implementation Journey]] - 15 ITIL services on K8s
- [[projects/observability-pipeline-phase-1|Observability Pipeline Phase 1]] - Weeks 1-2 (Foundation)
- [[projects/observability-pipeline-phase-2|Observability Pipeline Phase 2]] - Weeks 3-4 (Processors)
- [[projects/observability-pipeline-phase-3|Observability Pipeline Phase 3]] - Weeks 5-6 (Destinations)
- [[projects/observability-pipeline-phase-4|Observability Pipeline Phase 4]] - Weeks 7-8 (API & Dashboard)

**Roadmaps & Vision**:
- [[projects/strategic-roadmap|Strategic Roadmap]] - 4-phase, 16-week improvement plan
- [[projects/self-optimization-plan|Self-Optimization Plan]] - Meta-goal: Use Cortex to build Cortex
- [[projects/full-orchestration-plan|Full Orchestration Plan]] - Massive coordinated builds
- [[projects/mcp-vision|MCP Vision]] - Unified construction company interface

---

## Recent Activity

*This section will be auto-updated by the curation workflow*

---

## Tags Index

Common tags used across the vault:

- `#cortex` - Core Cortex functionality
- `#kubernetes` - K8s-related docs
- `#runbook` - Operational procedures
- `#decision` - Architectural decisions
- `#draft` - Work in progress
- `#stale` - Needs review/update

---

## Statistics

- **Total Documents**: 88 files
- **Architecture**: 14 files (3 ADRs + 11 core docs)
- **Operations**: 33 files (4 runbooks, 5 troubleshooting, 10 lifecycle, 5 security, 6 reference, 3 status)
- **Knowledge**: 27 files (3 getting started, 6 frameworks, 5 system design, 5 integrations, 3 troubleshooting, 5 best practices)
- **Projects**: 14 files (5 active, 5 observability phases, 4 roadmaps)

---

*Last manual update: 2026-01-14 (Phase 2 complete)*
