# Instructor Guide: Terraform with VMware Workstation 17 Pro

Welcome! Let's build the `infrastructure/terraform` directory specifically tailored for your local **VMware Workstation 17 Pro** setup. 

First, a quick validation of your host environment:
* **Host RAM (16 GB):** Your host has 16 GB of memory. Allocating **2 GB** for the Backend VM and **2 GB** for the Database VM (totaling 4 GB) is safe. It leaves 12 GB for Windows, WSL, and your development tools.

---

## 1. Directory Structure

For VMware Workstation, we will structure our Terraform files as follows:

```text
infrastructure/
└── terraform/
    ├── providers.tf      # Defines the VMware Workstation provider & REST API connection
    ├── variables.tf      # Declares configuration variables (RAM, vCPUs, VM paths)
    ├── terraform.tfvars  # The actual values for variables (e.g., REST API credentials)
    ├── main.tf           # Declares the virtual machines, networks, and disks
    └── outputs.tf        # Outputs IP addresses and configurations for Ansible to use
```

---

## 2. File Explanations & Templates

Here is the exact purpose of each file and the boilerplate code for them.

### File 1: `providers.tf`
**Purpose:** Tells Terraform which plug-in to download to manage VMware Workstation. Since VMware Workstation doesn't have an official provider, we use a community-supported provider that talks to the VMware REST API.

```hcl
# infrastructure/terraform/providers.tf

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    vmwareworkstation = {
      source  = "elsantore/vmwareworkstation"
      version = "1.0.3"
    }
  }
}

provider "vmwareworkstation" {
  # These point to the vmrest.exe service running on your Windows host
  user     = var.vmrest_user
  password = var.vmrest_password
  url      = "http://127.0.0.1:8697/api"
}
```

---

### File 2: `variables.tf`
**Purpose:** Declares the inputs we want to make configurable. This prevents hardcoding passwords, file paths, or resource sizes inside the main logic.

```hcl
# infrastructure/terraform/variables.tf

variable "vmrest_user" {
  type        = string
  description = "Username for the VMware Workstation REST API (vmrest)"
}

variable "vmrest_password" {
  type        = string
  sensitive   = true
  description = "Password for the VMware Workstation REST API (vmrest)"
}

variable "source_template_path" {
  type        = string
  description = "Path to the base Ubuntu 24.04 VMX file on your host to clone from"
}

variable "vm_target_folder" {
  type        = string
  default     = "C:\\Users\\Hamza\\Documents\\Virtual Machines"
  description = "Folder where the cloned VMs will be stored"
}
```

---

### File 3: `terraform.tfvars`
**Purpose:** Stores the sensitive and host-specific values for your variables.
> [!IMPORTANT]
> This file contains credentials. It should be added to your `.gitignore` file so you do not commit it to your GitHub repository.

```hcl
# infrastructure/terraform/terraform.tfvars (DO NOT COMMIT THIS FILE)

vmrest_user          = "hamza_admin"
vmrest_password      = "securepassword123"
source_template_path = "C:\\Users\\Hamza\\Documents\\Virtual Machines\\Templates\\ubuntu2404-base.vmx"
```

---

### File 4: `main.tf`
**Purpose:** This is the heart of Terraform. It defines:
1. The **private LAN segment** (for communication between the backend and DB).
2. The **Backend VM** (configured with two networks: Host NAT + private LAN).
3. The **Database VM** (configured with one network: private LAN).

```hcl
# infrastructure/terraform/main.tf

# 1. Define the Private LAN Segment
resource "vmwareworkstation_lan_segment" "private_segment" {
  name = "gekko-private-lan"
}

# 2. Define the Backend VM
resource "vmwareworkstation_vm" "backend_server" {
  name             = "gekko-backend-vm"
  source_vmx_path  = var.source_template_path
  target_directory = "${var.vm_target_folder}\\gekko-backend-vm"
  
  # Resource Sizing
  cpus   = 2
  memory = 2048 # 2 GB RAM

  # NIC 1: Connected to the host's NAT network for Internet and SSH access
  network_adapter {
    type         = "nat"
    nic_name     = "ethernet0"
  }

  # NIC 2: Connected to the isolated LAN segment for Database access
  network_adapter {
    type         = "custom"
    nic_name     = "ethernet1"
    vnet_name    = vmwareworkstation_lan_segment.private_segment.name
  }
}

# 3. Define the Database VM
resource "vmwareworkstation_vm" "database_server" {
  name             = "gekko-database-vm"
  source_vmx_path  = var.source_template_path
  target_directory = "${var.vm_target_folder}\\gekko-database-vm"
  
  # Resource Sizing
  cpus   = 2
  memory = 2048 # 2 GB RAM

  # NIC 1: Only connected to the isolated LAN segment
  network_adapter {
    type         = "custom"
    nic_name     = "ethernet0"
    vnet_name    = vmwareworkstation_lan_segment.private_segment.name
  }
}
```

---

### File 5: `outputs.tf`
**Purpose:** Prints out critical data once Terraform finishes provisioning. Ansible will read these outputs to know what IP addresses to connect to.

```hcl
# infrastructure/terraform/outputs.tf

output "backend_vm_nat_ip" {
  value       = vmwareworkstation_vm.backend_server.ip
  description = "The NAT network IP of the backend server (accessible by host/Ansible)"
}
```

---

## 3. Instructor's Lesson: How the cloning works in VMware Workstation

In cloud environments (like AWS), Terraform calls an API that automatically grabs a public Ubuntu image and boots it. 

In **VMware Workstation**, it works differently:
1. You must manually (or via Packer) build a **base Ubuntu 24.04 VM** first.
2. Configure this base VM with your SSH key so you can access it, then shut it down.
3. You point `source_template_path` to that VM's `.vmx` file.
4. When you run `terraform apply`, the Workstation provider communicates with the local REST API to copy that base VM's files into new directories for your Backend and Database servers.
5. Once copied, it boots them up.
