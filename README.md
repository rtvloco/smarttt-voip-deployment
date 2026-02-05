# SmartTT VoIP Kubernetes Deployment

Kubernetes manifests for the SmartTT VoIP system using Kustomize.

## Components

- **Signaling Service**: WebRTC signaling server (WebSocket + REST API)
- **SFU (Selective Forwarding Unit)**: Media server for video/audio routing
- **Coturn**: TURN/STUN server for NAT traversal
- **PostgreSQL**: Database for call records and user data
- **Redis**: Cache and pub/sub for real-time communication

## Architecture

```
┌─────────────────┐
│   Clients       │
│  (WebRTC)       │
└────────┬────────┘
         │
    ┌────▼────┐
    │ Coturn  │ (TURN/STUN)
    └────┬────┘
         │
    ┌────▼─────────┐
    │  Signaling   │ (WebSocket + API)
    └────┬─────────┘
         │
    ┌────▼────┐
    │   SFU   │ (Media routing)
    └────┬────┘
         │
    ┌────▼──────────┐
    │ PostgreSQL    │
    │ Redis         │
    └───────────────┘
```

## Deployment

### Prerequisites

1. Kubernetes cluster with ArgoCD installed
2. Sealed Secrets controller running
3. Traefik ingress controller
4. Cert-manager for TLS certificates

### Deploy with ArgoCD

```bash
# Apply the ArgoCD Application
kubectl apply -f argocd/staging.yaml

# Check deployment status
kubectl get applications -n argocd voip-staging

# Watch pods
kubectl get pods -n voip-staging -w
```

### Manual Deployment (for testing)

```bash
# Staging
kubectl apply -k overlays/staging

# Production
kubectl apply -k overlays/production
```

## Configuration

### Environment-Specific Values

Configuration is managed via Kustomize overlays:

- **base/**: Default configuration
- **overlays/staging/**: Staging environment (2 replicas, debug logging)
- **overlays/production/**: Production environment (3+ replicas, optimized resources)

### Secrets

Secrets are managed using Sealed Secrets:

```bash
# Create sealed secret for PostgreSQL
kubectl create secret generic postgres-credentials \
  --from-literal=POSTGRES_USER=voip \
  --from-literal=POSTGRES_PASSWORD=<password> \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > overlays/staging/sealed-postgres-credentials.yaml

# Create sealed secret for TURN server
kubectl create secret generic turn-credentials \
  --from-literal=TURN_SECRET=<secret> \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > overlays/staging/sealed-turn-credentials.yaml
```

## Accessing Services

### Staging

- Signaling API: `https://voip-staging.smarttt.cloud`
- WebSocket: `wss://voip-staging.smarttt.cloud/ws`

### Production

- Signaling API: `https://voip.smarttt.cloud`
- WebSocket: `wss://voip.smarttt.cloud/ws`

## Monitoring

Prometheus metrics are exposed at `/metrics` on each service:

- Signaling: `http://signaling-service:8082/metrics`
- SFU: `http://sfu-service:8081/metrics`

Grafana dashboards are available in the monitoring namespace.

## Troubleshooting

### Check pod status
```bash
kubectl get pods -n voip-staging
kubectl logs -n voip-staging deployment/signaling-service
kubectl logs -n voip-staging deployment/sfu-service
```

### Check service endpoints
```bash
kubectl get svc -n voip-staging
kubectl get ingress -n voip-staging
```

### Test connectivity
```bash
# Test signaling service health
kubectl port-forward -n voip-staging svc/signaling-service 8080:8080
curl http://localhost:8080/health

# Test SFU service
kubectl port-forward -n voip-staging svc/sfu-service 8081:8081
curl http://localhost:8081/health
```

## Development

### Local Testing

Use docker-compose for local development:

```bash
cd infra
docker-compose -f docker-compose.tier-c.yml up
```

### Building Images

Images are built by CI/CD pipeline and pushed to GHCR:

- `ghcr.io/rtvloco/voip-signaling:${VERSION}`
- `ghcr.io/rtvloco/voip-sfu:${VERSION}`

## Migration from Ansible

This repo replaces the Ansible-based deployment. Key differences:

| Aspect | Ansible | Kustomize (This Repo) |
|--------|---------|----------------------|
| Deployment | Manual `ansible-playbook` | GitOps (ArgoCD auto-sync) |
| Config | Ansible Vault | Sealed Secrets |
| Environments | Inventory files | Kustomize overlays |
| Updates | Re-run playbook | Git push (auto-deploys) |

## References

- [Application Code](https://github.com/rtvloco/smarttt-voip)
- [Infrastructure Repo](https://github.com/rtvloco/smarttt-infra)
- [Monitoring Stack](https://github.com/rtvloco/smarttt-monitoring)
