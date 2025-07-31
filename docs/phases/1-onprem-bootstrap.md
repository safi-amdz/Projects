
# Phase 1: On-Premise Infrastructure Setup

This phase focuses on bootstrapping the core infrastructure on-premises using a local Dell PowerEdge R720 server. The objective is to establish a foundational stack of services that simulate enterprise-grade internal tools, including file management, search, and basic MLOps workflows.

---

## Goals

| Objective                            | Description |
|-------------------------------------|-------------|
| Core Service Deployment             | Set up Nextcloud, Qdrant, FastAPI, MLflow within VMs |
| Virtualization & Networking         | Use Proxmox to create isolated VMs with static IPs    |
| Internal DNS and Shared Access      | Provide internal resolution and local file access     |

---

## Substeps

### 1. Proxmox Host Setup
- Install Proxmox VE on the PowerEdge R720
- Configure RAID (if needed)
- Set static IP and enable remote admin access

### 2. Create Internal VM Infrastructure
- Create VM templates (Ubuntu Server 22.04 LTS)
- Spin up:
  - `vm-nextcloud`
  - `vm-qdrant`
  - `vm-mlflow`
  - `vm-fastapi`

### 3. Internal DNS Setup
- Option 1: Use OPNsense DNS Resolver
- Option 2: Use Pi-hole as DNS + DHCP server

### 4. Service Setup
- Nextcloud with local volume storage
- Qdrant vector DB with local embedding test set
- MLflow for lightweight experiment tracking
- FastAPI for inference and embedding operations

### 5. Verification
- Local DNS resolution for all service VMs
- Services reachable internally via IP and hostname
- Basic smoke tests on each deployed tool

---

## Deliverables

| Deliverable                        | Description |
|-----------------------------------|-------------|
| Network topology diagram          | Diagram of VM layout and interconnects |
| All services operational          | Accessible locally via static IP or DNS |
| Proxmox VM export (optional)      | Snapshot or export of base VMs |

---

## Estimated Time

| Task Category        | Time Estimate |
|----------------------|---------------|
| Proxmox Setup        | 1 Day         |
| VM Creation & DNS    | 1 Day         |
| Core Services Install| 1–2 Days      |
| Testing & Diagrams   | 1 Day         |

Total Time: **~4–5 Days**
