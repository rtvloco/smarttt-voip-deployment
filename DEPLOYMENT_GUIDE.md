# VoIP Deployment Guide

## ✅ Complete Kubernetes Manifests

The smarttt-voip-deployment repository now contains complete Kubernetes manifests converted from Ansible playbooks.

## Structure Created

```
smarttt-voip-deployment/
├── README.md                     ✅ Complete documentation
├── DEPLOYMENT_GUIDE.md           ✅ This file
├── .gitignore                    ✅ Prevents committing secrets
│
├── base/                         ✅ Base manifests (all components)
│   ├── namespace.yaml
│   ├── kustomization.yaml
│   ├── signaling/
│   │   ├── deployment.yaml      (WebRTC signaling server)
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   └── kustomization.yaml
│   ├── sfu/
│   │   ├── deployment.yaml      (Media server)
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── coturn/
│   │   ├── deployment.yaml      (TURN/STUN server)
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   └── kustomization.yaml
│   ├── postgres/
│   │   ├── deployment.yaml      (Database)
│   │   ├── service.yaml
│   │   ├── pvc.yaml
│   │   └── kustomization.yaml
│   └── redis/
│       ├── deployment.yaml      (Cache/Queue)
│       ├── service.yaml
│       └── kustomization.yaml
│
├── overlays/
│   ├── staging/                  ✅ Staging configuration
│   │   ├── kustomization.yaml   (2 replicas, staging images)
│   │   ├── signaling-patch.yaml
│   │   ├── sfu-patch.yaml
│   │   └── ingress.yaml         (voip-staging.smarttt.cloud)
│   │
│   └── production/               ✅ Production configuration
│       ├── kustomization.yaml   (3+ replicas, prod images)
│       ├── signaling-patch.yaml
│       ├── sfu-patch.yaml
│       ├── postgres-patch.yaml
│       ├── hpa.yaml             (Auto-scaling)
│       └── ingress.yaml         (voip.smarttt.cloud)
│
├── argocd/                       ✅ ArgoCD Applications
│   ├── staging.yaml
│   └── production.yaml
│
└── infra/                        (Kept for local testing)
    ├── docker-compose.tier-b.yml
    └── docker-compose.tier-c.yml
```

## Components Deployed

| Component | Image | Ports | Purpose |
|-----------|-------|-------|---------|
| **Signaling** | ghcr.io/rtvloco/voip-signaling | 8080 (WS), 8082 (API) | WebRTC signaling |
| **SFU** | ghcr.io/rtvloco/voip-sfu | 8081, 5000, 10000-20000 | Media routing |
| **Coturn** | coturn/coturn | 3478, 5349 | TURN/STUN for NAT |
| **PostgreSQL** | postgres:15-alpine | 5432 | Database |
| **Redis** | redis:7-alpine | 6379 | Cache/PubSub |

## Prerequisites: Create Sealed Secrets

Before deploying, create sealed secrets for credentials:

### PostgreSQL Credentials

```bash
cd ~/DEV/smarttt-voip-deployment/overlays/staging

# Create PostgreSQL secret
kubectl create secret generic postgres-credentials \
  --from-literal=POSTGRES_USER=voip \
  --from-literal=POSTGRES_PASSWORD='<CHANGE_ME>' \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-postgres-credentials.yaml

# Add to kustomization.yaml resources:
# - sealed-postgres-credentials.yaml
```

### TURN Server Credentials

```bash
# Create TURN secret
kubectl create secret generic turn-credentials \
  --from-literal=TURN_SECRET='<CHANGE_ME>' \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-turn-credentials.yaml

# Add to kustomization.yaml resources:
# - sealed-turn-credentials.yaml
```

### Update Configuration Values

Edit `overlays/staging/kustomization.yaml`:

```yaml
configMapGenerator:
- name: signaling-config
  behavior: merge
  literals:
  - TURN_REALM=voip-staging.smarttt.cloud
  - TURN_EXTERNAL_IP=<ACTUAL_EXTERNAL_IP>  # Get from coturn LoadBalancer
```

## Deployment Methods

### Option 1: ArgoCD (Recommended)

```bash
# Apply ArgoCD Application
kubectl apply -f argocd/staging.yaml

# Watch deployment
argocd app get voip-staging
kubectl get pods -n voip-staging -w
```

### Option 2: Direct with Kustomize

```bash
# Staging
kubectl apply -k overlays/staging

# Production
kubectl apply -k overlays/production
```

### Option 3: Preview Changes

```bash
# Preview what will be deployed
kubectl kustomize overlays/staging

# Show diff
kubectl diff -k overlays/staging
```

## Post-Deployment Steps

### 1. Get TURN Server External IP

