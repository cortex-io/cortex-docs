# MCP Server SSE Implementation Tasks

**Date**: 2026-01-17
**Status**: Pending Implementation
**Repository**: cortex-platform (code changes)

## Overview

Four MCP servers currently lack SSE (Server-Sent Events) transport support and need HTTP wrappers to enable remote access via the `mcp-remote` npm package.

## Background

- **MCP Protocol**: Supports stdio and SSE transports
- **Current State**: Most MCP servers are stdio-based (communicate via stdin/stdout)
- **Requirement**: Remote access via HTTPS requires SSE transport
- **Solution**: Add HTTP server wrapper that:
  1. Exposes `/sse` endpoint for SSE connections
  2. Bridges SSE ↔ stdio communication with underlying MCP server
  3. Maintains MCP protocol compliance

## Required Implementations

### 1. cortex-mcp
**Location**: `~/Projects/cortex-platform/services/mcp-servers/cortex/`
**Current Type**: Node.js stdio-based MCP server
**Status**: Missing SSE wrapper

**Requirements**:
- Add Express.js or similar HTTP server
- Implement `/sse` endpoint using SSE protocol
- Bridge SSE events to/from stdio MCP server process
- Add health check endpoint at `/health`
- Update Dockerfile to expose port 3000

**Ingress**: Already created at `https://cortex-mcp.ry-ops.dev/sse`

---

### 2. github-mcp
**Location**: `~/Projects/cortex-platform/services/mcp-servers/` (needs verification)
**Current Type**: Custom HTTP server
**Status**: Needs SSE endpoint added

**Requirements**:
- Add `/sse` endpoint to existing HTTP server
- Implement SSE protocol for MCP communication
- Ensure compatibility with mcp-remote client
- Add health check if not present

**Ingress**: Already created at `https://github-mcp.ry-ops.dev/sse`

---

### 3. cortex-desktop-mcp
**Location**: `~/Projects/cortex-platform/services/mcp-servers/cortex-desktop/`
**Current Type**: Needs investigation
**Status**: Missing SSE wrapper

**Requirements**:
- Add HTTP server with SSE support
- Implement `/sse` endpoint
- Bridge to underlying MCP functionality
- Update Dockerfile to expose port 8765
- Add health checks

**Ingress**: Already created at `https://cortex-desktop-mcp.ry-ops.dev/sse`

---

### 4. youtube-intelligence-mcp
**Location**: `~/Projects/cortex-platform/` (needs location verification)
**Current Type**: Needs investigation
**Status**: Missing SSE wrapper

**Requirements**:
- Add HTTP server with SSE support
- Implement `/sse` endpoint
- Bridge to YouTube intelligence functionality
- Update Dockerfile to expose port 8081
- Add health checks

**Ingress**: Already created at `https://youtube-intelligence-mcp.ry-ops.dev/sse`

---

## Implementation Pattern

### Reference Implementation
Use existing working MCP servers as reference:
- **proxmox-mcp-server**: Successfully running with SSE (1/1 Ready)
- **sandfly-mcp-server**: Successfully running with SSE (1/1 Ready)
- **unifi-mcp-server**: Successfully running with SSE (1/1 Ready)

### SSE Wrapper Template (Node.js)

```javascript
const express = require('express');
const { spawn } = require('child_process');

const app = express();
const PORT = process.env.PORT || 3000;

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

// SSE endpoint for MCP protocol
app.get('/sse', (req, res) => {
  // Set SSE headers
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.flushHeaders();

  // Spawn the stdio-based MCP server
  const mcpProcess = spawn('node', ['./stdio-server.js']);

  // Forward stdio output to SSE
  mcpProcess.stdout.on('data', (data) => {
    res.write(`data: ${data.toString()}\n\n`);
  });

  // Handle client disconnect
  req.on('close', () => {
    mcpProcess.kill();
    res.end();
  });

  // Handle MCP server errors
  mcpProcess.on('error', (error) => {
    console.error('MCP process error:', error);
    res.write(`event: error\ndata: ${error.message}\n\n`);
  });

  // Handle client messages (POST requests needed for bidirectional)
  // Note: Full implementation requires WebSocket or POST endpoint for client->server
});

app.listen(PORT, () => {
  console.log(`MCP SSE server listening on port ${PORT}`);
});
```

