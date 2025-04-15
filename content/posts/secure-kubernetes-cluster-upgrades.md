---
title: "Secure Kubernetes Cluster Upgrades: A Step-by-Step Guide"
date: 2025-04-15T04:26:00+03:00
draft: false
tags: ["kubernetes", "devops", "security", "automation", "k3s"]
categories: ["Kubernetes", "DevOps"]
---

# Secure Kubernetes Cluster Upgrades: A Step-by-Step Guide

In this post, I'll document a recent Kubernetes cluster upgrade process I implemented, focusing on security, automation, and best practices. I'll walk through the entire process from environment assessment to verification, highlighting challenges and solutions along the way.

## Environment Overview

Our setup consisted of a small K3s Kubernetes cluster running on Ubuntu 24.04 with:

- 1 master node (control plane)
- 2 worker nodes
- All nodes running an older kernel version (6.8.0-56-generic)
- Multiple security updates pending

## Upgrade Objectives

1. Update all system packages across all nodes
2. Apply kernel updates securely
3. Minimize downtime by implementing a rolling upgrade
4. Establish secure automation for future upgrades

## Step 1: Setting Up Secure Access

The first step was to establish secure, password-less authentication using SSH keys instead of using plaintext passwords.

```bash
# Generate a secure ED25519 SSH key
ssh-keygen -t ed25519 -f ~/.ssh/k8s_worker_access -C "k8s_worker_automation_$(date +%Y%m%d)" -N ""

# Set up the automation directory structure
mkdir -p ~/k8s-automation/inventory ~/k8s-automation/scripts

# Create an inventory of worker nodes
cat << EOF > ~/k8s-automation/inventory/workers.txt
10.10.10.xx
10.10.10.yy
EOF
```

## Step 2: Setting Up the Upgrade Script

I created an upgrade script that would:

1. Safely drain each node before updates
2. Apply system updates
3. Handle required reboots
4. Verify the node is ready before proceeding to the next one

```bash
#!/bin/bash

SSH_KEY="/root/.ssh/k8s_worker_access"
declare -A NODE_MAP
NODE_MAP["10.10.10.xx"]="uk8s-worker"
NODE_MAP["10.10.10.yy"]="uk8s-worker2"

while IFS= read -r worker_ip; do
  node_name="${NODE_MAP[$worker_ip]}"
  echo "Processing worker: $worker_ip (Kubernetes node: $node_name)"
  
  # Drain the node
  kubectl drain "$node_name" --ignore-daemonsets --delete-emptydir-data --force

  # Perform upgrade
  ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no "root@${worker_ip}" '
      export DEBIAN_FRONTEND=noninteractive
      apt-get update
      apt-get upgrade -y
      apt-get autoremove -y
      apt-get clean
  '

  # Reboot if necessary
  ssh -i "$SSH_KEY" "root@${worker_ip}" '
      if [ -f /var/run/reboot-required ]; then
          echo "Rebooting node..."
          nohup bash -c "sleep 2; reboot" &
          exit
      fi
  '

  # Wait for node to come back
  echo "Waiting for node to be ready..."
  sleep 30
  kubectl wait --for=condition=ready "node/${node_name}" --timeout=300s

  # Uncordon the node
  kubectl uncordon "$node_name"
  
  echo "Node $worker_ip ($node_name) has been upgraded and is ready"
done < "/root/k8s-automation/inventory/workers.txt"
```

## Step 3: Distributing SSH Keys Securely

With the automation script prepared, I securely distributed the SSH keys to all nodes:

```bash
# Install sshpass for initial key distribution
apt-get install -y sshpass

# Distribute keys to each worker node
for worker in $(cat ~/k8s-automation/inventory/workers.txt); do
    echo "Distributing SSH key to $worker"
    sshpass -p "[REDACTED]" ssh-copy-id -i ~/.ssh/k8s_worker_access.pub \
    -o StrictHostKeyChecking=no "root@${worker}"
done
```

## Step 4: Upgrading Worker Nodes

With everything prepared, I proceeded with the worker node upgrades. The process was executed in a sequential manner to ensure cluster stability:

```bash
# Execute the upgrade script
~/k8s-automation/scripts/upgrade-workers.sh
```

During the upgrade process:

1. Each worker node was cordoned and drained
2. All system packages were updated (approximately 36 packages including security updates)
3. New kernel (6.8.0-57-generic) was installed
4. Nodes were rebooted where required
5. The script waited for the node to rejoin the cluster
6. The node was uncordoned before moving to the next node

