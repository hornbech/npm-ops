# NPM Observability Design

**Date:** 2026-04-18  
**Status:** Approved

## Goal

Monitor incoming requests to Nginx Proxy Manager and visualize them in a real-time + historical dashboard. Lightweight, self-hosted, runs on the same host as NPM.

## Approach

**GoAccess proxied through NPM (Approach B)**

- Single container (`xavierh/goaccess-for-nginxproxymanager`) added alongside the existing NPM stack
- GoAccess reads NPM's log directory read-only via volume mount
- NPM proxies the GoAccess dashboard at `stats.yourdomain.com` with SSL + basic auth
- WebSockets enabled in NPM proxy host for real-time live feed

## Architecture

```
[Internet] → NPM (443) → stats.yourdomain.com
                               ↓ (internal, host.docker.internal:7880)
                          GoAccess container :7880
                               ↑ (read-only volume mount)
                          /home/jhh/npm/data/logs
```

## docker-compose.yml (this repo: ~/projects/npm-obs)

```yaml
services:
  goaccess:
    image: xavierh/goaccess-for-nginxproxymanager:latest
    container_name: goaccess
    restart: unless-stopped
    ports:
      - '127.0.0.1:7880:7880'
    volumes:
      - /home/jhh/npm/data/logs:/opt/log:ro
    environment:
      - LOG_TYPE=NPM
      - TZ=Europe/Copenhagen
      - HTML_REFRESH=10
      - KEEP_LAST=30
```

## Log Source

- Host path: `/home/jhh/npm/data/logs`
- NPM docker-compose volume: `./data:/data` (logs at `./data/logs`)
- Files parsed: `proxy-host-*_access.log`, `proxy-host-*_access.log.*.gz`, `redirection-host-*_access.log*`
- 14 proxy hosts confirmed present, rotated archives included

## NPM Proxy Host Configuration

| Field | Value |
|---|---|
| Domain | `stats.yourdomain.com` |
| Scheme | `http` |
| Forward Hostname | `host.docker.internal` |
| Forward Port | `7880` |
| Websockets Support | ON |
| SSL | Let's Encrypt, Force SSL ON |
| Access List | Basic auth (username + password) |

## Reference

- Image: [xavier-hernandez/goaccess-for-nginxproxymanager](https://github.com/xavier-hernandez/goaccess-for-nginxproxymanager) — 695 stars, MIT, updated Dec 2025