## Deployment Readiness

All infrastructure is already in place:

✅ **Ingresses Created** (with HTTPS + Let's Encrypt):
- cortex-mcp.ry-ops.dev
- github-mcp.ry-ops.dev
- cortex-desktop-mcp.ry-ops.dev
- youtube-intelligence-mcp.ry-ops.dev

✅ **DNS Configured**: All *.ry-ops.dev domains route to Traefik (10.88.145.200)

✅ **TLS Certificates**: Let's Encrypt DNS-01 challenge via cert-manager

✅ **Claude Desktop Config**: All servers pre-configured in `claude_desktop_config.json`

✅ **K3s Infrastructure**:
- Registry mirror configured (10.43.90.31:5000)
- LimitRanges updated
- NetworkPolicies in place
- ArgoCD auto-sync enabled

## Testing Plan

For each MCP server after SSE implementation:

1. **Build & Push Image**:
   ```bash
   cd ~/Projects/cortex-platform/services/mcp-servers/<server-name>
   docker build -t <registry>/<server-name>:latest .
   docker push <registry>/<server-name>:latest
   ```

2. **Update Deployment** (if needed):
   - Update image tag in cortex-gitops manifests
   - Commit and push to trigger ArgoCD sync

3. **Verify Pod Health**:
   ```bash
   kubectl get pods -n cortex-system | grep <server-name>
   kubectl logs -n cortex-system <pod-name>
   ```

4. **Test SSE Endpoint**:
   ```bash
   curl https://<server-name>-mcp.ry-ops.dev/sse
   ```

5. **Test with mcp-remote**:
   ```bash
   npx -y mcp-remote https://<server-name>-mcp.ry-ops.dev/sse
   ```

6. **Verify in Claude Desktop**:
   - Restart Claude Desktop app
   - Check MCP server connection status
   - Test server functionality

## Current Status Summary

**Working (12/14 MCP servers with SSE)**:
- ✅ cortex-mcp-server (different from cortex-mcp)
- ✅ github-mcp-server
- ✅ github-security-mcp-server
- ✅ kubernetes-mcp-server
- ✅ langflow-chat-mcp-server
- ✅ n8n-mcp-server
- ✅ proxmox-mcp-server
- ✅ sandfly-mcp-server
- ✅ unifi-mcp-server
- ✅ cortex-desktop-mcp (pod running, SSE needs verification)
- ✅ youtube-channel-intelligence (pod running, SSE needs verification)

**Pending SSE Implementation (4 servers)**:
- ❌ cortex-mcp - Node.js stdio server
- ❌ github-mcp - Custom HTTP server
- ⚠️ cortex-desktop-mcp - Pod running, may just need SSE endpoint
- ⚠️ youtube-intelligence-mcp - Pod running, may just need SSE endpoint

**Other Issues**:
- cloudflare-mcp-server: Readiness probe failing (0/1 Running)
- youtube-ingestion: Pending (insufficient cluster resources)

## Next Steps

1. **Investigation Phase**:
   - Review source code for each of the 4 servers
   - Determine current implementation details
   - Identify if SSE wrapper needed or just endpoint addition

2. **Create GitHub Issues**:
   - One issue per server in cortex-platform repository
   - Include implementation requirements
   - Reference this document

3. **Implementation Phase**:
   - Develop SSE wrappers based on reference pattern
   - Test locally with mcp-remote
   - Update Dockerfiles if needed

4. **Deployment Phase**:
   - Build and push new images
   - Update manifests in cortex-gitops (if image tags changed)
   - Monitor ArgoCD sync and pod health

5. **Verification Phase**:
   - Test all endpoints via curl
   - Verify mcp-remote connections
   - Confirm Claude Desktop integration

## References

- **MCP Specification**: https://modelcontextprotocol.io/
- **MCP SSE Transport**: https://modelcontextprotocol.io/docs/concepts/transports#sse
- **mcp-remote Package**: https://www.npmjs.com/package/mcp-remote
- **Reference Servers**: proxmox-mcp, sandfly-mcp, unifi-mcp (already working)

---

**Created by**: Claude Sonnet 4.5
**Date**: 2026-01-17
