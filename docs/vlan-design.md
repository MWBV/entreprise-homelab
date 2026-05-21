# VLAN Design

> **Phase:** Network Segmentation
> **Last updated:** May, 20th 2026
> **Status:** Draft

------

## Overview

This document covers the VLAN layout used across the enterprise-homelab environment. Six VLANs segment the network into functional zones, each with a dedicated subnet and a defined purpose. Inter-VLAN traffic is controlled at the pfSense gateway using a Zero Trust deny-all policy with explicit allow rules.

------

## VLAN Table

| VLAN ID | Name           | Subnet        | Gateway    | Purpose                         |
| ------- | -------------- | ------------- | ---------- | ------------------------------- |
| 10      | Management     | 10.10.10.0/24 | 10.10.10.1 | pfSense OOB, admin access only  |
| 20      | Infrastructure | 10.10.20.0/24 | 10.10.20.1 | BIND9, Kea DHCP, FreeIPA        |
| 30      | Corporate      | 10.10.30.0/24 | 10.10.30.1 | Workstations, FOG, Samba, Squid |
| 40      | DMZ            | 10.10.40.0/24 | 10.10.40.1 | Gitea, HAProxy, reverse proxy   |
| 50      | Security       | 10.10.50.0/24 | 10.10.50.1 | Wazuh, Suricata, SIEM           |
| 60      | Databases      | 10.10.60.0/24 | 10.10.60.1 | PostgreSQL (isolated data tier) |

------

## Per-VLAN Detail

### VLAN 10 — Management

| Property | Value                         |
| -------- | ----------------------------- |
| Subnet   | 10.10.10.0/24                 |
| Gateway  | 10.10.10.1 (pfSense)          |
| DHCP     | Static only with no DHCP pool |
| Access   | Admin workstation IP only     |

**Hosts:**

| Hostname | IP         | Role               |
| -------- | ---------- | ------------------ |
|          |            |                    |
| pfsense  | 10.10.10.1 | Gateway / firewall |

**Notes:**

- No DHCP on this VLAN all addresses statically assigned
- Access locked to a single admin IP via pfSense firewall rule

------

### VLAN 20 — Infrastructure

| Property | Value                    |
| -------- | ------------------------ |
| Subnet   | 10.10.20.0/24            |
| Gateway  | 10.10.20.1 (pfSense)     |
| DHCP     | Static reservations only |
| Access   | Internal VMs only        |

**Hosts:**

| Hostname | IP          | Role                      |
| -------- | ----------- | ------------------------- |
| dns01    | 10.10.20.10 | BIND9 + Kea DHCP          |
| ipa01    | 10.10.20.20 | FreeIPA (Kerberos + LDAP) |

**Notes:**

- BIND9 handles authoritative DNS for `lab.local` and caching for external resolution
- Kea DHCP serves all other VLANs via DHCP relay configured on pfSense
- FreeIPA is the identity source for all Linux VMs and workstations

------

### VLAN 30 — Corporate

| Property | Value                                 |
| -------- | ------------------------------------- |
| Subnet   | 10.10.30.0/24                         |
| Gateway  | 10.10.30.1 (pfSense)                  |
| DHCP     | Kea pool: 10.10.30.100 – 10.10.30.200 |
| Access   | Workstations, internal services       |

**Hosts:**

| Hostname   | IP          | Role                       |
| ---------- | ----------- | -------------------------- |
| fog01      | 10.10.30.10 | FOG Project (PXE imaging)  |
| samba01    | 10.10.30.20 | Samba file shares          |
| squid01    | 10.10.30.30 | Squid forward proxy        |
| ws-win01   | DHCP        | Windows 10 workstation     |
| ws-linux01 | DHCP        | Ubuntu Desktop workstation |

**Notes:**

- All outbound internet traffic routed through Squid on 10.10.30.30
- Workstations authenticate via FreeIPA (VLAN 20) on login
- Samba ACLs enforced via FreeIPA group membership

------

### VLAN 40 — DMZ

