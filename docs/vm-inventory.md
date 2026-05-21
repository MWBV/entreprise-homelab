# VM Inventory

> **Phase:** Network Segmentation
>  **Last updated:** May, 20th 2026
>  **Status:** Draft

------

## Overview

Complete inventory of all virtual machines in the enterprise-homelab environment. All VMs run on a single Dell PowerEdge R440 managed by Proxmox VE. This document is the single source of truth for VM IDs, IPs, resource allocation, and status.

------

## Hardware Summary

| Property          | Value               |
| ----------------- | ------------------- |
| Host              | Dell PowerEdge R440 |
| Hypervisor        | Proxmox VE          |
| Total RAM         | 64 GB               |
| Allocated RAM     | ~51 GB              |
| Headroom          | ~13 GB              |
| Total Storage     | ~1.5 TB             |
| Allocated Storage | ~1 TB               |

------

## Full VM Table

| VM ID | Hostname   | OS                | vCPU | RAM  | Disk   | VLAN       | IP          | Status |
| ----- | ---------- | ----------------- | ---- | ---- | ------ | ---------- | ----------- | ------ |
| 100   | pfsense    | FreeBSD (pfSense) | 2    | 2 GB | 32 GB  | 10 (trunk) | 10.10.10.1  | 🔲      |
| 101   | dns01      | Ubuntu 22.04      | 2    | 2 GB | 20 GB  | 20         | 10.10.20.10 | 🔲      |
| 102   | ipa01      | Rocky Linux 9     | 2    | 4 GB | 40 GB  | 20         | 10.10.20.20 | 🔲      |
| 103   | fog01      | Ubuntu 22.04      | 2    | 4 GB | 200 GB | 30         | 10.10.30.10 | 🔲      |
| 104   | samba01    | Ubuntu 22.04      | 2    | 4 GB | 300 GB | 30         | 10.10.30.20 | 🔲      |
| 105   | squid01    | Ubuntu 22.04      | 1    | 2 GB | 20 GB  | 30         | 10.10.30.30 | 🔲      |
| 106   | proxy01    | Ubuntu 22.04      | 1    | 1 GB | 10 GB  | 40         | 10.10.40.10 | 🔲      |
| 107   | gitea01    | Ubuntu 22.04      | 2    | 4 GB | 40 GB  | 40         | 10.10.40.20 | 🔲      |
| 108   | pg01       | Ubuntu 22.04      | 2    | 4 GB | 60 GB  | 60         | 10.10.60.10 | 🔲      |
| 109   | suricata01 | Ubuntu 22.04      | 2    | 4 GB | 40 GB  | 50         | 10.10.50.20 | 🔲      |
| 110   | wazuh01    | Ubuntu 22.04      | 4    | 8 GB | 100 GB | 50         | 10.10.50.10 | 🔲      |
| 111   | ws-win01   | Windows 10        | 2    | 4 GB | 60 GB  | 30         | DHCP        | 🔲      |
| 112   | ws-linux01 | Ubuntu Desktop    | 2    | 4 GB | 60 GB  | 30         | DHCP        | 🔲      |

**Status key:** 🔲 Not created · 🔄 In progress · ✅ Running · ⚠️ Issue

------

## Per-VM Detail

### VM 100 — pfsense

| Property   | Value                                       |
| ---------- | ------------------------------------------- |
| Role       | Gateway, firewall, VLAN routing, DHCP relay |
| OS         | FreeBSD (pfSense CE)                        |
| VLAN       | 10 (trunk — carries all VLANs)              |
| IP         | 10.10.10.1                                  |
| Web UI     | https://10.10.10.1                          |
| Proxmox ID | 100                                         |
| Boot order | 1st — must be running before any other VM   |

**Services:** pfSense firewall, inter-VLAN routing, DHCP relay to Kea, syslog to Wazuh

------

### VM 101 — dns01

| Property   | Value                                    |
| ---------- | ---------------------------------------- |
| Role       | Authoritative + caching DNS, DHCP server |
| OS         | Ubuntu 22.04 LTS                         |
| VLAN       | 20 — Infrastructure                      |
| IP         | 10.10.20.10                              |
| Proxmox ID | 101                                      |

**Services:** BIND9 (port 53), ISC Kea DHCP (port 67)
 **Config files:** `/etc/bind/named.conf`, `/etc/kea/kea-dhcp4.conf`
 **Zone:** `lab.local`

------

### VM 102 — ipa01

| Property   | Value                        |
| ---------- | ---------------------------- |
| Role       | Identity & Access Management |
| OS         | Rocky Linux 9                |
| VLAN       | 20 — Infrastructure          |
| IP         | 10.10.20.20                  |
| Proxmox ID | 102                          |

**Services:** FreeIPA (Kerberos TCP/UDP 88, LDAP 389, LDAPS 636)
 **Kerberos realm:** `LAB.LOCAL`
 **Domain:** `lab.local`
 **Web UI:** https://10.10.20.20/ipa/ui

------

### VM 103 — fog01

| Property   | Value                         |
| ---------- | ----------------------------- |
| Role       | PXE boot, workstation imaging |
| OS         | Ubuntu 22.04 LTS              |
| VLAN       | 30 — Corporate                |
| IP         | 10.10.30.10                   |
| Proxmox ID | 103                           |

**Services:** FOG Project (TFTP, HTTP, PXE), NFS for image storage
 **Web UI:** http://10.10.30.10/fog
 **Note:** Large disk (200 GB) — stores OS images

------

### VM 104 — samba01

