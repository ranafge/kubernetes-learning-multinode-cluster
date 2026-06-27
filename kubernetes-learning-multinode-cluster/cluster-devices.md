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
| 101 | mastertwo | 192.168.0.101 | Ubuntu 24.04 | 2G | 2 | Primary Master |
| 102 | workeroone | 192.168.0.102 | Ubuntu 24.04 | 2G | 2 | **Worker** |
| 106 | workspace(master) | 192.168.0.106 | ubuntu 24.4 | 2G | 2 |  **Primary Master(Optional)**|
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
```
# 🚀 Complete HAProxy & Keepalived Load Balancer Setup Guide

> **File Name:** `haproxy-keepalived-loadbalancer.md`  
> **Version:** 1.0  
> **Author:** Linux Load Balancer Engineer  
> **Target OS:** Ubuntu 24.04 LTS / Debian 12  
> **Last Updated:** June 2026

## Architecture Overview
### Component Details

| Component | Node 1 (MASTER) | Node 2 (BACKUP) |
|-----------|-----------------|-----------------|
| **Hostname** | lb1.example.com | lb2.example.com |
| **IP Address** | 192.168.0.107 | 192.168.0.108 |
| **Role** | Primary Load Balancer | Secondary Load Balancer |
| **Priority** | 100 | 90 |
| **VIP** | `192.168.0.99` (Floating) | `192.168.0.99` (Standby) |
| **Services** | HAProxy + Keepalived | HAProxy + Keepalived |

### ASCII Architecture Diagram

```text
                              🌐 CLIENT REQUESTS
                                     │
                                     ▼
                    ┌─────────────────────────────────┐
                    │     VIP: 192.168.0.99           │
                    │     (Virtual IP - Floating)     │
                    └─────────────────────────────────┘
                                     │
            ┌────────────────────────┼────────────────────────┐
            │                        │                        │
            ▼                        ▼                        ▼
    ┌───────────────┐        ┌───────────────┐        ┌───────────────┐
    │   lb1         │        │   lb2         │        │   (Future)    │
    │   107         │◄──────►│   108         │        │               │
    │   MASTER      │  VRRP  │   BACKUP      │        │               │
    │   Priority 100│        │   Priority 90 │        │               │
    │   Keepalived  │        │   Keepalived  │        │               │
    │   HAProxy     │        │   HAProxy     │        │               │
    └───────────────┘        └───────────────┘        └───────────────┘
            │                        │
            └────────────┬───────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Backend-1  │  │   Backend-2  │  │   Backend-3  │
│   101        │  │   106        │  │   100        │
│   mastertwo  │  │ masterthree  │  │   masterone  │
└──────────────┘  └──────────────┘  └──────────────┘
```

---

> ⚠️ **উভয় নোডে (lb1 এবং lb2) এই কমান্ডগুলো রান করতে হবে।**

---

# ১. সিস্টেম আপডেট এবং প্যাকেজ ইন্সটল

```bash
# প্যাকেজ লিস্ট আপডেট
sudo apt update

# সিস্টেম আপগ্রেড
sudo apt upgrade -y

# HAProxy এবং Keepalived ইন্সটল
sudo apt install -y haproxy keepalived
```

---

# ২. ফায়ারওয়াল বন্ধ করুন (ল্যাব এনভায়রনমেন্ট)

```bash
sudo ufw disable
sudo systemctl disable --now ufw
```

---

# ৩. Swap বন্ধ করুন

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

---

# ৪. ইন্সটলেশন ভেরিফাই

```bash
haproxy -v

keepalived -v
```

---

# ৫. হেলথ চেক স্ক্রিপ্ট ডিরেক্টরি তৈরি

```bash
sudo mkdir -p /etc/keepalived/scripts
```

---

# ৬. API Server Health Check Script

```bash
sudo tee /etc/keepalived/scripts/check_apiserver.sh >/dev/null <<'EOF'
#!/bin/bash

errorExit() {
    echo "*** $@" 1>&2
    exit 1
}

check_api() {
    local url="$1"
    curl --silent --max-time 2 --insecure "${url}/healthz" \
    -o /dev/null || errorExit "Error GET ${url}/healthz"
}

check_api "https://localhost:6443"

if ip addr | grep -q 192.168.0.99; then
    check_api "https://192.168.0.99:6443"
fi

