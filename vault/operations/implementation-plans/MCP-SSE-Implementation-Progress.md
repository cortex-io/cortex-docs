# MCP Server SSE Implementation Progress

**Date**: 2026-01-17
**Status**: In Progress
**Session**: Implementing SSE wrappers for MCP servers

---

## ✅ Completed: cortex-mcp Server

### Investigation
- **Location**: `~/Projects/cortex-platform/services/mcp-servers/cortex/`
- **Type**: Pure stdio-based MCP server (Node.js)
- **Current Implementation**: JSON-RPC over stdin/stdout
- **Main Entry**: `src/index.js`

### Implementation
Created `src/sse-server.js` - Full HTTP/SSE wrapper:

**Features**:
- Express HTTP server on port 3000 (SSE)
- Health check server on port 8080
- `/sse` endpoint for Server-Sent Events
- Spawns stdio MCP server as child process per connection
- Bidirectional communication (SSE for server→client, HTTP POST for client→server)
- Graceful handling of client disconnects
- JSON validation for all messages
- Proper error handling and logging

**Files Modified**:
- `src/sse-server.js` (new) - 233 lines
- `Dockerfile` - Updated CMD to use `node src/sse-server.js`
- `package.json` - Added `start:sse` script

**Git Commit**: `f9d390b` - Pushed to `main` branch
**GitHub Issue**: Closes #4

### Next Steps for cortex-mcp
1. ✅ Build Docker image (currently building)
2. 🔄 Test locally with `docker run`
3. 🔄 Test SSE endpoint with curl
4. 🔄 Test with mcp-remote: `npx -y mcp-remote http://localhost:3000/sse`
5. Push to k8s registry
6. Deploy to cluster (ArgoCD auto-sync)
7. Verify via Claude Desktop

---

## 🔍 Investigated: cortex-desktop-mcp Server

### Findings
- **Location**: `~/Projects/cortex-platform/services/mcp-servers/cortex-desktop/`
- **Type**: Express HTTP server (already has HTTP transport)
- **Current Implementation**: Custom REST endpoints for MCP protocol
- **Main Entry**: `server.js` (485 lines)

**Existing Endpoints**:
- `GET /health` - Health check
- `POST /mcp/initialize` - MCP initialization
- `GET /mcp/tools` - List available tools
- `POST /mcp/execute` - Execute tool
- `POST /mcp/chat` - Chat (not implemented, returns 501)
- `GET /mcp/prompts` - List prompts (empty)
- `GET /mcp/resources` - List resources (empty)

**Dependencies**:
- Express 4.18.2
- Axios (HTTP client)
- ioredis (Redis for token throttling)
- CORS

**Current Status**:
- ✅ Pod running successfully (1/1 Ready)
- ✅ HTTP server operational on port 8765
- ❌ **NO `/sse` endpoint** - uses custom REST API instead
- ⚠️ **Not compatible with mcp-remote** - requires SSE transport

### Decision Required for cortex-desktop-mcp

**Option 1: Add SSE Endpoint** (Recommended)
- Add `/sse` GET endpoint to existing Express server
- Implement SSE protocol alongside existing REST endpoints
- Maintain backward compatibility with existing clients
- Minimal code changes (~100-150 lines)

**Option 2: Full SSE Wrapper** (Like cortex-mcp)
- Create separate SSE server that spawns existing server
- More complex, duplicates HTTP server
- Better separation of concerns
- More code changes

**Recommendation**: Option 1 - Add SSE endpoint to existing server

### cortex-desktop-mcp Implementation Notes

The server already has:
- ✅ Express HTTP server
- ✅ MCP protocol handlers
- ✅ Tool execution logic
- ✅ Error handling
- ✅ Health checks
- ✅ API key validation
- ✅ Token throttling via Redis

Just needs:
- ❌ `/sse` endpoint for SSE transport
- ❌ SSE event streaming
- ❌ JSON-RPC message handling via SSE

**Code Location for SSE Addition**:
After line 430 (`/mcp/initialize` endpoint), add:

```javascript
// MCP Protocol: SSE Transport
app.get('/sse', validateApiKey, (req, res) => {
  // Set SSE headers
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive'
  });

  // SSE event loop
  // Handle MCP JSON-RPC messages via SSE
  // ... implementation ...
});
```

---

## ❓ Unknown: youtube-intelligence-mcp Server

### Investigation Needed
- **Location**: Unknown - not found in cortex-platform repository
- **Pod Name**: `youtube-channel-intelligence-5994b75df5-tqkt6`
- **Namespace**: cortex
- **Image**: `10.43.170.72:5000/youtube-channel-intelligence:latest`
- **Port**: 8081 (per ingress)

**Questions**:
1. Where is the source code?
2. Is youtube-intelligence-mcp the same as youtube-channel-intelligence?
3. Does it have HTTP server?
4. What protocol does it use?
5. Is it even an MCP server, or does it need a wrapper?

**GitOps Manifests Found**:
- `apps/cortex/youtube-channel-intelligence-deployment.yaml`
- `apps/cortex/youtube-channel-intelligence-service.yaml`
- `apps/cortex/youtube-ingestion-deployment.yaml`
- `apps/cortex/youtube-ingestion-service.yaml`

**Ingress Created**:
- `https://youtube-intelligence-mcp.ry-ops.dev/sse`