| Property   | Value                  |
| ---------- | ---------------------- |
| Role       | Department file shares |
| OS         | Ubuntu 22.04 LTS       |
| VLAN       | 30 — Corporate         |
| IP         | 10.10.30.20            |
| Proxmox ID | 104                    |

**Services:** Samba (SMB ports 139, 445)
 **Auth:** FreeIPA via Winbind
 **Shares:** IT, Engineering, Corporate, Security
 **Note:** Largest disk (300 GB) — dedicated to share storage

------

### VM 105 — squid01

| Property   | Value                                     |
| ---------- | ----------------------------------------- |
| Role       | Forward proxy, outbound traffic filtering |
| OS         | Ubuntu 22.04 LTS                          |
| VLAN       | 30 — Corporate                            |
| IP         | 10.10.30.30                               |
| Proxmox ID | 105                                       |

**Services:** Squid (port 3128)
 **Config:** `/etc/squid/squid.conf`
 **Logs forwarded to:** Wazuh on 10.10.50.10

------

### VM 106 — proxy01

| Property   | Value                          |
| ---------- | ------------------------------ |
| Role       | Reverse proxy, TLS termination |
| OS         | Ubuntu 22.04 LTS               |
| VLAN       | 40 — DMZ                       |
| IP         | 10.10.40.10                    |
| Proxmox ID | 106                            |

**Services:** HAProxy or Nginx (ports 80, 443)
 **Backend:** Gitea on 10.10.40.20:3000
 **TLS cert:** Self-signed or FreeIPA internal CA

------

### VM 107 — gitea01

| Property   | Value                    |
| ---------- | ------------------------ |
| Role       | Self-hosted Git platform |
| OS         | Ubuntu 22.04 LTS         |
| VLAN       | 40 — DMZ                 |
| IP         | 10.10.40.20              |
| Proxmox ID | 107                      |

**Services:** Gitea (Docker, port 3000), Gitea Actions / Woodpecker CI
 **Auth:** FreeIPA LDAP
 **Database:** PostgreSQL on 10.10.60.10
 **Compose file:** `/opt/gitea/docker-compose.yml`

------

### VM 108 — pg01

| Property   | Value               |
| ---------- | ------------------- |
| Role       | Relational database |
| OS         | Ubuntu 22.04 LTS    |
| VLAN       | 60 — Databases      |
| IP         | 10.10.60.10         |
| Proxmox ID | 108                 |

**Services:** PostgreSQL (port 5432)
 **Accessible from:** DMZ (VLAN 40) only
 **Databases:** `gitea`
 **Note:** Most isolated VM — no internet access, no Corporate access

------

### VM 109 — suricata01

| Property   | Value            |
| ---------- | ---------------- |
| Role       | Network IDS      |
| OS         | Ubuntu 22.04 LTS |
| VLAN       | 50 — Security    |
| IP         | 10.10.50.20      |
| Proxmox ID | 109              |

**Services:** Suricata (promiscuous mode)
 **Ruleset:** ET Open (via suricata-update)
 **Log:** `/var/log/suricata/eve.json` → forwarded to Wazuh
 **Note:** Requires a second NIC in promiscuous mode with port mirroring from pfSense

------

### VM 110 — wazuh01

| Property   | Value                                   |
| ---------- | --------------------------------------- |
| Role       | SIEM, HIDS, centralized log correlation |
| OS         | Ubuntu 22.04 LTS                        |
| VLAN       | 50 — Security                           |
| IP         | 10.10.50.10                             |
| Proxmox ID | 110                                     |

**Services:** Wazuh Manager (port 1514/1515), Wazuh Dashboard (OpenSearch, port 443)
 **Agents:** All VMs + both workstations
 **Log sources:** pfSense syslog, Suricata eve.json, Squid access.log, all agents
 **Web UI:** https://10.10.50.10

------

### VM 111 — ws-win01

| Property   | Value                         |
| ---------- | ----------------------------- |
| Role       | Windows corporate workstation |
| OS         | Windows 10                    |
| VLAN       | 30 — Corporate                |
| IP         | DHCP (10.10.30.100–200)       |
| Proxmox ID | 111                           |

**Auth:** FreeIPA via SSSD or realm join
 **Wazuh agent:** Installed
 **FOG image:** Captured from this VM as base Windows image

------

### VM 112 — ws-linux01

| Property   | Value                       |
| ---------- | --------------------------- |
| Role       | Linux corporate workstation |
| OS         | Ubuntu Desktop 22.04        |
| VLAN       | 30 — Corporate              |
| IP         | DHCP (10.10.30.100–200)     |
| Proxmox ID | 112                         |

**Auth:** FreeIPA (`ipa-client-install`)
 **Wazuh agent:** Installed
 **FOG image:** Captured from this VM as base Ubuntu image

------

## Resource Totals

| Resource | Allocated | Available          | Headroom |
| -------- | --------- | ------------------ | -------- |
| vCPU     | ~26       | ~32 (R440 typical) | ~6       |
| RAM      | ~51 GB    | 64 GB              | ~13 GB   |
| Storage  | ~1 TB     | ~1.5 TB            | ~500 GB  |

------

## Errors & Fixes

| VM                  | Error                                  | Cause                                | Fix                                                 |
| ------------------- | -------------------------------------- | ------------------------------------ | --------------------------------------------------- |
| <!-- e.g. ipa01 --> | <!-- e.g. ipa-server-install fails --> | <!-- e.g. hostname not resolving --> | <!-- e.g. set FQDN in /etc/hosts before install --> |

