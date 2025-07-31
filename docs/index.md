
# AI-Driven Hybrid Cloud Infrastructure · Documentation Hub


## Overview

This documentation hub presents a structured, end-to-end blueprint for how a mid-sized organization can build and evolve a secure, AI-powered document management and retrieval system. The project begins with on-premises infrastructure and will gradually move some of the services to the cloud, and it will result in a hybrid architecture that balances cost, security, and scalability.

The goal is to simulate a real-world IT transformation journey while showcasing modern practices in networking, automation, observability, and MLOps.

## Summary at a Glance

| Category           | Details                                                                 |
|--------------------|-------------------------------------------------------------------------|
| Organization Type  | Mid-sized U.S. Government contractor                                    |
| Objective          | Build a hybrid AI-powered document search platform                      |
| Starting Point     | On-premises infrastructure on a Dell R720 server                        |
| Target Outcome     | Gradual migration of services to AWS Free Tier while maintaining hybrid |
| Key Technologies   | Proxmox, Nextcloud, FastAPI, LangChain, Qdrant, MLflow, Prefect, Terraform, Grafana |
| Compliance Focus   | DFARS, NIST 800-171                                                     |
| Documentation Goal | Showcase real-world enterprise skills for career portfolio              |


## Business Background

| Organization | ABC |
|--------------|-----------------------------|
| Type         | Mid-sized U.S. Government contractor |
| Focus        | Cybersecurity, communications, logistics |
| Compliance   | DFARS, NIST 800-171          |

**Challenges:**
- Employees spend too much time searching for information buried in outdated shared drives.
- The infrastructure is fragmented, with siloed systems and no centralized intelligence.
- Due to compliance and budget constraints, not everything can move to the cloud.

**Solution:**
An internal AI-powered knowledge platform that makes documents easy to find, understand, and reuse—without violating security requirements.

## Project Objectives

| Goal | Description |
|------|-------------|
| Document Search Modernization | Use a local AI system to make files searchable |
| Selective Cloud Migration | Migrate cloud-suitable services using AWS Free Tier |
| Monitoring and Automation | Improve observability, logging, and task automation |
| Hybrid Operations | Maintain both on-prem and cloud infrastructure with security and access control |

## Project Structure

This project is divided into seven major phases. Each phase includes focused tasks that build upon previous work.

### Phase 1: On-Premise Infrastructure Setup

| Task | Description |
|------|-------------|
| Core Services | Deploy Nextcloud, Qdrant, MLflow, FastAPI |
| Virtualization | Use Proxmox to host VMs |
| Internal DNS | Set up local name resolution and shared storage |

### Phase 2: Network Design and Security

| Task | Description |
|------|-------------|
| Segmentation | Isolate services using OPNsense |
| VLANs | Configure VLANs and static IPs |
| Firewalling | Enforce access policies and traffic control |

### Phase 3: Cloud Foundations

| Task | Description |
|------|-------------|
| AWS Resources | Set up VPC, IAM roles, and S3 buckets |
| Infrastructure as Code | Use Terraform for provisioning |
| EC2 Setup | Prepare virtual machines for later deployment |

### Phase 4: Cloud Migrations

| Task | Description |
|------|-------------|
| Storage Migration | Back up documents from Nextcloud to S3 |
| API Offloading | Deploy FastAPI layer to EC2 |
| Monitoring | Set up basic CloudWatch integration |

### Phase 5: AI and Automation

| Task | Description |
|------|-------------|
| Document Pipeline | Set up RAG architecture using LangChain and FastAPI |
| Job Orchestration | Use Prefect to automate ETL and retraining |
| Result Storage | Log outputs in Qdrant and MLflow |

### Phase 6: Monitoring and Observability

| Task | Description |
|------|-------------|
| Logging | Centralize data from hybrid systems |
| Visualization | Use Grafana dashboards |
| Alerting | Set up failure detection and performance alerts |

### Phase 7: Reliability and Failure Simulation

| Task | Description |
|------|-------------|
| Recovery Tests | Simulate failures and validate failover plans |
| Access Expiration | Test IAM key expiry and alert systems |
| Restore Scenarios | Demonstrate backup and disaster recovery capabilities |

## What Makes This Project Unique

| Aspect | Explanation |
|--------|-------------|
| Real-World Scope | Mirrors actual migration and hybrid operations in mid-size companies |
| Security Awareness | Balances compliance and on-prem control with cloud advantages |
| Technical Depth | Combines Proxmox, OPNsense, AWS, MLflow, Terraform, and Grafana |
| Full Lifecycle Coverage | Goes from initial deployment to production hardening and chaos engineering |

All design decisions, tradeoffs, and configurations are documented in the these sections.
