---
title: "Optimizing Resource Limits and Pod Distribution for ArgoCD"
date: 2025-04-27
draft: false
description: "Advanced guide for optimizing ArgoCD deployment with proper resource management and high availability"
tags: ["kubernetes", "argocd", "gitops", "devops", "optimization"]
categories: ["Infrastructure", "Kubernetes"]
series: ["GitOps Implementation"]
weight: 2
---

# Optimizing Resource Limits and Pod Distribution for ArgoCD

## Overview
This guide details the process of optimizing an ArgoCD deployment in Kubernetes by implementing proper resource limits and ensuring optimal pod distribution across worker nodes. These optimizations enhance reliability, performance, and high availability of the ArgoCD installation.

## Initial Environment Analysis

### Cluster Configuration
- Kubernetes Cluster (K3s v1.31.6+k3s1)
- Architecture:
  * 1 master node (control plane)
  * 2 worker nodes
- Initial state: Pods distributed across all nodes including master

## Master Node Protection

### Implementing Control Plane Isolation
```bash
# Adding taint to master node
kubectl taint nodes <master-node> node-role.kubernetes.io/control-plane=:NoSchedule
```

### Purpose of Master Node Taint
- Prevents regular workloads from scheduling on the control plane
- Reserves master node resources for critical cluster operations
- Enforces best practices for Kubernetes architecture

## Resource Limits Configuration

### Resource Allocation by Component

#### Application Controller
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

#### API Server
```yaml
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

#### Repository Server
```yaml
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

#### Dex Server
```yaml
resources:
  requests:
    cpu: 50m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

#### Redis
```yaml
resources:
  requests:
    cpu: 50m
    memory: 32Mi
  limits:
    cpu: 100m
    memory: 64Mi
```

#### ApplicationSet Controller
```yaml
resources:
  requests:
    cpu: 50m
    memory: 32Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

#### Notifications Controller
```yaml
resources:
  requests:
    cpu: 50m
    memory: 32Mi
  limits:
    cpu: 100m
    memory: 128Mi
```

## Pod Distribution Strategy

### Anti-Affinity Rules
```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - <component-name>
        topologyKey: kubernetes.io/hostname
```

### Final Pod Distribution

#### Worker Node 1:
- argocd-repo-server
- argocd-server

#### Worker Node 2:
- argocd-application-controller
- argocd-applicationset-controller
- argocd-dex-server
- argocd-notifications-controller
- argocd-redis

## Optimization Results

### Resource Management Benefits
1. Predictable resource usage
2. Prevention of resource contention
3. Efficient resource allocation
4. Protection against memory/CPU exhaustion

### High Availability Improvements
1. Proper workload distribution
2. Enhanced fault tolerance
3. Better resource utilization
4. Reduced single point of failure risk

### Performance Metrics
- CPU usage maintained within defined limits
- Memory consumption optimized
- Improved response times
- Better overall cluster stability

## Monitoring and Maintenance

### Health Checks
```bash
# View pod distribution
kubectl get pods -n argocd -o wide

# Check resource usage
kubectl top pods -n argocd
```

### Resource Adjustment
Monitor resource usage and adjust limits based on:
- Actual usage patterns
- Application load
- Cluster capacity
- Performance requirements

## Best Practices Summary

1. Control Plane Protection
   - Master node tainted
   - Workloads on worker nodes only
   - Critical services protected

2. Resource Management
   - Appropriate limits set
   - Resources allocated based on component needs
   - Buffer for spike handling

3. High Availability
   - Pod anti-affinity rules
   - Distributed workload
   - Failure domain isolation

## Advanced Troubleshooting

### Resource Management Issues

#### Pod Eviction Problems
```bash
# Check node resources
kubectl describe node <node-name>

# View pod resource usage
kubectl top pods -n argocd --containers

# Check eviction events
kubectl get events -n argocd --field-selector reason=Evicted
```

Common causes:
1. Memory pressure
2. CPU throttling
3. Storage pressure
4. Node pressure

