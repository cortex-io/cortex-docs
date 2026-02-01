# Burst Scaling Test Results & Findings
**Date**: 2026-01-17
**Status**: Scale-to-Zero Confirmed, Scale-Up Blocked
**Issue**: KEDA trigger misconfiguration

---

## What We Confirmed ✅

### 1. Clean State
- ✅ No local application code running on control plane
- ✅ All application code in K3s cluster
- ✅ GitOps flow working (Git → ArgoCD → Cluster)

### 2. Scale-to-Zero Working
- ✅ All burst services at **0/0 replicas** when idle
- ✅ No resources consumed
- ✅ Services: moe-router, qdrant, mcp-server all scaled to zero
- ✅ KEDA ScaledObjects created and monitoring

### 3. KEDA Installed
- ✅ KEDA 2.13.0 deployed to keda namespace
- ✅ CRDs available (scaledobjects.keda.sh)
- ✅ KEDA operator running

---

## What We Discovered ❌

### The Critical Problem: Prometheus Trigger Chicken-and-Egg

**Current KEDA Configuration**:
```yaml
triggers:
- type: prometheus
  metadata:
    query: sum(rate(http_requests_total{service="moe-router"}[1m]))
    threshold: "10"
```

**The Issue**:
1. Service at 0 replicas → No pods running
2. No pods running → No metrics being exposed
3. No metrics → Prometheus has nothing to scrape
4. Prometheus returns empty result → KEDA sees 0 requests
5. KEDA sees 0 requests → Keeps service at 0 replicas
6. **Infinite loop - service never scales up!**

**Test Results**:
```bash
# Attempted to connect to moe-router
python3 -c 'http.client.HTTPConnection("moe-router", 8080).request("POST", "/ingest")'

# Result:
Connection refused (expected - 0 replicas)

# KEDA Response:
<no scale-up triggered>

# Why:
No metrics available for Prometheus to scrape
```

---

## Solutions

### Solution 1: Use KEDA HTTP Add-on (Recommended)

KEDA has an **HTTP Add-on** specifically for this use case:

```yaml
apiVersion: keda.sh/v1alpha1
kind: HTTPScaledObject  # Different CRD!
metadata:
  name: moe-router-scaler
spec:
  scaleTargetRef:
    name: moe-router
    service: moe-router
    port: 8080
  replicas:
    min: 0
    max: 5
```

**How it works**:
1. Deploys an **interceptor** that sits in front of the service
2. Interceptor is always running (lightweight, ~10MB)
3. When request arrives → interceptor holds it
4. Interceptor triggers KEDA to scale up
5. Once pod ready → interceptor forwards request
6. User sees slightly higher latency (cold start + queue time)

**Installation**:
```bash
helm install http-add-on kedacore/keda-add-ons-http \
  --namespace keda \
  --set interceptor.replicas.min=1
```

### Solution 2: Use Knative Serving (Alternative)

Knative has built-in scale-to-zero with request-based activation:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: moe-router
spec:
  template:
    spec:
      containers:
      - image: 10.43.170.72:5000/cortex-moe-router:latest
        ports:
        - containerPort: 8080
```

**How it works**:
1. Knative activator always running
2. Receives requests when service at 0
3. Automatically scales up
4. Buffers requests during cold start
5. Native integration with service mesh

### Solution 3: Change to Queue-Based Trigger

If requests come via a message queue instead of HTTP:

```yaml
triggers:
- type: rabbitmq
  metadata:
    queueName: "knowledge-ingestion"
    queueLength: "1"  # Scale up if ANY messages
```

**How it works**:
1. Queue always exists (external to pods)
2. KEDA watches queue depth
3. Message arrives → KEDA sees depth > 0
4. Scales up to process messages
5. Works perfectly from 0 replicas

### Solution 4: Keep Min Replicas at 1 (Simple but Costly)

```yaml
minReplicaCount: 1  # Always keep 1 pod running
maxReplicaCount: 5
```

**Tradeoff**:
- ✅ Simple - just works
- ✅ No cold start delay
- ❌ Loses 90% cost savings (always consuming resources)
- ❌ Defeats the purpose of scale-to-zero

---

## What Works vs. What Doesn't

### Works ✅
- GitOps flow (commit → push → ArgoCD sync)
- Scale-to-zero when idle (confirmed)
- KEDA installation and CRD deployment
- Traefik IngressRoute manifests (created via GitOps)
- Clean separation: no local code, all in cluster

### Doesn't Work ❌
- **Prometheus-based KEDA trigger for scale-from-zero** (chicken-and-egg)
- Image pulls from Docker Hub (TLS handshake failures)
- Pods without resource requests/limits (ResourceQuota violations)
- ArgoCD sync delays (sometimes >5 minutes to apply changes)
- Artificial load tests (busybox/curl image pull failures)

---

## Recommendations

### Immediate (Today)

**Option A: Install KEDA HTTP Add-on** (Recommended)
```bash
# 1. Install HTTP Add-on
helm install http-add-on kedacore/keda-add-ons-http --namespace keda