This approach ensured that:
- The cluster maintained availability throughout the upgrade
- Workloads were properly rescheduled
- No manual intervention was required once the process started

## Step 5: Upgrading the Master Node

The master node required special attention as it hosts the control plane components:

1. I cordoned the master node to prevent new workloads but did not drain it
2. Checked for pending updates
3. Applied the updates
4. Monitored the reboot process

```bash
# Cordon the master node
kubectl cordon uk8s-master

# Check for updates
ssh -i ~/.ssh/k8s_worker_access root@10.10.10.zz 'apt list --upgradable'

# In this case, the master node had fewer pending updates (only 4 packages)
# compared to the worker nodes (36 packages) as it had been patched more recently

# Apply updates
ssh -i ~/.ssh/k8s_worker_access root@10.10.10.zz '
    export DEBIAN_FRONTEND=noninteractive
    apt-get update && \
    apt-get upgrade -y && \
    apt-get autoremove -y && \
    apt-get clean && \
    if [ -f /var/run/reboot-required ]; then
        echo "Reboot required - proceeding with reboot"
        nohup bash -c "sleep 2; reboot" &
        exit
    fi
'
```

## Challenges Faced & Solutions

### Challenge 1: Kubernetes API Unavailability During Master Reboot

When the master node rebooted, the Kubernetes API became temporarily unavailable. This was expected, but it required proper handling.

**Solution:** I added logic to wait for the API to become available again and implemented a retry mechanism.

### Challenge 2: Kernel Module Loading After Reboot

After rebooting the master node with the new kernel, there were issues with IP tables modules not being loaded correctly.

**Solution:** I explicitly loaded the required kernel modules and restarted the K3s service:

```bash
ssh root@10.10.10.zz '
    modprobe ip_tables && 
    modprobe iptable_nat && 
    modprobe iptable_filter && 
    modprobe xt_mark && 
    systemctl restart k3s
'
```

### Challenge 3: Kubectl Configuration

The kubectl configuration needed to be updated to connect to the master node after reboot.

**Solution:** I retrieved the updated configuration from the master node and adjusted it:

```bash
ssh root@10.10.10.zz 'cat /etc/rancher/k3s/k3s.yaml' > ~/.kube/config.new
sed -i "s/127.0.0.1/10.10.10.zz/" ~/.kube/config.new
export KUBECONFIG=~/.kube/config.new

# Make this configuration persistent
cp ~/.kube/config.new ~/.kube/config
```

## Verification

After completing all upgrades, I verified the status of the cluster:

```bash
$ kubectl get nodes -o wide
NAME           STATUS   ROLES                  AGE   VERSION        INTERNAL-IP     OS-IMAGE             KERNEL-VERSION     
uk8s-master    Ready    control-plane,master   44d   v1.31.6+k3s1   10.10.10.zz     Ubuntu 24.04.2 LTS   6.8.0-57-generic   
uk8s-worker    Ready    worker                 44d   v1.31.6+k3s1   10.10.10.xx     Ubuntu 24.04.2 LTS   6.8.0-57-generic   
uk8s-worker2   Ready    worker                 44d   v1.31.6+k3s1   10.10.10.yy     Ubuntu 24.04.2 LTS   6.8.0-57-generic   
```

All nodes were successfully upgraded with:
- Latest system packages
- New kernel version (6.8.0-57-generic)
- All nodes in Ready state
- Cluster fully operational

## Best Practices & Lessons Learned

1. **Security First**: Never use plaintext passwords; always use SSH keys for automation
2. **Backup Before Upgrading**: Always ensure you have proper backups before starting
3. **Rolling Updates**: Upgrade one node at a time to maintain cluster availability
4. **Proper Draining**: Always drain nodes properly before maintenance
5. **Master Node Care**: Take extra precautions with the control plane node
6. **Recovery Plan**: Have a plan for API unavailability during master node upgrades
7. **Kernel Module Awareness**: Be prepared to load specific kernel modules after a kernel upgrade. Consider adding them to `/etc/modules` to load automatically on boot
8. **Validation**: Always verify the cluster is healthy after each step

## Conclusion

This upgrade process demonstrates a secure, automated approach to keeping Kubernetes clusters up-to-date with minimal downtime. By implementing proper SSH key authentication, creating reusable automation scripts, and following a methodical approach, future upgrades can be handled efficiently and securely.

The resulting cluster is now running the latest kernel and security patches, improving both performance and security. This process can be adapted for clusters of any size, making it a valuable addition to your operations playbook.