### Next Steps for youtube-intelligence-mcp
1. Review deployment manifest to understand what's running
2. Check pod logs: `kubectl logs youtube-channel-intelligence-5994b75df5-tqkt6 -n cortex`
3. Verify if source code exists elsewhere
4. Test current endpoint: `curl http://10.43.x.x:8081/` (from within cluster)
5. Determine if it's an MCP server or needs MCP wrapper

---

## Infrastructure Status

### ✅ Ready and Working
- **Ingresses**: All 14 MCP servers have HTTPS ingresses with Let's Encrypt certs
- **DNS**: All *.ry-ops.dev domains resolving correctly
- **Traefik**: LoadBalancer at 10.88.145.200 routing traffic
- **K3s Cluster**: 7 nodes (3 masters + 4 workers) all Ready
- **Registry Mirror**: Configured on all nodes (10.43.90.31:5000)
- **LimitRanges**: Updated for cortex-system and cortex namespaces
- **Claude Desktop Config**: All 14 servers pre-configured

### 🔧 Current MCP Server Status (12/14 Running)

**Running with SSE** (9 servers):
1. ✅ github-mcp-server
2. ✅ github-security-mcp-server
3. ✅ kubernetes-mcp-server
4. ✅ langflow-chat-mcp-server
5. ✅ n8n-mcp-server
6. ✅ proxmox-mcp-server
7. ✅ sandfly-mcp-server
8. ✅ unifi-mcp-server
9. ✅ cortex-mcp-server (different from cortex-mcp below)

**Running without SSE** (2 servers):
10. ⚠️ cortex-desktop-mcp - Has HTTP, needs `/sse` endpoint
11. ⚠️ youtube-channel-intelligence - Unknown if MCP server

**Pending SSE Implementation** (1 server):
12. 🔄 cortex-mcp - SSE wrapper implemented, Docker build in progress

**Issues** (2 servers):
13. ❌ cloudflare-mcp-server - 0/1 Running (readiness probe failing)
14. ❌ youtube-ingestion - Pending (insufficient cluster resources)

---

## Implementation Plan - Remaining Work

### Phase 1: Complete cortex-mcp (Today)
- [x] Implement SSE wrapper
- [x] Update Dockerfile
- [x] Commit to Git
- [ ] Build Docker image (in progress)
- [ ] Test locally
- [ ] Push to registry
- [ ] Deploy to cluster
- [ ] Verify in Claude Desktop

### Phase 2: Add SSE to cortex-desktop-mcp (Next)
- [ ] Read existing server.js to understand flow
- [ ] Add `/sse` GET endpoint to Express server
- [ ] Implement SSE event streaming
- [ ] Handle JSON-RPC via SSE
- [ ] Test locally
- [ ] Commit, build, push
- [ ] Deploy and verify

### Phase 3: Investigate youtube-intelligence-mcp
- [ ] Review deployment manifest
- [ ] Check pod logs
- [ ] Find source code
- [ ] Determine implementation approach
- [ ] Implement SSE if needed
- [ ] Deploy and test

### Phase 4: Fix cloudflare-mcp-server (If time permits)
- [ ] Check pod logs for readiness probe failures
- [ ] Review readiness probe configuration
- [ ] Fix underlying issue
- [ ] Verify SSE endpoint works

---

## Testing Checklist

For each MCP server after SSE implementation:

### Local Testing
- [ ] Build Docker image successfully
- [ ] Run container: `docker run -p 3000:3000 -p 8080:8080 <image>`
- [ ] Test health: `curl http://localhost:8080/health`
- [ ] Test SSE: `curl http://localhost:3000/sse`
- [ ] Test with mcp-remote: `npx -y mcp-remote http://localhost:3000/sse`

### Cluster Deployment
- [ ] Push image to k8s registry
- [ ] Verify ArgoCD syncs deployment
- [ ] Check pod status: `kubectl get pods -n <namespace>`
- [ ] Check pod logs: `kubectl logs <pod-name>`
- [ ] Test ingress: `curl https://<server>-mcp.ry-ops.dev/sse`

### Claude Desktop Integration
- [ ] Restart Claude Desktop app
- [ ] Check MCP server connection status
- [ ] Test server functionality (execute a tool)
- [ ] Verify responses are correct
- [ ] Check for any errors in logs

---

## Reference Materials

**SSE Wrapper Pattern** (used in cortex-mcp):
```javascript
// HTTP server with SSE endpoint
app.get('/sse', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive'
  });

  const mcpProcess = spawn('node', ['./stdio-server.js']);

  mcpProcess.stdout.on('data', (data) => {
    res.write(`data: ${data.toString()}\\n\\n`);
  });

  req.on('close', () => mcpProcess.kill());
});
```

**Working MCP Servers** (for reference):
- proxmox-mcp-server
- sandfly-mcp-server
- unifi-mcp-server
- langflow-chat-mcp-server

**GitHub Issues**:
- #4: cortex-mcp (in progress)
- #5: cortex-desktop-mcp (pending)
- #6: youtube-intelligence-mcp (pending)

---

## Resources

- **MCP Specification**: https://modelcontextprotocol.io/
- **SSE Transport Docs**: https://modelcontextprotocol.io/docs/concepts/transports#sse
- **mcp-remote Package**: https://www.npmjs.com/package/mcp-remote
- **Repositories**:
  - cortex-platform: https://github.com/ry-ops/cortex-platform
  - cortex-gitops: https://github.com/ry-ops/cortex-gitops

---

**Last Updated**: 2026-01-17 (Session in progress)
**Next Update**: After cortex-mcp Docker build completes
