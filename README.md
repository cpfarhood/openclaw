# OpenClaw Deployment

AI assistant that connects to messaging platforms and executes tasks autonomously.

## Architecture

- **Main Container:** OpenClaw gateway (Node.js)
- **Sidecar:** Headless Chromium for browser automation
- **Storage:** 5Gi PVC on Rook Ceph for workspace/sessions
- **Network:** Internal HTTPRoute via Cilium Gateway API

## Prerequisites

1. **Anthropic API Key** - Required for Claude models
2. **Messaging Platform Tokens** - Optional (Telegram, Discord, Slack)
3. **Sealed Secrets** - Controller running in cluster

## Quick Start

### 1. Create Secrets

```bash
# Create sealed secret with your API keys
kubectl create secret generic openclaw-env-secret \
  --namespace=openclaw \
  --from-literal=ANTHROPIC_API_KEY='sk-ant-xxx' \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > secrets/sealedsecret-openclaw-env.yaml

# Optional: Add messaging platform tokens
# --from-literal=TELEGRAM_BOT_TOKEN='xxx' \
# --from-literal=DISCORD_BOT_TOKEN='xxx' \
# --from-literal=SLACK_BOT_TOKEN='xoxb-xxx' \
# --from-literal=SLACK_APP_TOKEN='xapp-xxx'
```

### 2. Configure Trusted Proxies

Edit `helmrelease/helmrelease-openclaw.yaml` and update the `trustedProxies` IP to match your Cilium/Traefik LoadBalancer CIDR.

### 3. Deploy

```bash
# Apply to cluster (via GitOps or directly)
kubectl apply -f namespace/
kubectl apply -f secrets/sealedsecret-openclaw-env.yaml
kubectl apply -f helmrelease/
kubectl apply -f httproute/
```

### 4. Access

- **Web UI:** https://openclaw.animaniacs.farh.net
- **API:** Port 18789

## Configuration

### OpenClaw Config (`openclaw.json`)

The main configuration is in the HelmRelease `values.configMaps.config.data.openclaw.json`:

- **Gateway:** Port, trusted proxies
- **Browser:** Chromium CDP settings
- **Agents:** Claude model, timezone, concurrency
- **Session:** Storage and reset policy
- **Logging:** Levels and format
- **Tools:** Web search/fetch capabilities

### Environment Variables (Secrets)

Sensitive values are injected via the `openclaw-env-secret`:

- `ANTHROPIC_API_KEY` - **Required**
- `TELEGRAM_BOT_TOKEN` - Optional
- `DISCORD_BOT_TOKEN` - Optional
- `SLACK_BOT_TOKEN` - Optional
- `SLACK_APP_TOKEN` - Optional

### Messaging Platform Setup

To enable channels, add configuration to `openclaw.json`:

```json
"channels": {
  "telegram": {
    "botToken": "${TELEGRAM_BOT_TOKEN}",
    "enabled": true
  },
  "discord": {
    "token": "${DISCORD_BOT_TOKEN}",
    "enabled": true
  },
  "slack": {
    "botToken": "${SLACK_BOT_TOKEN}",
    "appToken": "${SLACK_APP_TOKEN}",
    "enabled": true
  }
}
```

Environment variable substitution (`${VAR}`) pulls from the sealed secret.

## Storage

- **PVC Size:** 5Gi (Rook Ceph Block)
- **Access Mode:** ReadWriteOnce (single pod)
- **Paths:**
  - `/home/node/.openclaw/workspace` - Agent workspace
  - `/home/node/.openclaw/sessions` - Session state
  - `/home/node/.openclaw/openclaw.json` - Runtime config

## Skills

OpenClaw supports installing skills from [ClawHub](https://clawhub.com).

Edit the `init-skills` container in the HelmRelease to add skills:

```bash
for skill in weather <other-skills>; do
  ...
done
```

## Security

- **Pod Security:** Restricted profile enforced
- **Non-root:** Runs as UID 1000
- **Read-only FS:** Containers use read-only root filesystem
- **No privileges:** All capabilities dropped

## Resource Limits

| Container | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| Main      | 200m        | 2000m     | 512Mi          | 2Gi          |
| Chromium  | 100m        | 1000m     | 256Mi          | 1Gi          |

## Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n openclaw
kubectl logs -n openclaw -l app.kubernetes.io/name=openclaw -c main
kubectl logs -n openclaw -l app.kubernetes.io/name=openclaw -c chromium
```

### Check Config
```bash
kubectl exec -n openclaw -it deployment/openclaw -- cat /home/node/.openclaw/openclaw.json
```

### Check Browser
```bash
# Chromium CDP should respond on localhost:9222
kubectl exec -n openclaw -it deployment/openclaw -- curl -s http://localhost:9222/json/version
```

## References

- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [Helm Chart](https://github.com/serhanekicii/openclaw-helm)
- [Documentation](https://serhanekici.com/openclaw-helm.html)
- [ClawHub Skills](https://clawhub.com)
