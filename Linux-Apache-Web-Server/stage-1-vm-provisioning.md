# Linux + Apache Web Server Project — Stage 1: VM Provisioning & Base OS Installation

**Project:** Linux + Apache Web Server
**Stage:** 1 of 5 — Environment Setup
**Date:** August 2026
**Author:** Marshel Mburu Macharia

---

## Overview

This is the first stage of a multi-stage project to build, harden, and monitor a production-style Apache web server on Linux. The end goal is a portfolio piece demonstrating the full lifecycle of standing up a web server: base OS installation, service deployment, security hardening, log integration with a SIEM (Wazuh), and validation through simulated attack traffic from a Kali Linux VM.

Stage 1 focuses purely on infrastructure: provisioning the virtual machine, installing the base operating system, and establishing reliable remote access — the foundation everything else in the project builds on.

---

## Objectives

- Provision a dedicated virtual machine for the web server role
- Install Ubuntu Server 26.04 LTS as the base operating system
- Configure disk layout using LVM for future flexibility
- Establish SSH-based remote administration
- Validate connectivity and confirm the system is fully patched

---

## Environment

| Component | Detail |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Host OS | Windows 11 (ThinkPad) |
| Guest OS | Ubuntu Server 26.04 LTS |
| VM Name | `apache-web-01` |
| Allocated RAM | 2.5 GB |
| vCPUs | 2 |
| Disk | 25 GB (LVM, dynamically allocated) |
| Boot Mode | Legacy BIOS |
| Network Mode | NAT (host-only reachability via port forwarding) |

This VM sits alongside two existing lab VMs (a Kali Linux attack box and an Ubuntu/Wazuh SIEM instance) already in use for other projects in this portfolio, keeping the entire lab environment consolidated under one hypervisor.

---

## Build Process

### 1. VM Provisioning
Created a new VirtualBox VM configured for a 64-bit Ubuntu guest, with 2 vCPUs and 2.5 GB RAM — sufficient headroom for Apache plus future logging agents without over-provisioning host resources.

### 2. Disk & Storage Layout
Configured guided storage using an LVM volume group rather than a flat partition. This was a deliberate choice: LVM allows the root logical volume to be resized later without a rebuild, which matters for a server expected to accumulate logs and application data over the project's remaining stages. Disk encryption (LUKS) was intentionally left disabled, as it adds no security value in an isolated lab environment and would only introduce friction (boot-time passphrase prompts) during iterative testing.

### 3. Base OS Installation
Installed Ubuntu Server 26.04 LTS (the current long-term support release) using the standard, non-minimized server profile — preserving standard utilities and documentation that later stages (log inspection, service debugging) will rely on.

### 4. Remote Access Configuration
Enabled the OpenSSH server during installation to avoid dependency on the VirtualBox console for ongoing administration. Because the VM's network adapter is set to NAT — which isolates the guest into its own private network segment — direct SSH from the host was initially unreachable.

**Resolution:** Configured VirtualBox NAT port forwarding, mapping host port `2222` to guest port `22`, and permitted the connection through Windows Defender Firewall on the Private network profile. This restored SSH access from the host without exposing the VM beyond the local machine.

### 5. Validation
Confirmed the system was fully patched (`apt update && apt upgrade`, zero pending updates) and verified SSH connectivity end-to-end from a Windows terminal:

```
ssh -p 2222 marshel@127.0.0.1
```

---

## Troubleshooting Log

| Issue | Root Cause | Resolution |
|---|---|---|
| SSH connection timed out when targeting the VM's internal IP directly | Plain NAT mode isolates the guest into a private network invisible to the host | Configured host-to-guest port forwarding (2222 → 22) |
| SSH connection refused after adding the port forwarding rule | Windows Defender Firewall was blocking VirtualBox's networking on the Private profile | Granted VirtualBoxVM.exe explicit firewall permission on Private networks |

Documenting this here deliberately — real-world server deployments run into exactly this class of networking/firewall friction, and working through it methodically is as relevant a skill as the installation itself.

---

## Outcome

Stage 1 is complete. The result is a patched, LVM-backed Ubuntu Server 26.04 instance, fully administrable via SSH from the host machine, ready to serve as the base for Apache installation and configuration in Stage 2.

---

## Next Steps (Stage 2 and beyond)

1. Install and configure Apache HTTP Server, including a custom virtual host
2. Harden the web server (disable directory listing, suppress version banners, configure firewall rules, TLS)
3. Forward Apache access/error logs into the existing Wazuh SIEM instance
4. Generate attack traffic from the Kali Linux VM (recon scans, common web attack patterns) to validate detection coverage
5. Document findings with MITRE ATT&CK mappings, consistent with the rest of this portfolio