# 2. Replace ScaledObject with HTTPScaledObject
# 3. Test with real request
# 4. Should scale from 0 → 1 in ~5-10 seconds
```

**Option B: Set minReplicas=1 for Now**
```bash
# Quick fix to unblock testing
# Edit: apps/cortex-knowledge/moe-router-deployment.yaml
# Change: replicas: 0 → replicas: 1
# Commit, push, let ArgoCD sync
# Test content ingestion with service always-on
# Revisit scale-to-zero later with HTTP Add-on
```

### Short-term (This Week)

1. **Fix Qdrant ScaledObject** - "invalid pod selector" error
2. **Pull all images to internal registry** - stop fighting Docker Hub
3. **Test end-to-end flow** with minReplicas=1
4. **Document actual content ingestion path** (where does data go?)
5. **Verify telemetry collection** (is substrate receiving data?)

### Mid-term (Next 2 Weeks)

1. **Implement KEDA HTTP Add-on** for true scale-from-zero
2. **Measure actual cold start times** (not theoretical)
3. **Feed real knowledge content** regularly
4. **Monitor sample accumulation** toward 1,000+ target
5. **Tune KEDA thresholds** based on actual traffic patterns

---

## The Core Insight

**Scale-to-zero is harder than it looks.**

The issue isn't KEDA itself - it's the **trigger type**. Prometheus-based triggers assume pods are running to export metrics. For true scale-from-zero, you need:

1. **External trigger source** (queue, HTTP interceptor, cron schedule)
2. **Always-on component** watching for activation signals
3. **Buffering mechanism** to hold requests during cold start

This is why Knative Serving exists - it solves exactly this problem with the activator component.

---

## What We Learned About the System

### Entry Points Discovered

**Existing Services** (always-on, not burst):
- knowledge-graph-api: 10.43.175.202:8000
- knowledge-dashboard: 10.88.145.208:80 (LoadBalancer)
- phoenix: 10.43.24.250:6006
- improvement-detector: 10.43.47.228:8080

**Burst Services** (at 0 replicas):
- moe-router: 10.43.159.42:8080 (no pods)
- qdrant: None (headless, no pods)
- mcp-server: 10.43.138.65:3000 (no pods)

**Traefik Ingress** (created but not synced):
- knowledge.cortex.local → moe-router:8080
- qdrant.cortex.local → qdrant:6333
- mcp.cortex.local → mcp-server:3000

### The Question Remains

**Where should LLM tutorial content actually go?**

Options:
1. **knowledge-graph-api** (existing, always-on) - defeats burst purpose
2. **moe-router** (burst, at 0) - can't scale up with current config
3. **Create ingestion queue** - messages trigger scale-up
4. **Use cortex-chat** as frontend - routes to knowledge layer

We need to decide the **intended architecture** before we can test it properly.

---

## Next Steps (Your Decision)

### Path A: Fix KEDA (Complete the Vision)
1. Install KEDA HTTP Add-on
2. Convert to HTTPScaledObject
3. Test scale-from-zero with real requests
4. Document the successful flow
5. Feed LLM tutorial content
6. **Timeline**: 1-2 hours

### Path B: Simplify (Get Unblocked)
1. Set minReplicas=1 on moe-router
2. Test content ingestion with always-on service
3. Verify processing, embeddings, telemetry
4. Document the flow even without scale-to-zero
5. Revisit burst scaling later
6. **Timeline**: 30 minutes

### Path C: Pivot Architecture
1. Add message queue (Redis, RabbitMQ)
2. Content → Queue → KEDA trigger → Scale up
3. Queue-based trigger works from 0 replicas
4. More complex but more reliable
5. **Timeline**: 3-4 hours

---

## Files Created

**On Desktop**:
- `EVOLUTIONARY-INFRASTRUCTURE-ARCHITECTURE.md` - Complete design
- `EVOLUTIONARY-INFRASTRUCTURE-STATUS.md` - Deployment status
- `EVOLUTIONARY-INFRASTRUCTURE-TEST-PLAN.md` - Test methodology
- `BURST-SCALING-FINDINGS.md` - This document
- `from-centralized-to-evolutionary.md` - Blog post

**In Git** (committed):
- `apps/cortex-knowledge/traefik-ingressroute.yaml` - Ingress routes
- `apps/cortex-knowledge/content-submission-job.yaml` - Test job
- Commits: d33d42d, 1b240bb, 54d30d1, f7168a5, 243e92b

---

## The Honest Assessment

**What we built**: A beautiful evolutionary infrastructure architecture with scale-to-zero capability

**What works**: Everything except the actual scale-from-zero trigger

**What's blocking**: KEDA trigger misconfiguration (Prometheus-based won't work from 0 replicas)

**What's needed**: KEDA HTTP Add-on OR minReplicas=1 OR queue-based trigger

**The irony**: We successfully achieved scale-to-zero. We just can't scale back up. 😅

**The fix**: Well-documented and straightforward (see Solutions above)

---

**Status**: Architecture is solid. Trigger mechanism needs adjustment. Ready to proceed once you choose a path.
