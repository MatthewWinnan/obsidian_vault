# Proxy Server Setup

Project idea to configure a reverse proxy server to serve internal endpoints, with Tailscale providing methods for external access.

## Goals

- Expose internal services through a central proxy
- Use Tailscale for secure external access
- Avoid exposing services directly to the internet

## Architecture Ideas

- **Reverse Proxy**: Caddy, Nginx, or Traefik to route requests to internal services
- **Tailscale Integration**: 
  - Tailscale Funnel for public HTTPS endpoints
  - Tailscale Serve for private endpoints within tailnet
- **Internal Services**: Route to various home lab services (Proxmox UI, NixOS services, etc.)

## Research Links

- [Tailscale Funnel](https://tailscale.com/kb/1223/funnel) - Expose services to the internet
- [Tailscale Serve](https://tailscale.com/kb/1242/tailscale-serve) - Serve content within tailnet
- [Caddy Reverse Proxy](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)

## TODO

- [ ] Decide on proxy server (Caddy vs Nginx vs Traefik)
- [ ] Plan service routing structure
- [ ] Configure Tailscale Funnel/Serve
- [ ] Set up SSL certificates
- [ ] Document configuration

---
[[Tailscale|← Tailscale]] | [[General Remote Access|← Remote Access]] | [[index|← Home]]