#### Resource Limit Violations
```bash
# Check pod QoS class
kubectl get pods -n argocd -o custom-columns="NAME:.metadata.name,QOS:.status.qosClass"

# View resource quotas
kubectl describe resourcequota -n argocd

# Monitor resource usage trends
kubectl top pods -n argocd --containers --sort-by=cpu
```

### Pod Distribution Issues

#### Node Affinity Problems
```bash
# Check node labels
kubectl get nodes --show-labels

# Verify pod placement
kubectl get pods -n argocd -o wide

# Debug scheduler decisions
kubectl describe pod <pod-name> -n argocd | grep -A 10 Events
```

#### Taint and Toleration Issues
```bash
# List node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check pod tolerations
kubectl get pod <pod-name> -n argocd -o jsonpath='{.spec.tolerations}'
```

### K3s-Specific Optimizations

#### Worker Node Management
```bash
# Check K3s service status
sudo systemctl status k3s-agent

# View K3s logs
sudo journalctl -u k3s-agent

# Monitor node conditions
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[*].type
```

#### Storage Issues
```bash
# Check default storage class
kubectl get sc

# View persistent volumes
kubectl get pv,pvc -n argocd

# Monitor storage usage
kubectl describe nodes | grep "Allocated resources"
```

### High Availability Troubleshooting

#### Pod Distribution Check
```bash
# View pod spread
kubectl get pods -n argocd -o wide --sort-by=.spec.nodeName

# Check anti-affinity rules
kubectl get pods -n argocd -o yaml | grep -A 10 affinity
```

#### Load Balancing Issues
```bash
# Check service endpoints
kubectl get endpoints -n argocd

# View service distribution
kubectl describe svc argocd-server -n argocd
```

### Performance Optimization

#### CPU Optimization
```bash
# Monitor CPU throttling
kubectl get pods -n argocd -o custom-columns="NAME:.metadata.name,CPU_REQUESTS:.spec.containers[*].resources.requests.cpu,CPU_LIMITS:.spec.containers[*].resources.limits.cpu"

# Check CPU pressure
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
```

#### Memory Management
```bash
# View memory usage
kubectl top pods -n argocd --containers

# Check memory limits
kubectl get pods -n argocd -o custom-columns="NAME:.metadata.name,MEMORY_REQUESTS:.spec.containers[*].resources.requests.memory,MEMORY_LIMITS:.spec.containers[*].resources.limits.memory"
```

### Network Optimization

#### Network Policy Verification
```bash
# List network policies
kubectl get networkpolicies -n argocd

# Test network connectivity
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash
```

#### Service Mesh Integration
```bash
# Check proxy sidecars
kubectl get pods -n argocd -o jsonpath='{.items[*].spec.containers[*].name}'

# Verify mesh configuration
kubectl describe pod -n argocd -l app.kubernetes.io/name=argocd-server
```

### Recovery and Rollback

#### Configuration Backup
```bash
# Backup resource configurations
kubectl get deployment,statefulset -n argocd -o yaml > argocd-workloads.yaml

# Export current settings
kubectl get configmap -n argocd -o yaml > argocd-config.yaml
```

#### Rollback Procedures
```bash
# Revert to previous configuration
kubectl apply -f argocd-workloads.yaml

# Verify rollback
kubectl get pods -n argocd
```

### Monitoring and Alerts

#### Prometheus Integration
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
spec:
  endpoints:
  - port: metrics
    interval: 30s
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
```

#### Alert Configuration
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: argocd-alerts
spec:
  groups:
  - name: argocd
    rules:
    - alert: HighResourceUsage
      expr: container_memory_usage_bytes{namespace="argocd"} > 1Gi
      for: 5m
      labels:
        severity: warning
```

## Conclusion
These optimizations ensure a robust, reliable, and efficient ArgoCD deployment. The configuration provides:
- Protected control plane
- Efficient resource utilization
- High availability
- Proper workload distribution
- Sustainable scaling capability

Regular monitoring and adjustment of these settings ensure continued optimal performance as your GitOps implementation grows.

Return to the [main guide]({{< ref "/posts/argocd-guide/_index" >}}) for complete ArgoCD implementation documentation.

