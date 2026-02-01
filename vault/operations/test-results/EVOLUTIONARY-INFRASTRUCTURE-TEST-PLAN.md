# Evolutionary Infrastructure - Test Plan & Journey Documentation
**Date**: 2026-01-17
**Purpose**: Verify burst scaling works, feed real data, document the learning journey

---

## Pre-Flight Checklist

### ✅ Clean State Verified
- [x] No local application code running
- [x] All burst services at 0/0 replicas (scale-to-zero active)
- [x] KEDA installed and watching
- [x] ArgoCD syncing manifests from Git

### ⏳ Traefik Ingress
- [ ] IngressRoutes deployed via ArgoCD (commit f7168a5)
- [ ] Endpoints accessible: knowledge.cortex.local
- [ ] DNS/hosts configured

### 🎯 Ready to Test
- [ ] Burst scaling functional
- [ ] Cold start times measured
- [ ] Real data fed to cortex-knowledge
- [ ] Journey documented

---

## Phase 1: Verify Burst Scaling (The Proof)

### The Question
**Does scale-to-zero → scale-up actually work?**

### The Test
Instead of artificial load tests that fight image pulls, we'll use **real requests** to trigger scale-up.

### Method 1: Internal Request (Simple)
```bash
# From within cluster
kubectl run -n cortex-knowledge quick-test --image=busybox --rm -it \
  --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"test","image":"busybox","command":["wget","-q","-O-","http://moe-router:8080/health"],"resources":{"requests":{"cpu":"50m","memory":"64Mi"},"limits":{"cpu":"100m","memory":"128Mi"}}}]}}' \
  -- /bin/true
```

**Expected behavior**:
1. kubectl run starts pod
2. Pod tries to wget http://moe-router:8080/health
3. Service exists but 0 pods (connection fails initially)
4. KEDA detects the request attempt (if Prometheus sees it)
5. Scales moe-router 0 → 1
6. Pod starts (~5-10 seconds)
7. Request succeeds

### Method 2: External Request (Traefik)
```bash
# Add to /etc/hosts first
echo "10.88.145.XXX knowledge.cortex.local" >> /etc/hosts

# Trigger scale-up
for i in {1..20}; do
  curl -s http://knowledge.cortex.local/health
  sleep 1
done
```

### Method 3: Feed Real Data (Best)
Skip artificial tests. Send the LLM tutorial content directly via whatever interface cortex-knowledge uses. This naturally triggers scale-up while providing real value.

---

## Phase 2: Feed Real Data (The Purpose)

### The Content
**LLM Tutorial Transcript** (provided by user):
- Topic: Building LLM apps with RAG (Retrieval Augmented Generation)
- Technique: Chunking PDFs, vector embeddings, Q&A chains
- Tools: LangChain, Streamlit, watsonx AI, ChromaDB
- Domain: **Perfect for cortex-knowledge layer** (knowledge ingestion, retrieval, RAG)

### The Journey We're Documenting

```
User provides content
       │
       ▼
Where does it enter the system? ────────────┐
       │                                     │
       ▼                                     │
Does it hit MoE Router first?               │ Document every step
       │                                     │
       ▼                                     │
How long until scale-up? (cold start)       │
       │                                     │
       ▼                                     │
Does Qdrant scale when MoE Router active?   │
       │                                     │
       ▼                                     │
How is the content processed?               │
       │                                     │
       ▼                                     │
Where are embeddings stored?                │
       │                                     │
       ▼                                     │
What tool chains are invoked?               │
       │                                     │
       ▼                                     │
Is telemetry flowing to collector?          │
       │                                     │
       ▼                                     │
How long until scale back to zero? ─────────┘
```

### Questions to Answer

**Entry Point**:
- Where do we POST this content to cortex-knowledge?
- Is there a knowledge-graph-api endpoint?
- Do we use MCP Server tools directly?
- Does it go through MoE Router?

