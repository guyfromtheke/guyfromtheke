---
title: "ArgoCD Quick Reference Guide"
date: 2025-04-27
draft: false
description: "Quick reference guide for common ArgoCD operations and configurations"
tags: ["kubernetes", "argocd", "gitops", "devops", "quick-reference"]
categories: ["Infrastructure", "Kubernetes"]
series: ["GitOps Implementation"]
weight: 5
---

# ArgoCD Quick Reference Guide

## Common Operations

### Installation
```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access the UI
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Application Management
```bash
# Create new application
argocd app create myapp \
  --repo https://github.com/your-org/your-app.git \
  --path kustomize \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# Sync application
argocd app sync myapp

# Get application status
argocd app get myapp

# Delete application
argocd app delete myapp
```

### User Management
```bash
# Add cluster admin
argocd account update-password \
  --current-password <initial-password> \
  --new-password <new-password>

# Create project
argocd proj create myproject \
  --description "My new project" \
  --src https://github.com/your-org/* \
  --dest https://kubernetes.default.svc,default
```

## Quick Troubleshooting

### Health Checks
```bash
# Check pod health
kubectl get pods -n argocd

# Check logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server

# Verify sync status
argocd app get myapp --refresh
```

### Common Issues & Solutions

#### Sync Failed
1. Check application logs:
   ```bash
   argocd app logs myapp
   ```
2. Verify Git credentials:
   ```bash
   argocd repo list
   ```
3. Test repository access:
   ```bash
   argocd repo test https://github.com/your-org/your-app.git
   ```

#### Access Issues
1. Reset admin password:
   ```bash
   argocd admin initial-password -n argocd
   ```
2. Check RBAC:
   ```bash
   kubectl get cm argocd-rbac-cm -n argocd -o yaml
   ```

## Quick Configurations

### SSL/TLS Setup
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-cert
  namespace: argocd
spec:
  secretName: argocd-server-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - argocd.your-domain.com
```

### Basic Auth Integration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  accounts.admin: apiKey,login
```

## Environment-Specific Examples

### Development
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-app
spec:
  source:
    repoURL: https://github.com/your-org/your-app.git
    targetRevision: develop
    path: k8s/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: development
```

### Production
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prod-app
spec:
  source:
    repoURL: https://github.com/your-org/your-app.git
    targetRevision: main
    path: k8s/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Network Configurations

### Ingress Example
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  rules:
  - host: argocd.your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```

### Network Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: argocd-server-network-policy
  namespace: argocd
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: argocd-server
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
```

## Monitoring Quick Setup

### Prometheus Rules
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: argocd-rules
  namespace: monitoring
spec:
  groups:
  - name: argocd.rules
    rules:
    - alert: ArgoCDSyncFailed
      expr: argocd_app_sync_status{status="Failed"} > 0
      for: 5m
      labels:
        severity: critical
```

## Next Steps

For detailed information, refer to:
- [Complete Setup Guide]({{< ref "/posts/argocd-guide/setting-up-argocd-in-your-kubernetes-cluster" >}})
- [Optimization Guide]({{< ref "/posts/argocd-guide/optimizing-resource-limits-for-pod-distribution-for-argocd" >}})
- [Real-World Scenarios]({{< ref "/posts/argocd-guide/argocd-real-world-scenarios" >}})
- [Integration Patterns]({{< ref "/posts/argocd-guide/argocd-integration-patterns" >}})

