---
title: "Managing Proxmox Containers with Terraform"
date: 2025-03-26
author: "guyfromtheke"
description: "A step-by-step guide on how to create and manage LXC containers in Proxmox using Terraform"
tags: ["terraform", "proxmox", "infrastructure-as-code", "devops", "lxc"]
---

# Managing Proxmox Containers with Terraform

Infrastructure as Code (IaC) has revolutionized the way we manage and deploy infrastructure. In this blog post, I'll walk you through setting up and managing LXC containers in Proxmox using Terraform, a popular IaC tool. We'll also explore a common challenge when provisioning SSH access and how to work around it effectively.

## Prerequisites

Before we begin, make sure you have:

1. A Proxmox VE server up and running (I'm using version 8.x)
2. Terraform installed on your local machine (version 1.0+)
3. LXC templates downloaded on your Proxmox server
4. API token created in Proxmox with the appropriate permissions

## Project Structure

Let's set up a simple project structure for our Terraform configuration:

```
proxmox/
├── main.tf           # Main configuration file
├── variables.tf      # Variable declarations
├── terraform.tfvars  # Variable values
└── setup_container.sh # Post-deployment script
```

## Setting Up Terraform Configuration

Let's create each file one by one.

### 1. Variables Definition

First, let's define our variables in `variables.tf`:

```terraform
variable "pm_api_url" {
  description = "Proxmox API URL"
  type        = string
  default     = "https://your-proxmox-ip:8006/api2/json"
}

variable "pm_api_token_id" {
  description = "Proxmox API token ID"
  type        = string
  default     = "root@pam!your-token-name"
}

variable "pm_api_token_secret" {
  description = "Proxmox API token secret"
  type        = string
  # Do not set a default value for sensitive variables
  # Use environment variable TF_VAR_pm_api_token_secret instead
}

variable "pm_tls_insecure" {
  description = "Disable TLS verification (not recommended for production)"
  type        = bool
  default     = true
}

variable "target_node" {
  description = "Proxmox target node"
  type        = string
  default     = "your-node-name"
}

variable "container_hostname" {
  description = "Hostname for the LXC container"
  type        = string
  default     = "debian-lxc"
}

variable "container_template" {
  description = "OS template for the container"
  type        = string
  default     = "local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst"
}

variable "container_root_password" {
  description = "Root password for the container"
  type        = string
  sensitive   = true
  # Do not set a default value for sensitive variables
  # Use environment variable TF_VAR_container_root_password instead
}

variable "container_user" {
  description = "Username for the custom user"
  type        = string
  default     = "myuser" #or just create the user that you want,its your preference.
}

variable "container_user_password" {
  description = "Password for the custom user"
  type        = string
  sensitive   = true
  # Do not set a default value for sensitive variables
  # Use environment variable TF_VAR_container_user_password instead
}

variable "container_cores" {
  description = "Number of CPU cores for the container"
  type        = number
  default     = 1
}

variable "container_memory" {
  description = "Memory in MB for the container"
  type        = number
  default     = 512
}

variable "container_storage" {
  description = "Storage name for the container"
  type        = string
  default     = "local-lvm"
}

variable "container_disk_size" {
  description = "Disk size for the container"
  type        = string
  default     = "8G"
}

variable "container_ip" {
  description = "IP address for the container"
  type        = string
  default     = "x.x.x.x"
}
```

### 2. Variable Values

Next, let's set up our `terraform.tfvars` file with the values for our variables:

```terraform
# Configuration for Proxmox LXC container deployment
# IMPORTANT: Set the following environment variables before running terraform apply:
# - TF_VAR_pm_api_token_secret: Your Proxmox API token secret
# - TF_VAR_container_root_password: A strong password for the root user
# - TF_VAR_container_user_password: A strong password for the regular user

# Proxmox connection
pm_api_url          = "https://your-proxmox-ip:8006/api2/json"
pm_api_token_id     = "root@pam!your-token-name"
# The token secret will be read from environment variable TF_VAR_pm_api_token_secret
pm_tls_insecure     = true  # Set to true to skip certificate validation for self-signed certificates

# Container configuration
target_node            = "your-node-name"  # Verify this is your actual Proxmox node name
container_hostname     = "debian-lxc"
container_template     = "local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst"
# Root password will be read from environment variable TF_VAR_container_root_password
container_user          = "myuser"
# User password will be read from environment variable TF_VAR_container_user_password

# Resources
container_cores     = 1
container_memory    = 512
container_storage   = "local-lvm"
container_disk_size = "8G"
container_ip        = "x.x.x.x"  # The static IP we want to assign
```

### 3. Main Configuration

Now, let's create the `main.tf` file that defines our Proxmox LXC container:

```terraform
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "~> 2.9.14"
    }
  }
}

provider "proxmox" {
  pm_api_url          = var.pm_api_url
  pm_api_token_id     = var.pm_api_token_id
  pm_api_token_secret = var.pm_api_token_secret
  pm_tls_insecure     = var.pm_tls_insecure
}

resource "proxmox_lxc" "debian_container" {
  target_node  = var.target_node
  hostname     = var.container_hostname
  ostemplate   = var.container_template
  password     = var.container_root_password
  unprivileged = true
  start        = true

  # Resource settings
  cores    = var.container_cores
  memory   = var.container_memory
  rootfs {
    storage = var.container_storage
    size    = var.container_disk_size
  }

  # Network configuration with static IP
  network {
    name   = "eth0"
    bridge = "vmbr0"
    ip     = "${var.container_ip}/24"
    gw     = "x.x.x.x"  # Replace with your gateway
  }

  # Additional security and features
  features {
    nesting = true
  }

  # Proper startup configuration
  onboot      = true
  startup     = "order=1"

  # Description
  description = "Debian 12 container managed by Terraform"
}
```

## The SSH Provisioning Challenge

When I first attempted to deploy this configuration, I tried to include a `remote-exec` provisioner to set up SSH and create a user within the container. Here's what the provisioner looked like:

```terraform
provisioner "remote-exec" {
  inline = [
    "useradd -m -s /bin/bash ${var.container_user}",
    "echo '${var.container_user}:${var.container_user_password}' | chpasswd",
    "apt-get update && apt-get install -y sudo",
    "usermod -aG sudo ${var.container_user}",
    "apt-get install -y openssh-server",
    "systemctl enable ssh",
    "systemctl start ssh"
  ]
  
  connection {
    type     = "ssh"
    user     = "root"
    password = var.container_root_password
    host     = var.container_ip
  }
}
```

However, I ran into a classic chicken-and-egg problem: The `remote-exec` provisioner requires SSH to already be available on the container, but we're trying to use it to install SSH in the first place!

## The Two-Step Solution

The solution is to break the process into two steps:

1. First, deploy the container with just the basic configuration (without the SSH provisioner)
2. Then use a separate script that runs on the Proxmox host to set up SSH and create users

### The Post-Deployment Script

Let's create a script called `setup_container.sh` that will run on the Proxmox host:

```bash
#!/bin/bash
# Script to set up SSH and user in LXC container
# Usage: setup_container.sh <container_id> <username> <password>

CONTAINER_ID=$1
USERNAME=$2
PASSWORD=$3

if [ -z "$CONTAINER_ID" ] || [ -z "$USERNAME" ] || [ -z "$PASSWORD" ]; then
  echo "Usage: $0 <container_id> <username> <password>"
  exit 1
fi

echo "Setting up container $CONTAINER_ID..."

# Run commands inside the container
pct exec $CONTAINER_ID -- apt-get update
pct exec $CONTAINER_ID -- apt-get install -y sudo openssh-server
pct exec $CONTAINER_ID -- systemctl enable ssh
pct exec $CONTAINER_ID -- systemctl start ssh
pct exec $CONTAINER_ID -- useradd -m -s /bin/bash $USERNAME
pct exec $CONTAINER_ID -- bash -c "echo '$USERNAME:$PASSWORD' | chpasswd"
pct exec $CONTAINER_ID -- usermod -aG sudo $USERNAME

echo "Container setup complete. You can now SSH to the container as $USERNAME@<container_ip>"
```

This script uses the Proxmox Container Tools (`pct`) command to execute commands directly inside the container without requiring SSH access.

## Deployment Steps

Now, let's put it all together and deploy the container:

1. **Initialize Terraform:**
   ```bash
   terraform init
   ```

2. **Set environment variables for sensitive values:**
   ```bash
   export TF_VAR_pm_api_token_secret="your-api-token-secret"
   export TF_VAR_container_root_password="your-root-password"
   export TF_VAR_container_user_password="your-user-password"
   ```

3. **Apply the Terraform configuration:**
   ```bash
   terraform apply
   ```

4. **Note the container ID** - this is typically shown in the Proxmox UI or can be retrieved with:
   ```bash
   # On the Proxmox host
   pct list
   ```

5. **Upload the setup script to the Proxmox host:**
   ```bash
   scp setup_container.sh username@proxmox-server-ip:/tmp/
   ```

6. **SSH to the Proxmox host and run the setup script:**
   ```bash
   ssh username@proxmox-server-ip
   cd /tmp
   chmod +x setup_container.sh
   ./setup_container.sh 129 myuser your-user-password  # Replace 129 with your container ID
   ```

7. **Verify SSH access:**
   ```bash
   ssh myuser@ip.of.my.container  # Replace with your container's IP
   ```

## Conclusion

Using Terraform to manage Proxmox LXC containers provides a repeatable, version-controlled approach to infrastructure management. While I encountered a challenge with SSH provisioning, the two-step approach offers a practical solution.

In future iterations of this setup, you might want to:

1. Consider using cloud-init-enabled templates that include SSH by default (this will be in a future blogpost )
2. Use a more secure method for handling secrets such as hashicorp vault, git crypt, etc


Remember to always handle your API tokens and credentials securely, and avoid storing them in version control systems or exposing them in your configuration files.

Happy Proxmoxing & containerizing! :)

