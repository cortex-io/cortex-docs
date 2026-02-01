# Tailscale Ingress Fix - Make *.ry-ops.dev Accessible

**Problem**: MetalLB LoadBalancer IP (10.88.145.200) is not accessible via Tailscale VPN because MetalLB L2 mode uses ARP (Layer 2) which doesn't work across routed VPN (Layer 3).

**Solution**: Configure the Tailscale ingress server (10.88.145.199) as a reverse proxy to forward traffic to 10.88.145.200.

## Current State

- ✅ Tailscale subnet routes are advertised: 10.42.0.0/16, 10.43.0.0/16, 10.88.145.0/24
- ✅ Ingress server (10.88.145.199) is reachable via Tailscale
- ✅ K3s nodes are reachable (10.88.145.190-196)
- ❌ MetalLB LoadBalancer IP (10.88.145.200) times out via Tailscale
- ❌ All *.ry-ops.dev domains are inaccessible

## Services Using MetalLB LoadBalancer

All these ingress resources use 10.88.145.200:
- chat.ry-ops.dev
- grafana.ry-ops.dev
- prometheus.ry-ops.dev
- langflow.ry-ops.dev
- argocd.ry-ops.dev
- cortex-api.ry-ops.dev
- mcp.ry-ops.dev
- And 20+ more *.ry-ops.dev domains

## Step-by-Step Fix

### Step 1: SSH to Ingress Server

```bash
ssh k3s@10.88.145.199
# Password: toor
```

### Step 2: Install HAProxy

```bash
sudo apt update
sudo apt install -y haproxy
```

### Step 3: Configure HAProxy

Create HAProxy configuration to proxy HTTP/HTTPS traffic:

```bash
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

# HTTP (Port 80)
frontend http_front
    bind *:80
    mode tcp
    default_backend http_back

backend http_back
    mode tcp
    server metallb 10.88.145.200:80 check

# HTTPS (Port 443)
frontend https_front
    bind *:443
    mode tcp
    default_backend https_back

backend https_back
    mode tcp
    server metallb 10.88.145.200:443 check
EOF
```

### Step 4: Enable and Start HAProxy

```bash
sudo systemctl enable haproxy
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

Expected output:
```
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/lib/systemd/system/haproxy.service; enabled)
     Active: active (running) since ...
```

### Step 5: Verify HAProxy is Listening

```bash
sudo netstat -tlnp | grep haproxy
```

Expected output:
```
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      <pid>/haproxy
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      <pid>/haproxy
```

### Step 6: Test Locally on Ingress Server

```bash
curl -I http://127.0.0.1 -H "Host: chat.ry-ops.dev"
```

Expected: HTTP 200 or redirect to HTTPS

### Step 7: Update Local /etc/hosts

On your local Mac, update `/etc/hosts` to point all *.ry-ops.dev domains to 10.88.145.199:

```bash
sudo tee -a /etc/hosts > /dev/null <<'EOF'
# Cortex services via Tailscale ingress
10.88.145.199 chat.ry-ops.dev
10.88.145.199 grafana.ry-ops.dev
10.88.145.199 prometheus.ry-ops.dev
10.88.145.199 langflow.ry-ops.dev
10.88.145.199 argocd.ry-ops.dev
10.88.145.199 cortex-api.ry-ops.dev
10.88.145.199 mcp.ry-ops.dev
10.88.145.199 rancher.ry-ops.dev
10.88.145.199 linkerd.ry-ops.dev
10.88.145.199 sandfly.ry-ops.dev
10.88.145.199 sms.ry-ops.dev
10.88.145.199 cortex-reports.ry-ops.dev
10.88.145.199 cloudflare-mcp.ry-ops.dev
10.88.145.199 cortex-mcp.ry-ops.dev
10.88.145.199 github-mcp.ry-ops.dev
10.88.145.199 github-security-mcp.ry-ops.dev
10.88.145.199 kubernetes-mcp.ry-ops.dev
10.88.145.199 langflow-chat-mcp.ry-ops.dev
10.88.145.199 n8n-mcp.ry-ops.dev
10.88.145.199 proxmox-mcp.ry-ops.dev
10.88.145.199 sandfly-mcp.ry-ops.dev
10.88.145.199 unifi-mcp.ry-ops.dev
10.88.145.199 cortex-desktop-mcp.ry-ops.dev
10.88.145.199 youtube-ingestion-mcp.ry-ops.dev
10.88.145.199 youtube-intelligence-mcp.ry-ops.dev
10.88.145.199 tekton.ry-ops.dev
10.88.145.199 tekton-webhooks.ry-ops.dev
10.88.145.199 observability.ry-ops.dev
EOF
```

### Step 8: Test from Local Mac

```bash
# Test HTTP redirect
curl -I http://chat.ry-ops.dev

