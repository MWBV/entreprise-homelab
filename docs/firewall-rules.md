# Firewall Rules

> **Phase:** Network Segmentation
>  **Last updated:** May, 20th 2026
>  **Status:** Draft

------

## Overview

All firewall rules are enforced at the pfSense gateway. The default policy is **deny all**; traffic is only permitted where explicitly required by a service dependency. Rules are applied per VLAN interface in pfSense and evaluated top-down.

------

## Design Principles

- **Default deny:** No inter-VLAN traffic unless explicitly allowed
- **Least privilege:** Each VLAN can only reach what it needs to function
- **Unidirectional where possible:** Corporate can reach Infrastructure, not the reverse
- **Logging enabled:** All deny rules log to pfSense syslog, forwarded to Wazuh
- **Admin access locked to IP:** Management VLAN rules restricted to a single admin source IP

------

## Rule Summary Table

| #    | Source VLAN         | Destination VLAN    | Protocol | Port(s)      | Action  | Purpose                               |
| ---- | ------------------- | ------------------- | -------- | ------------ | ------- | ------------------------------------- |
| 1    | Management (10)     | Any                 | Any      | Any          | ✅ Allow | Full admin access; admin IP only      |
| 2    | Infrastructure (20) | Infrastructure (20) | Any      | Any          | ✅ Allow | Internal service comms                |
| 3    | Infrastructure (20) | Internet            | UDP      | 53           | ✅ Allow | BIND9 external DNS forwarders         |
| 4    | Corporate (30)      | Infrastructure (20) | UDP/TCP  | 53           | ✅ Allow | DNS resolution                        |
| 5    | Corporate (30)      | Infrastructure (20) | UDP      | 67/68        | ✅ Allow | DHCP relay                            |
| 6    | Corporate (30)      | Infrastructure (20) | TCP      | 88, 389, 636 | ✅ Allow | FreeIPA Kerberos + LDAP               |
| 7    | Corporate (30)      | DMZ (40)            | TCP      | 443          | ✅ Allow | HTTPS to Gitea via HAProxy            |
| 8    | Corporate (30)      | Corporate (30)      | TCP      | 445, 139     | ✅ Allow | Samba file shares                     |
| 9    | Corporate (30)      | Security (50)       | TCP      | 1514, 1515   | ✅ Allow | Wazuh agent communication             |
| 10   | Corporate (30)      | Internet            | TCP      | 80, 443      | ✅ Allow | Via Squid proxy only                  |
| 11   | Corporate (30)      | Databases (60)      | Any      | Any          | 🚫 Deny  | No direct DB access from workstations |
| 12   | DMZ (40)            | Databases (60)      | TCP      | 5432         | ✅ Allow | Gitea → PostgreSQL                    |
| 13   | DMZ (40)            | Infrastructure (20) | TCP      | 389, 636     | ✅ Allow | Gitea LDAP auth via FreeIPA           |
| 14   | DMZ (40)            | Security (50)       | TCP      | 1514, 1515   | ✅ Allow | Wazuh agent communication             |
| 15   | DMZ (40)            | Corporate (30)      | Any      | Any          | 🚫 Deny  | DMZ cannot initiate to Corporate      |
| 16   | Security (50)       | Any                 | TCP      | 1514, 1515   | ✅ Allow | Wazuh agent inbound                   |
| 17   | Security (50)       | Any                 | Any      | Any          | 🚫 Deny  | Security VLAN is receive-only         |
| 18   | Databases (60)      | Security (50)       | TCP      | 1514, 1515   | ✅ Allow | Wazuh agent communication             |
| 19   | Databases (60)      | Any                 | Any      | Any          | 🚫 Deny  | Databases VLAN fully isolated         |
| 20   | Any                 | Any                 | Any      | Any          | 🚫 Deny  | Default deny — catch-all              |

------

## Per-Interface Rules

### VLAN 10: Management (Outbound)

```
# Allow admin workstation full access
pass in on vlan10 from <admin_ip> to any

# Block everything else on Management VLAN
block in on vlan10 all
```

> Replace `<admin_ip>` with your actual admin machine IP. Do not allow the entire /24.

------

### VLAN 20 — Infrastructure (Outbound)

