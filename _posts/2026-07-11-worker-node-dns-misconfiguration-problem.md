---
layout: post
title: "Worker Node DNS Misconfiguration Problem"
date: 2026-07-11
categories: [k8s]
---

## Introduction

Even though **CoreDNS** is healthy in the **controlplane**, your worker nodes themselves cannot resolve external names (like google.com or registry-1.docker.io). That’s why Pods scheduled there fail to pull images.

```bash
vagrant@node01:~$ nslookup google.com
;; communications error to 127.0.0.53#53: timed out
;; communications error to 127.0.0.53#53: timed out
;; communications error to 127.0.0.53#53: timed out
;; no servers could be reached
```

**How to fix worker node DNS**

Replace the loopback resolver with public DNS servers:

```bash
sudo nano /etc/resolv.conf

# Replace the nameserver IP with the following
nameserver 8.8.8.8
nameserver 1.1.1.1

# Restart kubelet
sudo systemctl restart kubelet

# Verify fix
vagrant@node01:~$ nslookup google.com
Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
Name:   google.com
Address: 142.251.38.238
Name:   google.com
Address: 2a00:1450:4017:801::200e
```

**Make it persistent across reboots:**

On Ubuntu Vagrant boxes, **/etc/resolv.conf** may reset. To make it stick:

```bash
# Disable systemd‑resolved:
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved

# Remove the symlink:
sudo rm /etc/resolv.conf

# Create a new file:
sudo nano /etc/resolv.conf

# Add nameserver IP
nameserver 8.8.8.8
nameserver 1.1.1.1
```

> By putting **8.8.8.8** directly in **/etc/resolv.conf**, your node bypassed the broken local stub (127.0.0.53) and sent queries straight to a working DNS server.

**Summary:** CoreDNS is fine, but worker nodes cannot resolve external names because they’re stuck on 127.0.0.__. Fix **/etc/resolv.conf** to use real nameservers (like 8.8.8.8), restart kubelet, and your Pods will be able to pull images from Docker Hub.
