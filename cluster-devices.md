# 📦 Kubernetes Cluster Setup using kubeadm (Ubuntu 24.04 LTS)

> ✅ **Your Environment:** Ubuntu 24.04 LTS (Noble Numbat)  
> ✅ **Kubernetes Version:** v1.30+ (latest stable)  
> ✅ **Container Runtime:** containerd (recommended for Kubernetes 1.24+)

This documentation guides you in setting up a Kubernetes cluster with **one master node** and **one worker node** on Ubuntu 24.04 LTS.

---

## 📋 Proxmox VM Environment (Your Setup)

Based on your Proxmox VE, you have the following VMs:

| VM ID | Hostname | IP | OS | RAM | CPU | Role |
|-------|----------|-----|-----|-----|-----|------|
| 100 | masterone | 192.168.0.100 | Ubuntu 24.04 | 2G | 2 | Master (Optional) |
| 101 | mastertwo | 192.168.0.101 | Ubuntu 24.04 | 2G | 2 | **Primary Master** |
| 102 | workeroone | 192.168.0.102 | Ubuntu 24.04 | 2G | 2 | **Worker** |
| 103 | workertwo | 192.168.0.103 | Ubuntu 24.04 | 2G | 2 | Worker (Optional) |
| 104 | loadblone | 192.168.0.104 | Ubuntu 24.04 | 2G | 2 | Load Balancer (Optional) |
| 105 | loadbltwo | 192.168.0.105 | Ubuntu 24.04 | 2G | 2 | Load Balancer (Optional) |

> 💡 **This guide focuses on:** `mastertwo (101)` and `workeroone (102)`

---

## Assumptions

| Role | FQDN | IP | OS | RAM | CPU |
|------|------|-----|-----|-----|-----|
| Master | mastertwo | 192.168.0.101 | Ubuntu 24.04 | 2G | 2 |
| Worker | workeroone | 192.168.0.102 | Ubuntu 24.04 | 2G | 2 |

---

# 🔧 Part 1: Common Setup (Both Master & Worker Nodes)

> ⚠️ **Run ALL commands in this section on BOTH mastertwo AND workeroone as root user**

## 1.1 Become Root User

```bash
sudo su -