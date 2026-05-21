# Architecture Overview

> **Phase:** Network Segmentation
> **Last updated:** May, 20th 2026
> **Status:** Draft

------

## Summary

This lab simulates a corporate enterprise network built on a single Dell PowerEdge R440 running Proxmox VE. It spans six isolated VLANs with Zero Trust firewall enforcement at the pfSense gateway, dedicated Linux services for DNS, DHCP, and identity management, and a full security monitoring stack with IDS and SIEM. The environment is designed to reflect real-world infrastructure patterns used in mid-size organizations.

------

## Hardware

| Property   | Value               |
| ---------- | ------------------- |
| Server     | Dell PowerEdge R440 |
| Hypervisor | Proxmox VE          |
| RAM        | 64 GB               |
| Storage    | ~1.5 TB             |
| Topology   | Single node         |

------

## Network Design

### VLAN Layout

| VLAN | Name           | Subnet        | Purpose                                      |
| ---- | -------------- | ------------- | -------------------------------------------- |
| 10   | Management     | 10.10.10.0/24 | Proxmox host, pfSense OOB, admin access only |
| 20   | Infrastructure | 10.10.20.0/24 | BIND9, Kea DHCP, FreeIPA                     |
| 30   | Corporate      | 10.10.30.0/24 | Workstations, FOG, Samba                     |
| 40   | DMZ            | 10.10.40.0/24 | Gitea, HAProxy, reverse proxy                |
| 50   | Security       | 10.10.50.0/24 | Wazuh, Suricata, SIEM                        |
| 60   | Databases      | 10.10.60.0/24 | PostgreSQL (isolated data tier)              |

### Logical Topology

```
Internet
    │
    ▼
[ pfSense Gateway ]
    │
    ├── VLAN 10  Management     10.10.10.0/24
    ├── VLAN 20  Infrastructure 10.10.20.0/24
    ├── VLAN 30  Corporate      10.10.30.0/24
    ├── VLAN 40  DMZ            10.10.40.0/24
    ├── VLAN 50  Security       10.10.50.0/24
    └── VLAN 60  Databases      10.10.60.0/24
```

> **Diagram:** See 

------

## VM Inventory

| VM               | OS             | vCPU         | RAM        | Disk      | VLAN |
| ---------------- | -------------- | ------------ | ---------- | --------- | ---- |
| pfSense          | FreeBSD        | 2            | 2 GB       | 32 GB     | 10   |
| BIND9 + Kea DHCP | Ubuntu 22.04   | 2            | 2 GB       | 20 GB     | 20   |
| FreeIPA          | Rocky Linux 9  | 2            | 4 GB       | 40 GB     | 20   |
| FOG Server       | Ubuntu 22.04   | 2            | 4 GB       | 200 GB    | 30   |
| Samba / NAS      | Ubuntu 22.04   | 2            | 4 GB       | 300 GB    | 30   |
| Gitea (Docker)   | Ubuntu 22.04   | 2            | 4 GB       | 40 GB     | 40   |
| HAProxy / Nginx  | Ubuntu 22.04   | 1            | 1 GB       | 10 GB     | 40   |
| Squid Proxy      | Ubuntu 22.04   | 1            | 2 GB       | 20 GB     | 30   |
| PostgreSQL       | Ubuntu 22.04   | 2            | 4 GB       | 60 GB     | 60   |
| Suricata         | Ubuntu 22.04   | 2            | 4 GB       | 40 GB     | 50   |
| Wazuh            | Ubuntu 22.04   | 4            | 8 GB       | 100 GB    | 50   |
| Workstation x2   | Win10 / Ubuntu | 2            | 4 GB ea    | 60 GB ea  | 30   |
| **Total**        |                | **~26 vCPU** | **~51 GB** | **~1 TB** |      |

------

## Security Design Principles

### Zero Trust Firewall Policy

Default rule: **DENY ALL**. Traffic is allowed only where explicitly required.