```
# Allow intra-Infrastructure traffic
pass in on vlan20 from 10.10.20.0/24 to 10.10.20.0/24

# Allow BIND9 to reach external DNS forwarders
pass in on vlan20 proto udp from 10.10.20.10 to any port 53

# Block everything else
block in on vlan20 all
```

------

### VLAN 30 — Corporate (Outbound)

```
# DNS
pass in on vlan30 proto { udp tcp } from 10.10.30.0/24 to 10.10.20.10 port 53

# DHCP relay
pass in on vlan30 proto udp from 10.10.30.0/24 to 10.10.20.10 port { 67 68 }

# FreeIPA auth (Kerberos + LDAP)
pass in on vlan30 proto tcp from 10.10.30.0/24 to 10.10.20.20 port { 88 389 636 }

# Gitea via reverse proxy (HTTPS only)
pass in on vlan30 proto tcp from 10.10.30.0/24 to 10.10.40.10 port 443

# Samba shares (intra-Corporate)
pass in on vlan30 proto tcp from 10.10.30.0/24 to 10.10.30.20 port { 139 445 }

# Wazuh agent
pass in on vlan30 proto tcp from 10.10.30.0/24 to 10.10.50.10 port { 1514 1515 }

# Internet via Squid
pass in on vlan30 proto tcp from 10.10.30.0/24 to 10.10.30.30 port { 80 443 3128 }

# Explicit deny to Databases
block in on vlan30 from 10.10.30.0/24 to 10.10.60.0/24

# Default deny
block in on vlan30 all
```

------

### VLAN 40 — DMZ (Outbound)

```
# PostgreSQL
pass in on vlan40 proto tcp from 10.10.40.20 to 10.10.60.10 port 5432

# FreeIPA LDAP (Gitea SSO)
pass in on vlan40 proto tcp from 10.10.40.20 to 10.10.20.20 port { 389 636 }

# Wazuh agent
pass in on vlan40 proto tcp from 10.10.40.0/24 to 10.10.50.10 port { 1514 1515 }

# Explicit deny to Corporate
block in on vlan40 from 10.10.40.0/24 to 10.10.30.0/24

# Default deny
block in on vlan40 all
```

------

### VLAN 50 — Security (Outbound)

```
# Allow inbound Wazuh agent traffic from all VLANs (handled by source VLAN rules)
pass in on vlan50 proto tcp from any to 10.10.50.10 port { 1514 1515 }

# DNS only for Security VLAN hosts
pass in on vlan50 proto { udp tcp } from 10.10.50.0/24 to 10.10.20.10 port 53

# Default deny
block in on vlan50 all
```

------

### VLAN 60 — Databases (Outbound)

```
# Wazuh agent only
pass in on vlan60 proto tcp from 10.10.60.10 to 10.10.50.10 port { 1514 1515 }

# Default deny — fully isolated
block in on vlan60 all
```

------

## Logging

All `block` rules should have logging enabled in pfSense:

- Navigate to **Firewall → Rules → [VLAN interface]**
- Edit each block rule → enable **Log packets that are handled by this rule**
- pfSense logs forward to Wazuh via syslog (configured in Phase 5)

Logged fields per blocked packet: timestamp, source IP, destination IP, protocol, port, VLAN interface.

------

## Verification

After applying rules, test isolation from each VLAN:

```bash
# Corporate → Databases (should FAIL)
ping 10.10.60.10

# Corporate → DMZ on port 80 (should FAIL — HTTPS only)
nc -zv 10.10.40.10 80

# Corporate → DMZ on port 443 (should SUCCEED)
nc -zv 10.10.40.10 443

# DMZ → Databases on port 5432 (should SUCCEED)
nc -zv 10.10.60.10 5432

# DMZ → Corporate (should FAIL)
ping 10.10.30.10
```

------

## Errors & Fixes

| Error                                     | Cause                                        | Fix                                                          |
| ----------------------------------------- | -------------------------------------------- | ------------------------------------------------------------ |
| <!-- e.g. Wazuh agents not connecting --> | <!-- e.g. port 1514 blocked on Corporate --> | <!-- e.g. added explicit allow rule for 1514/1515 to VLAN 50 --> |

