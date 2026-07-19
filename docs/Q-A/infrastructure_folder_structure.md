# Project Folder Structure with Terraform & Ansible

Integrating Infrastructure as Code (IaC) directly into the project repository ensures that your infrastructure versioning matches your application codebase. Below is the proposed layout for the `infrastructure/` directory in your root folder.

---

## 1. Directory Tree Layout

```text
GEKKO-MicroERP-SaaS/
├── docs/                        # Project documentation
├── src/                         # NestJS Backend source code
├── prisma/                      # Database models and migrations
├── package.json
│
└── infrastructure/              # All infrastructure configuration lives here
    ├── terraform/               # Provisioning layer
    │   ├── modules/             # Reusable Terraform blocks (e.g. virtual machines, networks)
    │   │   ├── vm_instance/
    │   │   └── virtual_network/
    │   └── environments/        # Target platforms
    │       ├── dev-workstation/ # For your local VMware Workstation lab
    │       │   ├── main.tf
    │       │   ├── variables.tf
    │       │   ├── outputs.tf
    │       │   └── terraform.tfvars
    │       └── staging-proxmox/ # Future migration target (on-prem)
    │           ├── main.tf
    │           └── variables.tf
    │
    └── ansible/                 # Configuration layer
        ├── ansible.cfg          # Default Ansible settings (e.g. timeout, keys)
        ├── group_vars/          # Variables shared across VM groups
        │   ├── all.yml          # Global vars (e.g., Tailscale auth key, Docker versions)
        │   ├── backend.yml      # Backend-specific variables (e.g., Node version)
        │   └── database.yml     # DB-specific variables (e.g., Postgres configuration)
        ├── inventory/           # Define IP addresses for hosts
        │   ├── dev-workstation.ini
        │   └── staging-proxmox.ini
        ├── playbooks/           # Entry points for running Ansible tasks
        │   ├── site.yml         # Master playbook (runs both DB and App server tasks)
        │   ├── setup-backend.yml
        │   └── setup-database.yml
        └── roles/               # Modulable, reusable configuration code
            ├── common/          # Basic OS setup (APT updates, system tools)
            ├── nodejs/          # Installs Node.js & PM2
            ├── postgresql/      # Installs PostgreSQL & PostGIS
            ├── tailscale/       # Sets up Tailscale tunnel
            └── app_deploy/      # Handles copying dist/ code and restarting application
```

---

## 2. Key Directories & Design Choices

### A. Terraform: Organized by "Environment"
Instead of throwing all Terraform code into a single directory, splitting it by `environments/` makes scaling trivial.
* `dev-workstation/main.tf` will call the community Workstation provider.
* `staging-proxmox/main.tf` will call the Proxmox provider.
* When you run Terraform, you do it inside the specific environment folder:
  ```powershell
  cd infrastructure/terraform/environments/dev-workstation
  terraform apply
  ```

### B. Ansible: Organized by "Roles"
The `roles/` directory contains modular task directories.
* For instance, if you decide to replace NestJS with a different backend later, you only change the `nodejs` and `app_deploy` roles. The `postgresql` and `tailscale` roles remain untouched.
* You target different environments simply by swapping the inventory file when executing the playbook:
  ```bash
  # Targets local Workstation VMs
  ansible-playbook -i inventory/dev-workstation.ini playbooks/site.yml
  
  # Targets Proxmox VMs
  ansible-playbook -i inventory/staging-proxmox.ini playbooks/site.yml
  ```