| Source              | Destination         | Allowed Traffic                  |
| ------------------- | ------------------- | -------------------------------- |
| Management (10)     | All                 | Full (admin IPs only)            |
| Infrastructure (20) | Infrastructure (20) | Internal service comms           |
| Corporate (30)      | Infrastructure (20) | DNS, DHCP, FreeIPA auth          |
| Corporate (30)      | DMZ (40)            | HTTP/HTTPS to Gitea only         |
| Corporate (30)      | Databases (60)      | DENY                             |
| DMZ (40)            | Databases (60)      | TCP port 5432 only               |
| Security (50)       | All                 | Inbound agent/monitoring traffic |
| Any                 | Internet            | Via Squid proxy on VLAN 30       |
| Inter-VLAN default  | —                   | DENY                             |

### Defense in Depth Layers

| Layer                | Implementation                               |
| -------------------- | -------------------------------------------- |
| Network segmentation | pfSense VLANs + Zero Trust rules             |
| Identity & access    | FreeIPA, Kerberos auth, HBAC policies        |
| Endpoint monitoring  | Wazuh agents on all VMs and workstations     |
| Network monitoring   | Suricata IDS on promiscuous interface        |
| Log aggregation      | Wazuh SIEM, pfSense, Suricata, Squid, agents |
| Outbound filtering   | Squid forward proxy with access logging      |
| Code security        | Semgrep SAST + Gitleaks in CI/CD pipeline    |
| TLS termination      | HAProxy in front of all DMZ services         |

------

## Service Map

```
VLAN 20 — Infrastructure
  └── BIND9       → authoritative DNS for lab.local
  └── Kea DHCP    → per-VLAN address assignment via relay
  └── FreeIPA     → Kerberos + LDAP identity for all VMs

VLAN 30 — Corporate
  └── FOG Server  → PXE boot, workstation imaging
  └── Samba       → department file shares (ACLs via FreeIPA)
  └── Squid       → outbound proxy, logging, filtering
  └── Workstations → Windows 10, Ubuntu Desktop

VLAN 40 — DMZ
  └── HAProxy     → TLS termination, reverse proxy for Gitea
  └── Gitea       → self-hosted Git (Docker), FreeIPA LDAP auth

VLAN 50 — Security
  └── Suricata    → IDS, deep packet inspection, ET Open rules
  └── Wazuh       → SIEM, HIDS, FIM, centralized log correlation

VLAN 60 — Databases
  └── PostgreSQL  → backend for Gitea, accessible from DMZ only
```

------

## Data Flow Examples

### Developer pushes code to Gitea

```
Workstation (VLAN 30)
  → HAProxy (VLAN 40, HTTPS:443)
    → Gitea (VLAN 40)
      → PostgreSQL (VLAN 60, TCP:5432)
        → Gitea Actions triggers CI/CD pipeline
          → Semgrep SAST + Gitleaks scan
            → Pass: deploy  |  Fail: block merge
```

### Workstation authenticates on login

```
Workstation (VLAN 30)
  → FreeIPA (VLAN 20, Kerberos TCP:88)
    → TGT issued
      → Workstation accesses Samba share (VLAN 30)
        → Samba validates group membership via FreeIPA
          → ACL enforced
```

### Suricata detects suspicious traffic

```
Promiscuous NIC (port mirror from pfSense)
  → Suricata matches ET Open rule
    → Alert written to eve.json
      → Wazuh agent picks up log
        → Wazuh manager correlates + alerts dashboard
```

------

## Repository Structure

```
enterprise-homelab/
├── README.md
├── docs/
│   ├── architecture-overview.md   ← this file
│   ├── vlan-design.md
│   ├── firewall-rules.md
│   └── vm-inventory.md
├── diagrams/
│   ├── logical-topology.drawio
│   └── physical-topology.drawio
├── configs/
│   ├── pfsense/
│   ├── bind9/
│   ├── kea/
│   ├── samba/
│   ├── gitea/
│   ├── haproxy/
│   ├── squid/
│   └── wazuh/
└── cicd/
    └── .gitea/workflows/
        └── pipeline.yml
```

