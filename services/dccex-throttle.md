# DCC-EX Throttle

## Summary

Browser-based DCC-EX locomotive throttle with a server-side WebSocket-to-TCP proxy.
DCC-EX speaks raw TCP; this app bridges the gap so a browser can control a layout
over Cloudflare Tunnel without needing any local software.

## Repositories

| Artifact | Repository | Path |
|----------|-----------|------|
| App source | `~/github/dccex-throttle` | `github.com/rkamradt/dccex-throttle` |
| Helm chart | `~/github/rkamradt-helm-charts` | `dccex-throttle/` |
| ArgoCD Application | `~/github/rkamradt-helm-charts` | `apps/values.yaml` (app-of-apps entry) |

## Architecture

```
Browser (Cloudflare Tunnel)
  └─ HTTP GET /          → Express serves public/index.html
  └─ WS  ws[s]://host/ws → WebSocketServer
                              └─ TCP → DCC-EX command station (DCCEX_HOST:DCCEX_PORT)
```

The browser connects to `/ws` on the same origin it was served from, so it works
transparently through Cloudflare Tunnel without any special WebSocket configuration.

## Configuration

| Env Var | Default | Description |
|---------|---------|-------------|
| `DCCEX_HOST` | `10.147.0.8` | IP of the DCC-EX command station on the home LAN |
| `DCCEX_PORT` | `2560` | TCP port DCC-EX listens on |
| `PORT` | `3000` | HTTP/WS port the Node.js server binds to |

The K3s cluster is on the same home WiFi network as the DCC-EX station, so
`DCCEX_HOST` is reachable directly by IP from within the cluster.

## Deployment

ArgoCD is managed via the app-of-apps chart at `rkamradt-helm-charts/apps/values.yaml`.
The `dccex-throttle` entry there points ArgoCD at `rkamradt-helm-charts/dccex-throttle/`
and deploys to the `dccex-throttle` namespace.

Image is built and pushed by GitHub Actions on every push to `main` in the
`dccex-throttle` repo → `ghcr.io/rkamradt/dccex-throttle:sha-<SHA>`.

Update `image.tag` in `rkamradt-helm-charts/dccex-throttle/values.yaml` to roll out
a new version, then commit and push — ArgoCD will sync automatically.

## Ingress / Cloudflare Tunnel

An `Ingress` resource (Traefik) is deployed with hostname `dccex.rkamradt.dev`
(configure in `rkamradt-helm-charts/dccex-throttle/values.yaml`).

Point your Cloudflare Tunnel public hostname at:
```
http://dccex-throttle.dccex-throttle.svc.cluster.local:3000
```
or at the Traefik ingress controller's ClusterIP/NodePort.

## Working on this service

```bash
# Open the app in your editor
cd ~/github/dccex-throttle

# Run locally (requires DCC-EX reachable from your machine)
DCCEX_HOST=10.147.0.8 DCCEX_PORT=2560 npm start
# then open http://localhost:3000

# Update the helm chart
cd ~/github/rkamradt-helm-charts
# edit dccex-throttle/values.yaml or templates/

# Deployment is managed by the app-of-apps — no manual kubectl apply needed.
# ArgoCD picks it up from rkamradt-helm-charts/apps/values.yaml automatically.
```
