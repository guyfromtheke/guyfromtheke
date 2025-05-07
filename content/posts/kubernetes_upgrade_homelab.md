---
title: "How I Managed Kubernetes Upgrade in My Homelab Environment"
date: 2025-05-07
draft: false
weight: -100
summary: "A detailed guide on upgrading a K3s Kubernetes cluster in my homelab from v1.31.6 to v1.32.3"
tags: ["kubernetes", "homelab", "upgrade", "k3s"]
categories: ["Infrastructure", "Kubernetes", "K3S" ]
featured: true
---

# How I Managed Kubernetes Upgrade in My Homelab Environment

## Introduction

In this blog post, I'll share my experience upgrading a K3s Kubernetes cluster in a homelab environment from v1.31.6 to v1.32.3. This upgrade involved a master node and three worker nodes, presenting various challenges and learning opportunities along the way. I'll also add to the fact that i upgraded my internal dns infrastructure so that i resolve all my nodes in my homelab to resolve to gftke.local domain 

## Environment Overview

My homelab Kubernetes cluster consisted of:
- 1 Master Node: k8s-master.gftke.local (v1.31.6+k3s1)
- 3 Worker Nodes:
  - k8s-worker1.gftke.local (v1.31.6+k3s1)
  - k8s-worker2.gftke.local (v1.31.6+k3s1)
  - k8s-worker3.gftke.local (already on v1.32.3+k3s1)

Running workloads included:
- ArgoCD
- Ingress-Nginx
- PostgreSQL database
- Various system services

## Upgrade Strategy

The upgrade followed these key principles:
1. Minimize downtime
2. Maintain data integrity
3. Handle one node at a time
4. Have a rollback plan

### Pre-upgrade Checks

Before starting the upgrade, I performed several crucial checks:
- Verified all nodes' status -> kubect get nodes -o wide 
- Documented running workloads
- Checked PersistentVolumes and Claims
- Cleaned up any stale resources

## Challenges Faced and Solutions

### 1. Stale Node References

**Challenge:** The cluster had stale node entries (uk8s-*) showing as NotReady, which could interfere with the upgrade process.

**Solution:** 
```bash
# Removed stale nodes
kubectl delete node uk8s-master uk8s-worker uk8s-worker2
```

### 2. Stuck Terminating Pods

**Challenge:** Several pods were stuck in Terminating state after node drains.

**Solution:** 
```bash
# Force deleted stuck pods
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

### 3. Node Communication Issues

**Challenge:** Workers couldn't connect to the master node due to incorrect FQDN resolution.

**Solution:** 
- Used correct FQDN (k8s-master.gftke.local instead of k8s-master.homelab.local)
- Verified connectivity before upgrade:
```bash
ping -c1 k8s-master.gftke.local
```

### 4. PostgreSQL StatefulSet Issues

**Challenge:** PostgreSQL pod wouldn't schedule due to volume node affinity conflicts.

**Solution:**
- Cleaned up old StatefulSet and PVC
- Recreated with proper storage configuration
- Ensured proper node selection

## Upgrade Process

### 1. Master Node Upgrade

```bash
# Drain master node
kubectl drain k8s-master.gftke.local --ignore-daemonsets --delete-emptydir-data

# Upgrade K3s
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.32.3+k3s1" sh -

# Uncordon master
kubectl uncordon k8s-master.gftke.local
```

### 2. Worker Nodes Upgrade

For each worker:
```bash
# Drain worker
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data

# Stop k3s agent
systemctl stop k3s-agent

# Install new version
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION='v1.32.3+k3s1' \
  K3S_URL='https://k8s-master.gftke.local:6443' \
  K3S_TOKEN='<node-token>' \
  sh -s - agent

# Uncordon worker
kubectl uncordon <worker-node>
```

## Post-Upgrade Verification

After the upgrade, all nodes were running v1.32.3+k3s1:
```bash
$ kubectl get nodes -o wide
NAME                        STATUS   ROLES                  AGE   VERSION
k8s-master.gftke.local    Ready    control-plane,master   42h   v1.32.3+k3s1
k8s-worker1.gftke.local   Ready    <none>                 42h   v1.32.3+k3s1
k8s-worker2.gftke.local   Ready    <none>                 42h   v1.32.3+k3s1
k8s-worker3.gftke.local     Ready    <none>                 2d    v1.32.3+k3s1
```

## Lessons Learned

1. **Domain Name Resolution**: Always verify and use correct FQDNs for node communication.
2. **Clean State**: Remove stale resources before starting the upgrade.
3. **Stateful Applications**: Take extra care with StatefulSets and persistent storage.
4. **Node Token**: Ensure proper node token handling for worker registration.
5. **Incremental Upgrade**: Upgrading one node at a time minimizes risk.

## Best Practices Established

1. Always verify node connectivity before upgrade
2. Document the current state of the cluster
3. Have a rollback plan ready
4. Test upgrade process on a non-critical node first
5. Keep track of node tokens and FQDNs in a secure location.

## Conclusion

While upgrading a Kubernetes cluster in a homelab environment presents unique challenges, following a systematic approach and being prepared for common issues makes the process manageable. The key is to maintain proper documentation, understand your environment's specifics, and have solutions ready for potential problems.
This upgrade experience has helped me establish a more robust process for future upgrades and highlighted the importance of proper DNS resolution and storage management in a Kubernetes environment.