| Property | Value                                    |
| -------- | ---------------------------------------- |
| Subnet   | 10.10.40.0/24                            |
| Gateway  | 10.10.40.1 (pfSense)                     |
| DHCP     | Static reservations only                 |
| Access   | Inbound from Corporate (HTTP/HTTPS only) |

**Hosts:**

| Hostname | IP          | Role                          |
| -------- | ----------- | ----------------------------- |
| proxy01  | 10.10.40.10 | HAProxy / Nginx reverse proxy |
| gitea01  | 10.10.40.20 | Gitea (Docker)                |

**Notes:**

- HAProxy terminates TLS and forwards to Gitea on port 3000
- Gitea backend IP is never directly exposed to Corporate VLAN
- DMZ can reach Databases VLAN only on TCP port 5432

------

### VLAN 50 — Security

| Property | Value                                |
| -------- | ------------------------------------ |
| Subnet   | 10.10.50.0/24                        |
| Gateway  | 10.10.50.1 (pfSense)                 |
| DHCP     | Static reservations only             |
| Access   | Inbound agent traffic from all VLANs |

**Hosts:**

| Hostname   | IP          | Role                      |
| ---------- | ----------- | ------------------------- |
| wazuh01    | 10.10.50.10 | Wazuh Manager + Dashboard |
| suricata01 | 10.10.50.20 | Suricata IDS              |

**Notes:**

- Suricata runs on a promiscuous NIC with port mirroring from pfSense
- Wazuh agents on all VMs communicate back to 10.10.50.10
- Security VLAN has no outbound access except DNS and NTP

------

### VLAN 60 — Databases

| Property | Value                               |
| -------- | ----------------------------------- |
| Subnet   | 10.10.60.0/24                       |
| Gateway  | 10.10.60.1 (pfSense)                |
| DHCP     | Static reservations only            |
| Access   | DMZ (VLAN 40) on TCP port 5432 only |

**Hosts:**

| Hostname | IP          | Role       |
| -------- | ----------- | ---------- |
| pg01     | 10.10.60.10 | PostgreSQL |

**Notes:**

- Most restricted VLAN in the environment
- No inbound access from Corporate or Management
- No outbound internet access
- Wazuh agent traffic is the only exception to isolation (TCP to VLAN 50)

------

## Proxmox Bridge Configuration

All VLANs are carried over a single tagged trunk on `vmbr0`. Each VM connects to the trunk bridge with its VLAN tag set in the Proxmox network interface config.

```
# /etc/network/interfaces (Proxmox host)

auto vmbr0
iface vmbr0 inet static
    address 10.10.10.10/24
    gateway 10.10.10.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10 20 30 40 50 60
```

> VMs specify their VLAN tag in the Proxmox network device config, not in the guest OS.

------

## DHCP Relay (pfSense)

Kea DHCP on VLAN 20 (10.10.20.10) serves all VLANs via relay. Each pfSense VLAN interface has a DHCP relay rule configured:

| VLAN Interface      | Relay Target |
| ------------------- | ------------ |
| VLAN 30 (Corporate) | 10.10.20.10  |
| VLAN 40 (DMZ)       | 10.10.20.10  |
| VLAN 50 (Security)  | 10.10.20.10  |
| VLAN 60 (Databases) | 10.10.20.10  |

> Management (10) and Infrastructure (20) use static IPs only — no relay needed.

------

## Verification

Confirm VLAN isolation is working after pfSense is configured:

```bash
# From a Corporate VLAN host — should FAIL
ping 10.10.60.10

# From a Corporate VLAN host — should SUCCEED
ping 10.10.20.10   # DNS
ping 10.10.20.20   # FreeIPA

# From DMZ host — should SUCCEED on port 5432 only
nc -zv 10.10.60.10 5432

# From DMZ host — should FAIL
ping 10.10.30.10   # Cannot reach Corporate
```

------

## Errors & Fixes

| Error                                       | Cause                              | Fix                                                          |
| ------------------------------------------- | ---------------------------------- | ------------------------------------------------------------ |
| <!-- e.g. DHCP not assigning on VLAN 30 --> | <!-- e.g. relay not configured --> | <!-- e.g. added ip helper-address on pfSense VLAN interface --> |

