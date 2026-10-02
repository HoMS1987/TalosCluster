<div align="center">

# 🧊 Personal Talos Kubernetes Cluster

*A self-hosted, GitOps-driven single-node Kubernetes cluster built on Talos Linux.*

[![Talos](https://img.shields.io/endpoint?url=https://kromgo.homsserver.com/badges/talos_version?format=shields&style=for-the-badge&logo=talos&logoColor=white&label=%20)](https://www.talos.dev/)&nbsp;&nbsp;
[![Kubernetes](https://img.shields.io/endpoint?url=https://kromgo.homsserver.com/badges/kubernetes_version?format=shields&style=for-the-badge&logo=kubernetes&logoColor=white&label=%20)](https://www.kubernetes.io/)

[![Age-Days](https://img.shields.io/endpoint?url=https://kromgo.homsserver.com/badges/cluster_age_days?format=shields&style=flat-square&label=Age)](https://github.com/home-operations/kromgo)&nbsp;&nbsp;
[![Uptime-Days](https://img.shields.io/endpoint?url=https://kromgo.homsserver.com/badges/cluster_uptime_days?format=shields&style=flat-square&label=Uptime)](https://github.com/home-operations/kromgo)&nbsp;&nbsp;
[![Node-Count](https://img.shields.io/endpoint?url=https://kromgo.homsserver.com/badges/cluster_node_count?format=shields&style=flat-square&label=Nodes)](https://github.com/home-operations/kromgo)&nbsp;&nbsp;
[![Pod-Count](https://img.shields.io/endpoint?url=https://kromgo.homsserver.com/badges/cluster_pod_count?format=shields&style=flat-square&label=Pods)](https://github.com/home-operations/kromgo)&nbsp;&nbsp;
[![CPU-Usage](https://img.shields.io/endpoint?url=https://kromgo.homsserver.com/badges/cluster_cpu_usage?format=shields&style=flat-square&label=CPU)](https://github.com/home-operations/kromgo)&nbsp;&nbsp;
[![Memory-Usage](https://img.shields.io/endpoint?url=https://kromgo.homsserver.com/badges/cluster_memory_usage?format=shields&style=flat-square&label=Memory)](https://github.com/home-operations/kromgo)
<!-- Needs Alertmanager, which is currently disabled
[![Alerts](https://img.shields.io/endpoint?url=https://kromgo.homsserver.com/badges/cluster_alert_count?format=shields&style=flat-square&label=Alerts)](https://github.com/home-operations/kromgo)
-->

</div>

---

## 📌 Overview

This repository contains the full **GitOps-managed configuration** for my personal Kubernetes cluster.
The cluster runs on **Talos Linux** and is fully declarative: every component, application, and configuration is defined in Git and continuously reconciled using **FluxCD**.

Key goals of this setup:

* 🔁 **Reproducibility** – rebuild the entire cluster from Git
* 🔒 **Immutability & Security** – minimal OS, no SSH, API-driven management
* 📈 **Observability** – metrics and public cluster statistics
* 🤖 **Automation-first** – dependency updates and OS upgrades without manual intervention

---

## 🧠 Design Decisions

### Why Talos Linux?
- Immutable, minimal OS reduces attack surface
- No SSH or package manager
- Fully API-driven, ideal for GitOps-based Kubernetes clusters

### Why FluxCD?
- Continuous reconciliation instead of one-shot deployments
- Native Kubernetes integration
- Works seamlessly with SOPS for encrypted secrets

### Why a Single-Node Cluster?
- Simplifies operations and reduces complexity
- Ideal for homelab and learning environments
- Focuses on reproducibility rather than high availability

---

## 📡 Networking Assumptions

This cluster assumes a **simple and reliable home network environment**.

- The Talos VM relies on the TrueNAS host and a Fritzbox router for network connectivity
- No advanced routing, BGP, or multi-homing is assumed
- Networking is optimized for simplicity and stability rather than redundancy
- External access is handled via the external ingress controller and VPN tunnels

---

## 🧩 Core Components

| Component                 | Description                                                                       |
| ------------------------- | --------------------------------------------------------------------------------- |
| **Kubernetes**            | Container orchestration platform for running and managing workloads               |
| **Talos Linux**           | Immutable, API-driven Linux distribution purpose-built for Kubernetes             |
| **FluxCD**                | GitOps operator used for continuous reconciliation of cluster state               |
| **Cilium**                | CNI providing pod networking                                                      |
| **MetalLB**               | Load balancer for service IPs on the home network                                 |
| **ingress-nginx**         | Two ingress controllers: `internal` (LAN only) and `external` (public)            |
| **cert-manager**          | Automated Let's Encrypt certificates                                              |
| **Blocky**                | DNS server and ad blocker for the home network                                    |
| **Longhorn / OpenEBS**    | Block storage and local hostpath storage for stateful apps                        |
| **CloudNativePG**         | PostgreSQL operator for apps that need a database                                 |
| **VolSync**               | Backup and restore of persistent volumes to S3-compatible object storage          |
| **kube-prometheus-stack** | Metrics collection and alerting                                                   |
| **tuppr**                 | Automated Talos and Kubernetes upgrades driven by Git                             |
| **Mend Renovate**         | Automatically tracks and updates charts, container images and dependencies        |
| **SOPS**                  | Encryption of all secrets and credentials stored in Git, integrated with FluxCD   |
| **ClusterTool**           | Bootstrap tool from TrueForge used to build the basic cluster structure and setup |

---

## 🗂 Directory Structure

~~~text
clusters/
└── main/
    ├── kubernetes/
    │   ├── apps/           # User-facing applications
    │   ├── core/           # Cluster-wide services (Blocky, MetalLB config, ClusterIssuers)
    │   ├── flux-system/    # Flux itself and cluster-wide settings
    │   ├── kube-system/    # System components (Cilium, metrics-server, ...)
    │   ├── networking/     # Ingress controllers
    │   ├── observability/  # Prometheus stack, Headlamp, Kromgo
    │   └── system/         # Operators (cert-manager, CloudNativePG, Longhorn, VolSync, tuppr, ...)
    └── talos/              # Talos Linux machine and cluster configuration

repositories/
├── git/                    # Flux GitRepository sources
├── helm/                   # Flux HelmRepository sources
└── oci/                    # Flux OCIRepository sources
~~~

---

## 🔐 Secrets Management

All secrets and credentials are stored in this repository **encrypted with SOPS**.

- Secrets are committed to Git in encrypted form
- Decryption happens inside the cluster via FluxCD
- Decryption keys are managed externally and are never stored in Git
- This enables full GitOps workflows without exposing sensitive data

---

## ☁️ Cloud & External Dependencies

| Service        | Usage                                                        |
| -------------- | ------------------------------------------------------------ |
| **Cloudflare** | DNS management and S3-compatible object storage for backups  |
| **GitHub**     | Source control, CI, and GitOps reconciliation source         |

---

## 🖥 Hardware

### TrueNAS Host System

| Component             | Specification                                          |
| --------------------- | ------------------------------------------------------ |
| **Motherboard**       | ASRock Rack B650D4U                                    |
| **CPU**               | AMD Ryzen 9 7900 (12-Core)                             |
| **RAM**               | 64 GB (2× 32 GB) ECC DDR5                              |
| **GPU**               | NVIDIA Quadro P2000                                    |
| **Remote Management** | IPMI (ASRock Rack)                                     |

### Storage Configuration

| Pool          | Layout                | Drives                                                 |
| ------------- | --------------------- | ------------------------------------------------------ |
| **Boot Pool** | 1× Mirror, 2 wide     | 2× Samsung PM871a 512 GB SATA SSD                      |
| **Data Pool** | 1× RAIDZ2, 6 wide     | 5× Seagate Exos X18 16 TB + 1× Seagate Exos X20 16 TB  |
| **VM Pool**   | 1× Mirror, 2 wide     | 2× Samsung 990 PRO 1 TB NVMe                           |

### Talos Kubernetes Node (Virtual Machine)

The Kubernetes cluster runs as a **single-node Talos Linux virtual machine hosted on the TrueNAS system**.

| Component      | Specification                                  |
| -------------- | ---------------------------------------------- |
| **Node Count** | 1 (single-node cluster)                        |
| **Deployment** | Virtual machine on TrueNAS                     |
| **vCPU**       | 8 cores / 16 threads                           |
| **Memory**     | 32 GB                                          |
| **Storage**    | Virtual disk on the mirrored NVMe VM pool      |
| **GPU**        | NVIDIA Quadro P2000 (passthrough, time-sliced) |

---

## 📊 Monitoring & Status

* 📈 **Metrics** via Prometheus
* 🧮 **Cluster statistics** exposed via Kromgo and Shields.io

---

## 🙏 Acknowledgements

This cluster is heavily inspired by and built upon the excellent work of:

* **TrueForge** – https://trueforge.org/
* **Home Operations** – https://github.com/home-operations

Their open-source contributions and documentation made this setup possible.

---

> ⚠️ **Note**
> This repository is public for transparency and learning purposes.
> Secrets and credentials **are stored in Git in encrypted form** using **SOPS**.
> Decryption keys are managed externally and are **not** committed to the repository.
