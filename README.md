# Dagu Ansible Deployment - Distributed Execution

Professional Ansible automation for deploying Dagu workflow scheduler with distributed coordinator/worker architecture using gRPC over Tailscale mesh networking.

## Overview

Deploy Dagu with:
- ✅ **Distributed Architecture** - Coordinator with gRPC + multiple workers
- ✅ **Dynamic Inventory** - Auto-discover hosts from Tailscale
- ✅ **Flexible Workers** - Route tasks by labels (GPU, CPU, region)
- ✅ **NFS DAG Sharing** - Coordinator shares DAG files to workers
- ✅ **Secure Networking** - Tailscale mesh (no public exposure)
- ✅ **Secrets Management** - Bitwarden integration

## Architecture

```
Coordinator (workstation) - Tailscale: 100.64.0.0/10
┌──────────────────────────────┐
│ Web UI :8080                 │
│ gRPC Coordinator :50051      │
│ NFS Server (DAGs)            │
└──────────┬───────────────────┘
           │ gRPC over Tailscale
    ┌──────┴──────┬─────────────┐
┌───▼────┐   ┌───▼────┐   ┌───▼────┐
│Worker 1│   │Worker 2│   │Worker 3│
│Headless│   │Headless│   │Headless│
│general │   │gpu     │   │cpu     │
└────────┘   └────────┘   └────────┘
```

## Quick Start

### 1. Prerequisites

```bash
# Install Ansible
sudo apt update && sudo apt install ansible python3-pip

# Install Python dependencies
pip3 install bitwarden-sdk
```

**Required:**
- Tailscale account with admin access
- Bitwarden Secrets Manager account

### 2. Clone and Setup

```bash
git clone <your-repo>/ansible-playbooks.git
cd ansible-playbooks

# Install collections from Galaxy
ansible-galaxy collection install -r requirements.yml

# Configure environment
mkdir -p ~/.bitwarden
echo "YOUR-ACCESS-TOKEN" > ~/.bitwarden/access-token
chmod 600 ~/.bitwarden/access-token

# Add to ~/.bashrc or ~/.zshrc
export TAILSCALE_API_KEY="tskey-api-..."
export BWS_ACCESS_TOKEN="$(cat ~/.bitwarden/access-token)"

# Note: If also using Terraform, add this for Terraform compatibility:
# export TF_VAR_bws_access_token="$BWS_ACCESS_TOKEN"
```

### 3. Tag Hosts in Tailscale

In Tailscale Admin Console:

```
Coordinator: tag:dagu-coordinator
Workers:     tag:dagu-worker
```

### 4. Verify and Deploy

```bash
# Test inventory
ansible-inventory --graph
ansible all -m ping

# Deploy coordinator
ansible-playbook dagu-coordinator.yml

# Deploy workers
ansible-playbook dagu-workers.yml
```

Access Dagu UI: `http://<coordinator-hostname>:8080`

## Common Tasks

### Add New Worker

```bash
# 1. Create VM and install Tailscale
tailscale up --advertise-tags=tag:dagu-worker --authkey=<key>

# 2. Verify in inventory
ansible-inventory --graph | grep new-worker

# 3. Deploy
ansible-playbook dagu-workers.yml --limit new-worker

# 4. Deploy with custom labels (GPU example)
ansible-playbook dagu-workers.yml --limit gpu-worker \
  -e "worker_type=gpu" \
  -e "extra_worker_labels={'gpu':'nvidia-a100','vram':'80GB'}"
```

### Update Dagu Version

```bash
# Edit version
nano group_vars/all/all.yml
# Change: dagu_version: "1.25.0"

# Deploy
ansible-playbook site.yml --tags dagu
```

### Restart Services

```bash
# Restart coordinator
ansible dagu_coordinator -m systemd -a "name=dagu state=restarted" -b

# Restart workers
ansible dagu_worker -m systemd -a "name=dagu state=restarted" -b
```

### Update Collections

```bash
# Update from Ansible Galaxy
ansible-galaxy collection install -r requirements.yml --force
```

## Configuration

### Key Files

```
group_vars/
├── all/
│   ├── all.yml                  # Global config (versions, paths)
│   └── vault.yml                # Secrets (Bitwarden)
├── dagu_coordinator.yml         # Coordinator config
└── dagu_workers.yml             # Worker config

inventory/
└── tailscale.yaml               # Dynamic inventory

dagu-coordinator.yml             # Coordinator playbook
dagu-workers.yml                 # Worker playbook
requirements.yml                 # Collections & dependencies
```

### Worker Labels

Labels route tasks to appropriate workers:

```yaml
# General worker (default)
dagu_worker_labels:
  region: "eu-north-1"
  type: "general"

# GPU worker
dagu_worker_labels:
  type: "gpu"
  gpu: "nvidia-a100"
  vram: "80GB"

# CPU-intensive worker
dagu_worker_labels:
  type: "cpu-intensive"
  cores: "32"
  ram: "128GB"
```

Use in DAG files:

```yaml
# dag/train-model.yaml
steps:
  - name: train
    command: python train.py
    requiredLabels:
      gpu: "nvidia-a100"  # Only runs on GPU workers
```

## Troubleshooting

### Dynamic Inventory Not Working

```bash
# Check API key
echo $TAILSCALE_API_KEY

# Test API
curl -H "Authorization: Bearer $TAILSCALE_API_KEY" \
  "https://api.tailscale.com/api/v2/tailnet/-/devices"

# Refresh inventory
ansible-inventory --refresh --list
```

### Worker Not Connecting

```bash
# Check gRPC on coordinator
ssh coordinator 'sudo netstat -tlnp | grep 50051'

# Test from worker
ssh worker 'nc -zv coordinator 50051'

# Check worker logs
ssh worker 'sudo journalctl -u dagu -f'
```

### NFS Mount Issues

```bash
# Check exports
ssh coordinator 'sudo exportfs -v'

# Test from worker
ssh worker 'showmount -e coordinator'

# Check mounts
ssh worker 'df -h | grep nfs'
```

## Collections

This project uses collections published to Ansible Galaxy:

### Webmarcus Collections

- **webmarcus.nfs** - NFS server/client configuration
  - Repository: https://github.com/webmarcus/ansible-collection-nfs
  - Galaxy: https://galaxy.ansible.com/ui/repo/published/webmarcus/nfs/

- **webmarcus.dagu** - Dagu workflow scheduler
  - Repository: https://github.com/webmarcus/ansible-collection-dagu
  - Galaxy: https://galaxy.ansible.com/ui/repo/published/webmarcus/dagu/

### External Collections

- `freeformz.ansible` - Tailscale dynamic inventory
- `bitwarden.secrets` - Secrets Manager integration
- `geerlingguy.docker` - Docker installation
- `community.general` - General utilities

All collections are defined in `requirements.yml` and installed with:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Updating Collections

When new versions are published:

```bash
# Update requirements.yml version
nano requirements.yml

# Reinstall
ansible-galaxy collection install -r requirements.yml --force
```

Collections are automatically published to Galaxy via GitHub Actions when tagged.

## Documentation

- **Dagu Docs**: https://dagu.readthedocs.io/
- **Ansible Docs**: https://docs.ansible.com/
- **Tailscale Docs**: https://tailscale.com/kb/
- **Bitwarden Secrets**: https://bitwarden.com/products/secrets-manager/

## License

MIT

---

**Version**: 1.0.0
**Last Updated**: 2025-11-08
**Architecture**: Distributed Coordinator/Worker with gRPC + NFS over Tailscale
