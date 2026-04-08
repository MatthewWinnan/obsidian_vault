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
- [x] Create Tailscale account
- [x] Install Tailscale on proxy machine (th0r)
- [x] Enable MagicDNS in admin console
- [x] Enable HTTPS certificates in admin console
- [x] Verify machine has `*.ts.net` DNS name
- [x] Configure Funnel ACL in admin console

### Phase 2: Caddy Setup
- [x] Install Caddy with caddy-defender plugin
- [x] Create NixOS Caddy configuration
- [x] Configure sops-nix secrets for service IPs
- [x] Test basic reverse proxy to internal service

### Phase 3: Service Configuration
- [x] Map out all internal services and their addresses
- [x] Configure Caddy routes for Home Assistant
- [x] Test each service through the proxy
- [x] Configure trusted_proxies in Home Assistant

### Phase 4: External Access
- [x] Configure Tailscale Funnel via systemd service
- [x] Test access from outside tailnet
- [x] Add landing page
- [x] Document final configuration

---

## Example Service Routing Plan

| Service                                  | Internal Address  | Proxy Path/Subdomain      | Notes                  |
| ---------------------------------------- | ----------------- | ------------------------- | ---------------------- |
| Proxmox                                  | 192.168.1.10:8006 | `/proxmox` or `proxmox.*` | WebSocket for console  |
| [[Home Assistant Setup\|Home Assistant]] | fy3r.ts.net:8123  | `/ha` or `ha.*`           | WebSocket for frontend |
| Jellyfin                                 | 192.168.1.30:8096 | `/media` or `media.*`     |                        |
| Grafana                                  | 192.168.1.40:3000 | `/grafana` or `grafana.*` |                        |

---

## Current Implementation: th0r (NixOS)

### Status: ✅ Complete (2026-04-07)

th0r serves as the single public entry point with Caddy reverse proxy, Tailscale Funnel, and AI crawler protection.

### Architecture

```
Internet → Tailscale Funnel (TLS) → th0r (Caddy HTTP) → Internal Services
                ↓
        Let's Encrypt cert for *.ts.net
        managed by Tailscale
```

**Key insight:** Tailscale Funnel handles TLS termination, so Caddy only needs to serve HTTP locally.

### Access URLs

| Path | Service | Authentication |
|------|---------|----------------|
| `https://th0r.tail13fdb.ts.net/` | Landing page | None |
| `https://th0r.tail13fdb.ts.net/ha/` | Home Assistant | HA built-in auth |

### NixOS Configuration

**`nix/services/caddy-proxy/default.nix`:**
```nix
# Caddy reverse proxy with Tailscale Funnel
#
# Architecture:
#   Internet → Tailscale Funnel (TLS termination) → Caddy (HTTP) → Backend
#
# Tailscale handles certificates automatically (Let's Encrypt for *.ts.net)
{
  config,
  pkgs,
  ...
}:
let
  # Custom Caddy with defender plugin
  caddyCustom = pkgs.caddy.withPlugins {
    plugins = [
      "pkg.jsn.cam/caddy-defender@v0.10.0"
    ];
    hash = "sha256-uucD7cwXlZBA3FbxP6ep0Ysz2LcDBpb4Q/gHuqtcWG8=";
  };

  localPort = 8080;
  funnelPort = 443;

  # Landing page directory in nix store
  landingPageDir = ./landing-page;
in
{
  # Caddy reverse proxy (HTTP only - Tailscale handles TLS)
  services.caddy = {
    enable = true;
    package = caddyCustom;

    globalConfig = ''
      admin off
    '';

    extraConfig = ''
      :${toString localPort} {
        # Block AI crawlers (available: openai, deepseek, githubcopilot, aws, gcloud, etc.)
        defender block {
          ranges openai deepseek githubcopilot
        }

        # Serve landing page at root
        @root path /
        handle @root {
          root * ${landingPageDir}
          try_files /index.html
          file_server
        }

        # Home Assistant proxy
        handle /ha/* {
          uri strip_prefix /ha
          reverse_proxy {$HA_BACKEND} {
            header_up Host {upstream_hostport}
            header_up X-Forwarded-Proto https
          }
        }

        # Default: 404 for unknown paths
        handle {
          respond "Not Found" 404
        }
      }
    '';
  };

  # Load backend address from sops secret
  systemd.services.caddy.serviceConfig.EnvironmentFile = [
    config.sops.secrets."caddy-env".path
  ];

  # Tailscale Funnel service
  systemd.services.tailscale-funnel = {
    description = "Tailscale Funnel for Caddy proxy";
    after = [ "tailscaled.service" "caddy.service" "network-online.target" ];
    wants = [ "tailscaled.service" "network-online.target" ];
    wantedBy = [ "multi-user.target" ];

    path = [ pkgs.tailscale pkgs.jq ];

    preStart = ''
      # Wait for tailscale to be ready
      until tailscale status --json | jq -e '.BackendState == "Running"' > /dev/null 2>&1; do
        sleep 1
      done
    '';

    serviceConfig = {
      Type = "oneshot";
      RemainAfterExit = true;
      ExecStart = "${pkgs.tailscale}/bin/tailscale funnel --bg --https=${toString funnelPort} http://127.0.0.1:${toString localPort}";
      ExecStop = "${pkgs.tailscale}/bin/tailscale funnel --https=${toString funnelPort} off";
      Restart = "on-failure";
      RestartSec = "5s";
    };
  };

  # Firewall - only local port needed, Tailscale handles external
  networking.firewall.allowedTCPPorts = [ localPort ];
}
```