exit 0
EOF
```

---

# ৭. স্ক্রিপ্ট এক্সিকিউটেবল করুন

```bash
sudo chmod +x /etc/keepalived/scripts/check_apiserver.sh
```

---

# ৮. Keepalived MASTER Configuration (lb1)

* Node: **lb1**
* IP: **192.168.0.107**
* Priority: **100**
* State: **MASTER**

```bash
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<'EOF'
vrrp_script check_apiserver {
    script "/etc/keepalived/scripts/check_apiserver.sh"
    interval 3
    timeout 10
    fall 5
    rise 2
    weight -2
}

vrrp_instance VI_1 {
    state MASTER

    interface eth0

    virtual_router_id 1

    priority 100

    advert_int 1

    authentication {
        auth_type PASS
        auth_pass mysecret
    }

    virtual_ipaddress {
        192.168.0.99
    }

    track_script {
        check_apiserver
    }
}
EOF
```

---

# ৯. Keepalived BACKUP Configuration (lb2)

* Node: **lb2**
* IP: **192.168.0.108**
* Priority: **90**
* State: **BACKUP**

```bash
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<'EOF'
vrrp_script check_apiserver {
    script "/etc/keepalived/scripts/check_apiserver.sh"
    interval 3
    timeout 10
    fall 5
    rise 2
    weight -2
}

vrrp_instance VI_1 {
    state BACKUP

    interface eth0

    virtual_router_id 1

    priority 90

    advert_int 1

    authentication {
        auth_type PASS
        auth_pass mysecret
    }

    virtual_ipaddress {
        192.168.0.99
    }

    track_script {
        check_apiserver
    }
}
EOF
```

---

# ১০. Keepalived সার্ভিস চালু করুন

```bash
sudo systemctl enable keepalived

sudo systemctl restart keepalived

sudo systemctl status keepalived --no-pager
```

---

> ⚠️ **HAProxy কনফিগারেশন উভয় নোডে (lb1 এবং lb2) একই হবে।**

---

# ১১. HAProxy Configuration

```bash
sudo tee /etc/haproxy/haproxy.cfg >/dev/null <<'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice

    maxconn 4096

    user haproxy
    group haproxy

    stats socket /var/run/haproxy.sock mode 600 level admin

    chroot /var/lib/haproxy

    daemon

defaults
    log global

    mode tcp

    option tcplog

    option dontlognull

    retries 3

    timeout connect 10s
    timeout client 20s
    timeout server 20s

    option httpchk GET /healthz
    http-check expect status 200

frontend kubernetes-frontend
    bind 192.168.0.99:6443

    mode tcp

    option tcplog

    maxconn 2000

    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp

    option ssl-hello-chk

    balance roundrobin

    server backend1 192.168.0.101:6443 check fall 3 rise 2
    server backend2 192.168.0.106:6443 check fall 3 rise 2
    server backend3 192.168.0.100:6443 check fall 3 rise 2
EOF
```

---

# ১২. HAProxy সার্ভিস চালু করুন

```bash
sudo systemctl enable haproxy

sudo systemctl restart haproxy

sudo systemctl status haproxy --no-pager
```

---

# ১৩. Health Check

```bash
curl -k https://192.168.0.99:6443/healthz
```

Expected Output:

```text
ok
```

---

# ১৪. দরকারি কমান্ডসমূহ

## সার্ভিস স্ট্যাটাস

```bash
sudo systemctl status keepalived

sudo systemctl status haproxy
```

## সার্ভিস রিস্টার্ট

```bash
sudo systemctl restart keepalived

sudo systemctl restart haproxy
```

## VIP চেক

```bash
ip a s | grep 192.168.0.99
```

## API Health Check

```bash
curl -k https://192.168.0.99:6443/healthz
```

---

# Configuration Summary

| Component       | Value           |
| --------------- | --------------- |
| VIP             | 192.168.0.99    |
| API Port        | 6443            |
| Load Balancer 1 | 192.168.0.107   |
| Load Balancer 2 | 192.168.0.108   |
| Backend 1       | 192.168.0.101   |
| Backend 2       | 192.168.0.106   |
| Backend 3       | 192.168.0.100   |
| Load Balancing  | Round Robin     |
| VRRP ID         | 1               |
| Authentication  | PASS            |
| Keepalived Mode | MASTER / BACKUP |

---