**Pitfalls** (where things might break):
- ❌ Image pull failures (we've seen this)
- ❌ KEDA not triggering (Prometheus metrics missing?)
- ❌ Services don't scale (quota issues? LimitRange?)
- ❌ Cold start too slow (>60 seconds unacceptable?)
- ❌ Telemetry not flowing (substrate services down)

**Time Stands Still** (waiting points):
- ⏳ KEDA polling interval (10 seconds)
- ⏳ Pod cold start (5-30 seconds depending on service)
- ⏳ PVC mount for Qdrant (15-30 seconds)
- ⏳ ArgoCD sync interval (up to 3 minutes)

**What Works**:
- ✅ GitOps flow (commit → push → ArgoCD → cluster)
- ✅ Scale-to-zero when idle (confirmed: all at 0/0)
- ✅ KEDA watching (ScaledObjects created)
- ⏳ Scale-up on demand (to be tested)
- ⏳ Real data processing (to be tested)

**What Doesn't**:
- ❌ Image pulls from Docker Hub (TLS handshake failures)
- ❌ Pods without resource specs (quota violations)
- ❌ Qdrant ScaledObject (invalid pod selector error)
- ❌ Substrate services (placeholder images don't exist)

---

## Phase 3: Document the Flow

### What We'll Capture

**Timeline**:
```
T+0:00  - Content submitted to cortex-knowledge
T+0:05  - KEDA detects traffic
T+0:10  - MoE Router scales 0 → 1
T+0:15  - Pod starts (image pull)
T+0:20  - Pod ready (health checks pass)
T+0:25  - Qdrant scales 0 → 1 (follows MoE Router)
T+0:40  - Qdrant ready (PVC mount + index load)
T+0:45  - First request succeeds
T+1:00  - Content processed, embeddings stored
T+3:00  - No more traffic
T+5:00  - MoE Router scales 1 → 0 (2 min cooldown)
T+8:00  - Qdrant scales 1 → 0 (5 min cooldown)
```

**Metrics**:
- Cold start time (0 → Running → Ready)
- Request latency (first vs. steady-state)
- Resource usage (CPU, memory during burst)
- Scale-down time (idle → 0 replicas)

**Observations**:
- Where did it get stuck?
- What errors appeared?
- What worked smoothly?
- What surprised us?

---

## Phase 4: The Diagram

We'll create a visual flow diagram showing:

```
┌──────────────────────────────────────────────────────────────┐
│                     User Submits Content                      │
│            (LLM tutorial about RAG and LangChain)            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
                ┌────────────────────┐
                │  Traefik Ingress   │
                │ knowledge.cortex   │
                │      .local        │
                └─────────┬──────────┘
                          │
                          ▼
         ┌────────────────────────────────┐
         │      MoE Router Service        │ ◄── KEDA watching (0→5 replicas)
         │   (Currently 0/0 replicas)     │
         └────────────────┬───────────────┘
                          │
                ┌─────────┴─────────┐
                │   KEDA Trigger    │
                │  HTTP requests    │
                │    >10 req/sec    │
                └─────────┬─────────┘
                          │
                          ▼
         ┌────────────────────────────────┐
         │   MoE Router Pod Starting     │
         │   Cold Start: ~5-10 seconds    │
         └────────────────┬───────────────┘
                          │
                ┌─────────┴─────────┐
                │  Qdrant Trigger   │
                │ MoE Router ≥1 pod │
                └─────────┬─────────┘
                          │
                          ▼
         ┌────────────────────────────────┐
         │    Qdrant Pod Starting        │
         │  Cold Start: ~15-30 seconds    │
         │  (PVC mount + index load)      │
         └────────────────┬───────────────┘
                          │
                          ▼
         ┌────────────────────────────────┐
         │   Content Processing Flow     │
         │                                │
         │ 1. MoE routes to expert model  │
         │ 2. Embed content chunks        │
         │ 3. Store in Qdrant vectors     │
         │ 4. MCP tools invoked?          │
         │ 5. Telemetry collected         │
         └────────────────┬───────────────┘
                          │
                          ▼
         ┌────────────────────────────────┐
         │     Idle Period Begins        │
         │   (No more traffic)            │
         └────────────────┬───────────────┘
                          │
                ┌─────────┴─────────┐
                │  KEDA Cooldown    │
                │  2 min (MoE)      │
                │  5 min (Qdrant)   │
                └─────────┬─────────┘
                          │
                          ▼
         ┌────────────────────────────────┐
         │   Scale Back to Zero          │
         │   Resources released          │
         │   90% cost savings            │
         └───────────────────────────────┘
```

---

## Execution Plan

### Step 1: Wait for ArgoCD Sync (2-3 minutes)
```bash
# Wait for Traefik IngressRoutes to deploy
kubectl get ingressroute -n cortex-knowledge -w
```

### Step 2: Test Scale-Up Trigger
```bash
# Method: Feed real content via appropriate endpoint
# (To be determined based on cortex-knowledge interface)
```

### Step 3: Monitor Scale-Up
```bash
# Watch in separate terminals:

# Terminal 1: Pod status
kubectl get pods -n cortex-knowledge -w | grep -E "moe-router|qdrant|mcp-server"

# Terminal 2: HPA activity
kubectl get hpa -n cortex-knowledge -w

# Terminal 3: Events
kubectl get events -n cortex-knowledge -w --sort-by='.lastTimestamp'

# Terminal 4: ScaledObject status
watch -n 2 'kubectl get scaledobject -n cortex-knowledge'
```

### Step 4: Document Observations
Capture:
- Timestamps for each state transition
- Errors encountered
- Workarounds applied
- What worked vs. what failed
- Cold start actual times
- Resource usage during burst

### Step 5: Create Visual Diagram
Based on observations, create an accurate flow diagram showing the actual path content took through the system.

---

## Success Criteria

**Burst Scaling Works** if:
- ✅ Services scale from 0 → N when traffic arrives
- ✅ Cold start completes in <60 seconds (pod ready)
- ✅ Requests succeed after scale-up
- ✅ Services scale back to 0 after cooldown period

**Real Data Processing Works** if:
- ✅ LLM tutorial content ingested successfully
- ✅ Embeddings stored in Qdrant
- ✅ Knowledge-graph or other MCP tools invoked
- ✅ Telemetry collected (if substrate services healthy)

**Journey Documentation Complete** if:
- ✅ Every step from input → output mapped
- ✅ Pitfalls identified and documented
- ✅ Timing measurements recorded
- ✅ Visual diagram created
- ✅ Blog post updated with real-world results

---

## Current Status

**Deployment**:
- ✅ cortex-knowledge burst stack deployed
- ✅ KEDA installed and watching
- ✅ Services at 0/0 replicas (scale-to-zero confirmed)
- ⏳ Traefik ingress syncing (commit f7168a5)
- ❌ Some existing services have issues (improvement-detector, elasticsearch, outline-db-init) - quota violations

**Blockers Identified**:
1. **Qdrant ScaledObject error**: "invalid pod selector"
   - May need to fix StatefulSet selector labels
2. **Image pull failures**: Docker Hub TLS handshake issues
   - Use internal registry for all images
3. **ResourceQuota violations**: Pods without resource specs fail
   - All test pods must specify requests/limits

**Next Steps**:
1. Verify Traefik ingress deployed
2. Determine cortex-knowledge entry point for content
3. Submit LLM tutorial content
4. Monitor and document the entire flow
5. Create accurate diagram from observations
6. Update blog post with real results

---

**Ready to proceed once ArgoCD syncs Traefik ingress (ETA: 2-3 minutes from commit)**
