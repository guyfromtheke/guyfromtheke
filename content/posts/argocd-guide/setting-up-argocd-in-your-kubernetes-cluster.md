---
title: "Setting Up ArgoCD in Your Kubernetes Cluster"
date: 2025-04-27
draft: false
description: "A comprehensive guide to installing and configuring ArgoCD in a Kubernetes environment"
tags: ["kubernetes", "argocd", "gitops", "devops"]
categories: ["Infrastructure", "Kubernetes"]
series: ["GitOps Implementation"]
weight: 1
---

## What is ArgoCD?
ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. It automates the deployment of applications and helps maintain their desired state by continuously monitoring your Git repositories and ensuring your Kubernetes cluster matches the defined state in your Git configuration.

Key features:
- Automated deployment and sync from Git repositories
- Web UI and CLI for application management
- SSO Integration and RBAC
- Multi-cluster and multi-tenant support
- Health status analysis for Kubernetes resources

## Homelab Environment Overview

### Infrastructure Setup
- Single Kubernetes cluster with dedicated master and worker nodes
- Three-node setup:
  * One master node (control plane)
  * Two worker nodes for workload distribution
- Using K3s (v1.31.6+k3s1) as the Kubernetes distribution
- NGINX Ingress Controller for external access

### Network Architecture
- Internal network segment for cluster communication
- NodePort service exposure for external access
- HAProxy for load balancing (optional)

## Installation Process

### 1. Prerequisites Verification
```bash
# Verify cluster access and node status
kubectl get nodes
```

### 2. Creating ArgoCD Namespace
```bash
kubectl create namespace argocd
```

### 3. Deploying ArgoCD
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4. Accessing ArgoCD

#### Service Configuration
```bash
# Configure NodePort for access
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

#### Initial Access
- Default username: admin
- Password: Retrieved using:
  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
  ```

### 5. Ingress Configuration
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: 'true'
    nginx.ingress.kubernetes.io/backend-protocol: 'HTTPS'
    nginx.ingress.kubernetes.io/force-ssl-redirect: 'true'
spec:
  ingressClassName: nginx
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

## Challenges and Solutions

### Challenge 1: Initial Access Configuration
- Issue: Default ClusterIP service type limiting access
- Solution: Configured NodePort service type for direct access

### Challenge 2: Security Considerations
- Issue: Default admin password and insecure access
- Solution: 
  * Retrieved and documented initial admin password
  * Configured HTTPS access through Ingress
  * SSL passthrough enabled for secure communication

### Challenge 3: Architecture Best Practices
- Issue: Workload distribution and high availability
- Solution:
  * Proper node labeling
  * Resource allocation
  * Anti-affinity rules (covered in the optimization guide)

## Next Steps
1. Change the default admin password
2. Configure your Git repositories
3. Set up your first application
4. Implement SSO (optional)
5. Configure RBAC policies

## Advanced Configuration

### Implementing RBAC
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    g, developer-group, role:developer
```

### Setting Up SSO
- Configure OIDC providers
- Integrate with enterprise authentication
- Manage user sessions

### Backup and Disaster Recovery
```bash
# Backup ArgoCD settings
kubectl get -n argocd configmap,secret -o yaml > argocd-backup.yaml

# Backup application definitions
argocd admin export > applications-backup.yaml
```

### Monitoring Integration
- Prometheus metrics
- Grafana dashboards
- Alert configuration

### Security Hardening
1. Network Policies
2. Pod Security Policies
3. Secret Encryption
4. TLS Configuration

## Troubleshooting Guide

### Common Issues
1. Application sync failures
2. Authentication problems
3. Network connectivity issues
4. Resource constraints

### Debug Tools
```bash
# Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller

# View application status
argocd app get <application-name>

# Check sync issues
argocd app sync <application-name> --debug
```

## Extended Troubleshooting Guide

### Installation Issues

#### Pod Startup Failures
```bash
# Check pod status
kubectl get pods -n argocd

# Check pod events
kubectl describe pod <pod-name> -n argocd

# Check container logs
kubectl logs <pod-name> -n argocd
```

Common causes:
1. Resource constraints
2. Image pull failures
3. Volume mount issues
4. Node scheduling problems

#### Network Connectivity

1. Ingress Issues:
```bash
# Check ingress status
kubectl get ingress -n argocd
kubectl describe ingress argocd-ingress -n argocd

# Verify ingress controller
kubectl get pods -n ingress-nginx
```

2. Service Issues:
```bash
# Check service endpoints
kubectl get endpoints -n argocd

# Verify service configuration
kubectl describe svc argocd-server -n argocd
```

### Authentication Problems

#### RBAC Issues
```bash
# Verify RBAC configuration
kubectl get rolebindings -n argocd
kubectl get clusterrolebindings | grep argocd

# Check service account permissions
kubectl auth can-i --as system:serviceaccount:argocd:argocd-application-controller create deployments
```

#### SSO Configuration
```bash
# Check Dex logs
kubectl logs -l app.kubernetes.io/name=argocd-dex-server -n argocd

# Verify OIDC configuration
kubectl get cm argocd-cm -n argocd -o yaml
```

### Git Repository Access

#### SSH Key Issues
```bash
# Check SSH known hosts
kubectl get cm argocd-ssh-known-hosts-cm -n argocd

# Verify repository credentials
kubectl get secrets -n argocd | grep repo
```

#### HTTPS Access Problems
```bash
# Test repository connection
argocd repo test https://github.com/your-org/your-repo.git

# Check repository certificates
kubectl get cm argocd-tls-certs-cm -n argocd
```

### Performance Issues

#### Resource Usage
```bash
# Check resource usage
kubectl top pods -n argocd

# View detailed metrics
kubectl describe pod -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

#### Sync Performance
```bash
# Check sync times
argocd app list --output wide

# View application events
kubectl get events -n argocd --sort-by='.lastTimestamp'
```

### Recovery Procedures

#### Backup Current State
```bash
# Export all ArgoCD configurations
kubectl get all -n argocd -o yaml > argocd-backup.yaml

# Export specific resources
kubectl get applications -n argocd -o yaml > applications-backup.yaml
kubectl get appprojects -n argocd -o yaml > projects-backup.yaml
```

#### Restore Procedures
```bash
# Restore from backup
kubectl apply -f argocd-backup.yaml

# Verify restoration
kubectl get applications -n argocd
argocd app list
```

## Environment-Specific Notes

### K3s Considerations
- Default ingress controller differences
- Resource limitations
- Storage class configuration
- Load balancer implementation

### Homelab Setup
- Local DNS configuration
- Certificate management
- Backup strategies
- Network policies

## Conclusion
This setup provides a robust foundation for GitOps-based deployments using ArgoCD. The configuration ensures secure access, proper isolation, and follows Kubernetes best practices for production environments.

For detailed information about resource optimization and pod distribution, refer to the companion guide: [Optimizing Resource Limits for Pod Distribution for ArgoCD]({{< ref "/posts/argocd-guide/optimizing-resource-limits-for-pod-distribution-for-argocd" >}}).

