# Proxy Server Setup

Configure Caddy as a reverse proxy on a Tailscale node to securely expose internal services with HTTPS certificates provisioned through Tailscale.

## Goals

- Expose internal services through Caddy reverse proxy
- Use Tailscale for secure external access (no port forwarding required)
- Automatic HTTPS certificates via Tailscale's built-in CA
- Single entry point for all internal services

## Architecture

```
Internet → Tailscale Funnel → Caddy (on Tailscale node) → Internal Services
                                   ↓
Tailnet → Tailscale Serve → Caddy → Internal Services
```

- **Caddy** runs on a dedicated machine/VM joined to your tailnet
- **Tailscale** provides the network mesh and HTTPS certificates
- **Internal services** remain unexposed to the public internet

---

## Part 1: Getting Started with Tailscale

### 1.1 Create a Tailscale Account

1. Go to [tailscale.com/start](https://tailscale.com/start)
2. Sign up with your identity provider (Google, Microsoft, GitHub, etc.)
3. This creates your **tailnet** - your private mesh network

### 1.2 Install Tailscale on Your First Device

**Linux (Debian/Ubuntu):**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**NixOS** (add to configuration.nix):
```nix
services.tailscale.enable = true;
```
Then run: `sudo tailscale up`

**Other platforms:** [tailscale.com/download](https://tailscale.com/download)

### 1.3 Authenticate and Join Tailnet

```bash
sudo tailscale up
```
- Opens a browser link to authenticate
- Device joins your tailnet and receives a `100.x.x.x` IP address
- Check status: `tailscale status`

### 1.4 Key Concepts

| Concept | Description |
|---------|-------------|
| **Tailnet** | Your private mesh network |
| **MagicDNS** | Automatic DNS for devices (e.g., `myserver.tailnet-name.ts.net`) |
| **Funnel** | Expose services to the public internet |
| **Serve** | Expose services within your tailnet only |
| **HTTPS Certificates** | Tailscale provisions Let's Encrypt certs for `*.ts.net` domains |

### Resources
- [Tailscale Quickstart](https://tailscale.com/kb/1017/install)
- [How Tailscale Works](https://tailscale.com/blog/how-tailscale-works)
- [Tailscale on NixOS](https://tailscale.com/kb/1063/install-nixos)

---

## Part 2: Enable HTTPS Certificates

Tailscale can provision HTTPS certificates for your devices via their `*.ts.net` domain.

### 2.1 Enable MagicDNS and HTTPS

1. Go to [Tailscale Admin Console](https://login.tailscale.com/admin/dns)
2. Under **DNS**, enable **MagicDNS**
3. Under **HTTPS Certificates**, click **Enable HTTPS**

### 2.2 Get Your Machine's HTTPS Name

```bash
tailscale cert --help
```

Your machine will have a name like: `myproxy.tail-name.ts.net`

Check your full domain:
```bash
tailscale status --json | jq -r '.Self.DNSName'
```

### Resources
- [Tailscale HTTPS Certificates](https://tailscale.com/kb/1153/enabling-https)
- [Using Tailscale with Caddy](https://tailscale.com/kb/1190/caddy-certificates)

---

## Part 3: Install and Configure Caddy

### 3.1 Install Caddy

**Debian/Ubuntu:**
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

**NixOS:**
```nix
services.caddy.enable = true;
```

### 3.2 Install Caddy with Tailscale Module

For automatic certificate provisioning, use Caddy with the Tailscale module:

```bash
# Build Caddy with Tailscale support using xcaddy
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
xcaddy build --with github.com/tailscale/caddy-tailscale
```

Or use the pre-built Docker image:
```bash
docker pull ghcr.io/tailscale/caddy-tailscale:latest
```

### Resources
- [Caddy Installation](https://caddyserver.com/docs/install)
- [caddy-tailscale Plugin](https://github.com/tailscale/caddy-tailscale)

---

## Part 4: Configure Caddy with Tailscale HTTPS

### 4.1 Option A: Manual Certificate Provisioning

Generate certificates manually and configure Caddy to use them:

```bash
# Generate certs (stored in ~/.local/share/tailscale/certs/)
sudo tailscale cert myproxy.tail-name.ts.net
```

**Caddyfile:**
```caddyfile
myproxy.tail-name.ts.net {
    tls /var/lib/tailscale/certs/myproxy.tail-name.ts.net.crt /var/lib/tailscale/certs/myproxy.tail-name.ts.net.key

    reverse_proxy /proxmox/* 192.168.1.10:8006
    reverse_proxy /homeassistant/* 192.168.1.20:8123
    reverse_proxy /* 192.168.1.30:80
}
```

### 4.2 Option B: Automatic via caddy-tailscale Plugin (Recommended)

With the Tailscale Caddy plugin, certificates are managed automatically:

**Caddyfile:**
```caddyfile
{
    tailscale
}

myproxy.tail-name.ts.net {
    reverse_proxy /proxmox/* 192.168.1.10:8006 {
        header_up Host {upstream_hostport}
    }

    reverse_proxy /homeassistant/* 192.168.1.20:8123

    reverse_proxy /* 192.168.1.30:80
}
```

### 4.3 Subdomain Routing (Alternative)

Use subdomains instead of paths for cleaner URLs:

```caddyfile
{
    tailscale
}

proxmox.myproxy.tail-name.ts.net {
    reverse_proxy 192.168.1.10:8006
}

homeassistant.myproxy.tail-name.ts.net {
    reverse_proxy 192.168.1.20:8123
}
```

**Note:** Subdomain routing requires wildcard DNS which may need additional Tailscale configuration.

### Resources
- [Caddy Reverse Proxy Docs](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [Tailscale + Caddy Integration Guide](https://tailscale.com/kb/1190/caddy-certificates)

---

## Part 5: Expose via Tailscale Serve/Funnel

### 5.1 Tailscale Serve (Tailnet Only)

Expose Caddy to devices on your tailnet:

```bash
# Serve Caddy's HTTPS port to your tailnet
tailscale serve https / http://127.0.0.1:80
```

Or for background mode:
```bash
tailscale serve --bg https / http://127.0.0.1:80
```

### 5.2 Tailscale Funnel (Public Internet)

Expose to the public internet (no port forwarding needed):

```bash
# First, enable Funnel in Admin Console under Access Controls
tailscale funnel 443
```

Or configure via `tailscale serve`:
```bash
tailscale serve --funnel https / http://127.0.0.1:80
```

### 5.3 Check Serve/Funnel Status

```bash
tailscale serve status
```

### Resources
- [Tailscale Serve](https://tailscale.com/kb/1242/tailscale-serve)
- [Tailscale Funnel](https://tailscale.com/kb/1223/funnel)

---

## Implementation Checklist

### Phase 1: Tailscale Setup
- [ ] Create Tailscale account
- [ ] Install Tailscale on proxy machine
- [ ] Enable MagicDNS in admin console
- [ ] Enable HTTPS certificates in admin console
- [ ] Verify machine has `*.ts.net` DNS name

### Phase 2: Caddy Setup
- [ ] Install Caddy (with tailscale plugin for automatic certs)
- [ ] Create initial Caddyfile configuration
- [ ] Test basic reverse proxy to one internal service
- [ ] Verify HTTPS works with Tailscale certificate

### Phase 3: Service Configuration
- [ ] Map out all internal services and their addresses
- [ ] Configure Caddy routes for each service
- [ ] Test each service through the proxy
- [ ] Add any needed header modifications for WebSocket support, etc.

### Phase 4: External Access
- [ ] Decide: Serve (tailnet only) vs Funnel (public)
- [ ] Configure Tailscale Serve/Funnel
- [ ] Test access from outside tailnet (if using Funnel)
- [ ] Document final configuration

---

## Example Service Routing Plan

| Service | Internal Address | Proxy Path/Subdomain | Notes |
|---------|-----------------|---------------------|-------|
| Proxmox | 192.168.1.10:8006 | `/proxmox` or `proxmox.*` | WebSocket for console |
| [[Home Assistant Setup\|Home Assistant]] | homeassistant.ts.net:8123 | `/ha` or `ha.*` | WebSocket for frontend |
| Jellyfin | 192.168.1.30:8096 | `/media` or `media.*` | |
| Grafana | 192.168.1.40:3000 | `/grafana` or `grafana.*` | |

---

## Related Projects

- [[Home Assistant Setup]] - NixOS + Home Assistant on Raspberry Pi 4

---
[[Tailscale|← Tailscale]] | [[General Remote Access|← Remote Access]] | [[index|← Home]]