# Test HTTPS (skip cert verification since it's self-signed)
curl -Ik https://chat.ry-ops.dev

# Test in browser
open https://chat.ry-ops.dev
```

## Troubleshooting

### If HAProxy fails to start:

```bash
# Check config syntax
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# Check logs
sudo journalctl -u haproxy -n 50 --no-pager

# Check if ports are already in use
sudo netstat -tlnp | grep -E ':80|:443'
```

### If connection still fails:

```bash
# Test from ingress server to MetalLB
ping -c 3 10.88.145.200

# Test HTTP to MetalLB directly
curl -I http://10.88.145.200 -H "Host: chat.ry-ops.dev"

# Check HAProxy stats
echo "show stat" | sudo socat stdio /run/haproxy/admin.sock
```

### Check firewall rules:

```bash
# UFW status
sudo ufw status

# If blocking, allow ports
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

## Alternative: iptables NAT (if HAProxy doesn't work)

If HAProxy has issues, use iptables DNAT:

```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Add DNAT rules
sudo iptables -t nat -A PREROUTING -p tcp -d 10.88.145.199 --dport 80 -j DNAT --to-destination 10.88.145.200:80
sudo iptables -t nat -A PREROUTING -p tcp -d 10.88.145.199 --dport 443 -j DNAT --to-destination 10.88.145.200:443
sudo iptables -t nat -A POSTROUTING -j MASQUERADE

# Save rules
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

## Verification Checklist

After configuration:

- [ ] HAProxy is running: `sudo systemctl status haproxy`
- [ ] Ports 80/443 are listening: `sudo netstat -tlnp | grep haproxy`
- [ ] Local test works: `curl -I http://127.0.0.1 -H "Host: chat.ry-ops.dev"`
- [ ] Remote test works: `curl -I http://10.88.145.199 -H "Host: chat.ry-ops.dev"` (from Mac)
- [ ] Browser works: Open https://chat.ry-ops.dev in browser
- [ ] Other domains work: Test grafana.ry-ops.dev, prometheus.ry-ops.dev, etc.

## Success Criteria

You should be able to:
1. Access https://chat.ry-ops.dev from your Mac via Tailscale
2. Access all other *.ry-ops.dev domains
3. See the Traefik ingress routing working correctly
4. Access services both locally (on same network) and remotely (via Tailscale)

## Notes

- This configuration proxies at TCP level (mode tcp), so TLS termination happens at Traefik
- HAProxy simply forwards the encrypted traffic to MetalLB
- Host headers are preserved, so Traefik can route correctly
- This solution works for all services using the MetalLB LoadBalancer

## Current Chat Frontend Status

The chat frontend is fully functional:
- ✅ Frontend serving at http://10.88.145.190:32016 (NodePort)
- ✅ MoE router running with all API endpoints
- ✅ Nginx proxy correctly forwarding /api requests
- ✅ Login, conversations, and chat endpoints working
- ⚠️ Need to set ANTHROPIC_API_KEY for actual Claude responses

Once Tailscale ingress is fixed, chat will be accessible at https://chat.ry-ops.dev