```bash
# Wait for LoadBalancer to get external IP
kubectl get svc -n voip-staging coturn-service

# Update TURN_EXTERNAL_IP in kustomization.yaml with this IP
```

### 2. Verify All Pods Running

```bash
kubectl get pods -n voip-staging

# Expected pods:
# - postgres-*
# - redis-*
# - signaling-service-*-* (2 replicas in staging)
# - sfu-service-*-* (2 replicas in staging)
# - coturn-*
```

### 3. Check Services

```bash
kubectl get svc -n voip-staging

# Test signaling health endpoint
kubectl port-forward -n voip-staging svc/signaling-service 8080:8080
curl http://localhost:8080/health
```

### 4. Check Ingress

```bash
kubectl get ingress -n voip-staging

# Test public endpoint
curl https://voip-staging.smarttt.cloud/health
```

## Migration from Ansible

### What Changed

| Aspect | Ansible (Old) | Kustomize (New) |
|--------|--------------|-----------------|
| Deployment | `ansible-playbook deploy.yml` | ArgoCD auto-sync |
| Secrets | Ansible Vault | Sealed Secrets |
| Config | Inventory files | Kustomize overlays |
| Updates | Re-run playbook | Git push → auto-deploy |
| Rollback | Ansible playbook | ArgoCD rollback or Git revert |

### Key Differences

1. **GitOps**: All changes through Git, no manual `kubectl apply`
2. **Auto-sync**: ArgoCD watches Git and auto-deploys
3. **Self-healing**: If pods deleted, ArgoCD recreates them
4. **Declarative**: State defined in Git, not imperative commands

## Updating the Deployment

### Change Image Version

```yaml
# In overlays/staging/kustomization.yaml
images:
- name: ghcr.io/rtvloco/voip-signaling
  newTag: v1.2.3  # Change this
```

Commit and push → ArgoCD auto-deploys.

### Change Replicas

```yaml
# In overlays/staging/signaling-patch.yaml
spec:
  replicas: 3  # Change from 2 to 3
```

### Change Resources

```yaml
# In overlays/production/signaling-patch.yaml
resources:
  requests:
    cpu: 1000m  # Increase from 500m
    memory: 1Gi
```

## Rollback

### ArgoCD UI Rollback

```bash
# List history
argocd app history voip-staging

# Rollback to previous revision
argocd app rollback voip-staging <revision-id>
```

### Git Revert

```bash
cd ~/DEV/smarttt-voip-deployment
git log --oneline
git revert <commit-hash>
git push
# ArgoCD will sync automatically
```

## Monitoring

### Prometheus Metrics

Services expose metrics at:
- Signaling: `http://signaling-service:8082/metrics`
- SFU: `http://sfu-service:8081/metrics`

### Grafana Dashboards

Import VoIP dashboards in Grafana (monitoring namespace).

### Logs

```bash
# Signaling logs
kubectl logs -n voip-staging -l app=signaling-service --tail=100 -f

# SFU logs
kubectl logs -n voip-staging -l app=sfu-service --tail=100 -f

# All VoIP logs
kubectl logs -n voip-staging -l component=voip --tail=100 -f
```

## Troubleshooting

### Pods Not Starting

```bash
kubectl describe pod -n voip-staging <pod-name>
kubectl logs -n voip-staging <pod-name>
```

### Database Connection Issues

```bash
# Test PostgreSQL connection
kubectl exec -n voip-staging -it deploy/postgres -- psql -U voip -d voip

# Check if database exists
kubectl exec -n voip-staging deploy/postgres -- psql -U voip -c '\l'
```

### TURN Server Issues

```bash
# Check coturn logs
kubectl logs -n voip-staging deploy/coturn

# Verify external IP assigned
kubectl get svc -n voip-staging coturn-service

# Test TURN connectivity (from external client)
# Use: https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/
```

### Signaling Health Check Failing

```bash
# Port-forward and test locally
kubectl port-forward -n voip-staging svc/signaling-service 8080:8080
curl -v http://localhost:8080/health

# Check if connected to database
kubectl logs -n voip-staging -l app=signaling-service | grep -i postgres
```

## Next Steps

1. ✅ Commit changes to smarttt-voip-deployment repo
2. ✅ Create sealed secrets
3. ✅ Apply ArgoCD Application
4. ⏳ Monitor deployment
5. ⏳ Update DNS (voip-staging.smarttt.cloud → Ingress)
6. ⏳ Test WebRTC calls
7. ⏳ Deploy to production

## References

- [README.md](README.md) - Overview and quick start
- [smarttt-infra](https://github.com/rtvloco/smarttt-infra) - Infrastructure repo
- [smarttt-voip](https://github.com/rtvloco/smarttt-voip) - Application code
- [Kustomize Documentation](https://kustomize.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
