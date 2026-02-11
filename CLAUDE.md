# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains Kubernetes manifests for deploying OpenClaw - an AI assistant powered by Google Gemini that connects to messaging platforms and executes tasks autonomously. The deployment uses:

- **FluxCD** for GitOps continuous deployment
- **Helm** via HelmRelease for configuration management
- **Cilium Gateway API** for internal routing
- **Sealed Secrets** for secure credential management
- **Rook Ceph** for persistent storage

OpenClaw runs as a single-pod deployment (not horizontally scalable) with two containers:
1. Main container: Node.js gateway (OpenClaw)
2. Sidecar: Headless Chromium for browser automation

## Repository Structure

```
.
├── namespace/           # Namespace definition with pod security enforcement
├── helmrepository/      # FluxCD HelmRepository source definition
├── helmrelease/         # Main HelmRelease with all OpenClaw configuration
├── httproute/          # Cilium Gateway API HTTPRoute for ingress
└── secrets/            # SealedSecret examples (actual secrets not committed)
```

## Common Commands

### Deployment

```bash
# Deploy all resources (via GitOps or direct apply)
kubectl apply -f namespace/
kubectl apply -f secrets/sealedsecret-openclaw-env.yaml  # After creating
kubectl apply -f helmrelease/
kubectl apply -f httproute/
```

### Creating Sealed Secrets

```bash
# Create SealedSecret with API keys
kubectl create secret generic openclaw-env-secret \
  --namespace=openclaw \
  --from-literal=GOOGLE_GENERATIVE_AI_API_KEY='your-gemini-api-key' \
  --from-literal=TELEGRAM_BOT_TOKEN='xxx' \
  --from-literal=DISCORD_BOT_TOKEN='xxx' \
  --from-literal=SLACK_BOT_TOKEN='xoxb-xxx' \
  --from-literal=SLACK_APP_TOKEN='xapp-xxx' \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > secrets/sealedsecret-openclaw-env.yaml
```

### Monitoring and Debugging

```bash
# Check deployment status
kubectl get pods -n openclaw
kubectl get helmrelease -n openclaw
kubectl get httproute -n openclaw

# View logs
kubectl logs -n openclaw -l app.kubernetes.io/name=openclaw -c main
kubectl logs -n openclaw -l app.kubernetes.io/name=openclaw -c chromium

# Verify configuration
kubectl exec -n openclaw -it deployment/openclaw -- cat /home/node/.openclaw/openclaw.json

# Check Chromium CDP
kubectl exec -n openclaw -it deployment/openclaw -- curl -s http://localhost:9222/json/version

# Check persistent volume
kubectl exec -n openclaw -it deployment/openclaw -- ls -la /home/node/.openclaw/
```

## Key Configuration Points

### HelmRelease Values (`helmrelease/helmrelease-openclaw.yaml`)

The HelmRelease contains the entire OpenClaw configuration in `spec.values.configMaps.config.data.openclaw.json`:

- **Gateway settings**: Port (18789), trusted proxies (must match your LoadBalancer CIDR)
- **Browser profiles**: Chromium CDP URL (http://localhost:9222)
- **Agent configuration**: Gemini model (`google/gemini-2.0-flash-exp`), timezone, concurrency
- **Session management**: Per-sender scope, 60-minute idle reset
- **Tool capabilities**: Web fetch enabled, web search disabled by default
- **Channel configuration**: Add messaging platform configs with `${ENV_VAR}` substitution from sealed secrets

### Config Mode

The deployment uses `configMode: merge` which preserves runtime changes. Change to `overwrite` for strict GitOps (discards manual changes on each reconciliation).

### Trusted Proxies

**IMPORTANT**: Update `gateway.trustedProxies` in the HelmRelease to match your Cilium/Traefik LoadBalancer IP range. Default is `["10.42.0.1"]`.

### Storage

- 5Gi PVC on `rook-ceph-block` storage class
- Single ReadWriteOnce access mode (OpenClaw is NOT horizontally scalable)
- Mounted at `/home/node/.openclaw/` containing workspace, sessions, and config

## Security Model

- Pod security: **Restricted** profile enforced at namespace level
- Containers run as non-root (UID 1000)
- Read-only root filesystem
- All Linux capabilities dropped
- Secrets managed via Bitnami SealedSecrets (safe to commit encrypted values)

## Adding Messaging Platform Support

To enable Telegram/Discord/Slack:

1. Add tokens to the SealedSecret (see "Creating Sealed Secrets" above)
2. Edit `openclaw.json` in the HelmRelease to add channel configuration:

```json
"channels": {
  "telegram": {
    "botToken": "${TELEGRAM_BOT_TOKEN}",
    "enabled": true
  }
}
```

Environment variables are substituted from the sealed secret at runtime.

## HTTPRoute Configuration

The HTTPRoute (`httproute/httproute-openclaw.yaml`) routes traffic from the internal Cilium Gateway:

- ParentRef: `internal` gateway in `gateway-system` namespace
- Hostname: `openclaw.animaniacs.farh.net`
- Backend: OpenClaw service on port 18789

To change the domain or gateway, edit this file.

## Important Constraints

1. **Single replica only**: OpenClaw is not designed for horizontal scaling. The deployment uses `strategy: Recreate` to ensure only one pod runs at a time.

2. **FluxCD reconciliation**: Changes to files in this repo will be automatically applied by FluxCD HelmRelease controller (interval: 1h, timeout: 15m).

3. **Storage class**: Adjust `storageClass: rook-ceph-block` in the HelmRelease if your cluster uses a different storage provider.

4. **Resource limits**: Main container can use up to 2 CPU cores and 2Gi memory; Chromium sidecar up to 1 CPU core and 1Gi memory.
