# entreprise-homelab
Homelab simulating a corporate network with VLAN segmentation, centralized auth, IDS/SIEM monitoring and automated security testing in self host CI/CD pipeline.

Built on a Dell Poweredge R440 running proxmox VE. Every service is deployed in its dedicated Linux VM or container, segmented accross different isolated VLANs with enforced firewalls rules handled by Pfsense.

## Architecture

### Network Topology

```
Internet
    │
    ▼
[ pfSense Gateway ]
    │
    ├── VLAN 10  Management    10.10.10.0/24   Proxmox host, OOB admin access
    ├── VLAN 20  Infrastructure 10.10.20.0/24  BIND9, Kea DHCP, FreeIPA
    ├── VLAN 30  Corporate     10.10.30.0/24   Workstations, FOG, Samba
    ├── VLAN 40  DMZ           10.10.40.0/24   Gitea, HAProxy, reverse proxy
    ├── VLAN 50  Security      10.10.50.0/24   Wazuh, Suricata, SIEM
    └── VLAN 60  Databases     10.10.60.0/24   PostgreSQL (isolated)
```

### VM Inventory

| VM               | OS             | RAM     | Disk     | VLAN |
| ---------------- | -------------- | ------- | -------- | ---- |
| pfSense          | FreeBSD        | 2 GB    | 32 GB    | 10   |
| BIND9 + Kea DHCP | Ubuntu 22.04   | 2 GB    | 20 GB    | 20   |
| FreeIPA          | Rocky Linux 9  | 4 GB    | 40 GB    | 20   |
| FOG Server       | Ubuntu 22.04   | 4 GB    | 200 GB   | 30   |
| Samba / NAS      | Ubuntu 22.04   | 4 GB    | 300 GB   | 30   |
| Gitea (Docker)   | Ubuntu 22.04   | 4 GB    | 40 GB    | 40   |
| HAProxy / Nginx  | Ubuntu 22.04   | 1 GB    | 10 GB    | 40   |
| Squid Proxy      | Ubuntu 22.04   | 2 GB    | 20 GB    | 30   |
| PostgreSQL       | Ubuntu 22.04   | 4 GB    | 60 GB    | 60   |
| Suricata         | Ubuntu 22.04   | 4 GB    | 40 GB    | 50   |
| Wazuh            | Ubuntu 22.04   | 8 GB    | 100 GB   | 50   |
| Workstations x2  | Win10 / Ubuntu | 4 GB ea | 60 GB ea | 30   |

**Hardware:** Dell PowerEdge R440 · 64 GB RAM · ~1.5 TB storage · Single-node Proxmox VE

------

## Tech Stack

| Category            | Technology                      |
| ------------------- | ------------------------------- |
| Hypervisor          | Proxmox VE                      |
| Firewall / Routing  | pfSense                         |
| DNS                 | BIND9                           |
| DHCP                | ISC Kea                         |
| Identity & Auth     | FreeIPA (Kerberos + LDAP)       |
| Workstation Imaging | FOG Project (PXE)               |
| File Shares         | Samba (ACLs via FreeIPA groups) |
| Source Control      | Gitea (Docker)                  |
| Reverse Proxy       | HAProxy / Nginx                 |
| Forward Proxy       | Squid                           |
| Database            | PostgreSQL                      |
| IDS                 | Suricata (ET Open ruleset)      |
| SIEM / HIDS         | Wazuh (Manager + Dashboard)     |
| SAST                | Semgrep                         |
| Secrets Scanning    | Gitleaks                        |
| CI/CD               | Gitea Actions / Woodpecker CI   |

------

## Phases

### Phase 1 — Hypervisor & Network Segmentation

Configure Proxmox VE storage pools and virtual switches on the R440. Deploy pfSense as the main gateway, create all six VLANs, and implement Zero Trust inter-VLAN firewall rules. No services are deployed until the network foundation is solid and isolation is verified.

### Phase 2 — Core Infrastructure Services

Replace pfSense built-ins with dedicated Linux services. BIND9 handles internal DNS for the `lab.local` domain. Kea manages per-VLAN DHCP via relay. FreeIPA centralizes authentication, user accounts, and HBAC policies for all VMs and workstations.

### Phase 3 — Workstation Deployment & Storage

FOG Server handles PXE boot and automated imaging of Windows and Linux workstations. Samba provides department file shares with ACLs enforced against FreeIPA groups — Engineering cannot access IT shares, and so on.

### Phase 4 — Development, Databases & Proxy Traffic Control

Gitea runs in Docker on the DMZ VLAN, authenticated via FreeIPA LDAP, and sits behind an HAProxy reverse proxy with TLS termination. PostgreSQL is isolated on VLAN 60 — only the DMZ tier can reach it on port 5432. Squid handles outbound traffic filtering and logging for the Corporate VLAN.

### Phase 5 — Security Monitoring & DevSecOps

Suricata performs deep packet inspection on a promiscuous interface with port mirroring from pfSense. Wazuh aggregates logs from all agents, pfSense, Suricata, and Squid — with custom alert rules for SSH failures, VLAN policy violations, and file integrity events. A CI/CD pipeline in Gitea runs Semgrep SAST and Gitleaks secrets scanning on every push, blocking merges to `main` if checks fail.

------

## Status

| Phase                                       | Status         |
| ------------------------------------------- | -------------- |
| Phase 1 — Hypervisor & Network Segmentation | [] Not started |
| Phase 2 — Core Infrastructure Services      | [] Not started |
| Phase 3 — Workstation Deployment & Storage  | [] Not started |
| Phase 4 — Development, Databases & Proxy    | [] Not started |
| Phase 5 — Security Monitoring & DevSecOps   | [] Not started |

------

## Setup Guide

> **Note:** This is not a step-by-step install guide. Each phase has its own detailed documentation under `/docs`. This section covers prerequisites and the recommended build order.

### Prerequisites

- Proxmox VE installed on bare metal (R440 or equivalent)
- At least one NIC with 802.1Q VLAN tagging support
- Static IP assigned to Proxmox management interface on VLAN 10
- ISO images downloaded for Ubuntu 22.04, Rocky Linux 9, FreeBSD (pfSense)

### Recommended Build Order

```
1. Proxmox VE setup → storage pools, bridges, VLAN trunk
2. pfSense VM → WAN/LAN, VLAN interfaces, firewall rules
3. BIND9 + Kea → internal DNS, per-VLAN DHCP pools
4. FreeIPA → Kerberos realm, user/group structure
5. Samba + FOG → shares, PXE imaging
6. Gitea + PostgreSQL + HAProxy → dev environment
7. Squid → outbound proxy on Corporate VLAN
8. Suricata → IDS on Security VLAN
9. Wazuh → agents on all VMs, dashboard, custom rules
10. CI/CD pipeline → Gitea Actions, Semgrep, Gitleaks
```

### Repository Structure

```
enterprise-homelab/
├── README.md
├── docs/
│   ├── architecture-overview.md
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

## 