**`machines/th0r/nix/sops.nix`** - Secrets configuration:
```nix
{
  sops = {
    defaultSopsFile = ../../../secrets/th0r.yaml;
    age.sshKeyPaths = ["/etc/ssh/ssh_host_ed25519_key"];
    secrets = {
      "caddy-env" = { mode = "0400"; };
    };
  };
}
```

**`secrets/th0r.yaml`** (encrypted) should contain:
```yaml
caddy-env: HA_BACKEND=fr3yr:8123
```

### Home Assistant trusted_proxies

Add proxy IP to `nix/services/home-assistant.nix`:
```nix
http = {
  use_x_forwarded_for = true;
  trusted_proxies = [
    "127.0.0.1"
    "::1"
    "100.64.0.0/10"      # Tailscale CGNAT range
    "192.168.101.0/24"   # Local LAN (th0r proxy)
  ];
};
```

### Required Tailscale ACL Configuration

In Tailscale Admin Console (https://login.tailscale.com/admin/acls), add:

```json
{
  "nodeAttrs": [
    {
      "target": ["th0r"],
      "attr": ["funnel"]
    }
  ]
}
```

---

## Security Recommendations

### Current Security Posture

| Layer | Protection |
|-------|------------|
| Transport | HTTPS via Tailscale (Let's Encrypt) |
| Application | Home Assistant built-in authentication |
| Bot Protection | caddy-defender blocks AI crawlers |
| Path Restriction | Only `/` and `/ha/*` served, all else returns 404 |

### Recommended Actions

**Immediate:**
- [x] Enable 2FA/MFA in Home Assistant (Settings → People → Your User → Enable MFA)
- [x] Use strong, unique passwords
- [x] Keep Home Assistant updated
- [x] Return 404 for unknown paths (reduce attack surface)

**Consider Later:**
- [ ] Add rate limiting for brute-force protection
- [ ] Set up fail2ban for repeated failed attempts
- [ ] Forward auth (Authelia/Authentik) for SSO across services
- [ ] Monitor Home Assistant logs for suspicious activity

### Alternative: Tailnet-Only Access (Most Secure)

If public access isn't needed, disable Funnel entirely:

```nix
# Use serve instead of funnel - only tailnet members can access
ExecStart = "${pkgs.tailscale}/bin/tailscale serve --bg --https=443 http://127.0.0.1:8080";
```

### Forward Auth Options

For proper SSO with login pages:

| Option | Description | Resources |
|--------|-------------|-----------|
| **Authelia** | Lightweight, 2FA support, ~50MB RAM | [authelia.com](https://authelia.com) |
| **Authentik** | Full OIDC/SAML, user management, ~500MB+ RAM | [goauthentik.io](https://goauthentik.io) |
| **Tailscale-only** | Disable Funnel, use Tailscale identity | Simplest, most secure |

---

## Lessons Learned / Gotchas

### 1. caddy-tailscale Funnel Not Supported

The `funnel on` subdirective in caddy-tailscale's bind configuration is not implemented (see [GitHub issue #26](https://github.com/tailscale/caddy-tailscale/issues/26)).

**Solution:** Use systemd service with `tailscale funnel` command directly.

### 2. Tailscale CLI Changed

Old syntax `tailscale funnel 443 on` no longer works.

**New syntax:**
```bash
tailscale funnel --bg --https=443 http://127.0.0.1:8080
```

### 3. Caddy Host Header Matching

Using `http://localhost:8080` only matches requests with `Host: localhost`. Requests via Tailscale Funnel have `Host: th0r.tail13fdb.ts.net`.

**Solution:** Use `:8080` to match any host.

### 4. CSS Braces in Caddyfile

Caddy interprets `{` and `}` as placeholder markers. Inlining CSS with `respond` breaks because empty `{}` resolves to nothing.

**Solution:** Use `file_server` with `try_files` instead of inlining HTML.

### 5. Home Assistant trusted_proxies

HA requires the proxy's IP in `trusted_proxies` to accept `X-Forwarded-For` headers.

**Error:** `Received X-Forwarded-For header from an untrusted proxy`

**Solution:** Add both Tailscale (`100.64.0.0/10`) and local LAN (`192.168.101.0/24`) ranges.

### 6. file_server Without try_files

`handle /` with `file_server` returns empty because `/` is a directory request, not a file.

**Solution:** Add `try_files /index.html` to explicitly serve index.html.

---

## Related Projects

- [[Home Assistant Setup]] - NixOS + Home Assistant on Raspberry Pi 4

---
[[Tailscale|← Tailscale]] | [[General Remote Access|← Remote Access]] | [[index|← Home]]