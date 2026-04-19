# Network Architecture

This document describes the complete network topology, design decisions, and how traffic flows through the homelab.

---

## Design Overview

The network uses a **two-subnet architecture** to separate management/WAN traffic from internal lab traffic:

- **WAN subnet (192.168.0.0/24):** The Cox router side — Proxmox management, OPNsense WAN uplink, and switch management (VLAN 1)
- **LAN subnets (10.0.x.0/24):** Behind OPNsense — all lab devices segmented into VLANs

OPNsense acts as the gateway between the two, performing NAT, firewall filtering, DHCP, and DNS for all LAN VLANs.

---

## Physical Topology

```
Internet
    |
Cox Router (192.168.0.1)
    |
    ├── Wi-Fi: Gaming PC (192.168.0.128) — Proxmox mgmt access
    |
    └── Ethernet (eno1): Proxmox Node (192.168.0.50)
                              |
                         vmbr0 (WAN)
                              |
                    OPNsense WAN (192.168.0.135)
                    OPNsense LAN (vtnet1) ←── vmbr1 (VLAN-aware)
                                                    |
                                              enp2s0f0 (I350 f0)
                                                    |
                                          JGS516PE Switch
                                           ├── Port 1: Trunk
                                           ├── Port 2: Gaming PC (VLAN 10)
                                           └── Ports 3-16: Available
```

---

## VLAN Design

Each VLAN gets a /24 subnet with the VLAN ID embedded in the third octet for easy identification.

| VLAN ID | Name | Subnet | Gateway | DHCP Range | Purpose |
|---------|------|--------|---------|------------|---------|
| 1 | Switch Mgmt | 192.168.0.0/24 | N/A | N/A | Switch management (untagged) |
| 10 | Trusted | 10.0.10.0/24 | 10.0.10.1 | .100-.200 | Personal devices |
| 20 | Lab | 10.0.20.0/24 | 10.0.20.1 | .100-.200 | Lab VMs and services |
| 30 | IoT | 10.0.30.0/24 | 10.0.30.1 | .100-.200 | Smart home devices |
| 40 | Guest | 10.0.40.0/24 | 10.0.40.1 | .100-.200 | Guest/untrusted devices |

---

## IP Assignments

### WAN Subnet (192.168.0.0/24)
| Device | IP | Notes |
|--------|----|-------|
| Cox Router | 192.168.0.1 | ISP gateway |
| Proxmox | 192.168.0.50 | Management UI (vmbr0) |
| OPNsense WAN | 192.168.0.135 | DHCP from Cox router |
| Gaming PC Wi-Fi | 192.168.0.128 | DHCP, used for Proxmox mgmt |
| JGS516PE Switch | 192.168.0.239 | Management IP (VLAN 1) |

### VLAN 10 — Trusted
| Device | IP | Notes |
|--------|----|-------|
| OPNsense | 10.0.10.1 | Gateway |
| Gaming PC | 10.0.10.10 | Switch Port 2 |

### VLAN 20 — Lab
| Device | IP | Notes |
|--------|----|-------|
| OPNsense | 10.0.20.1 | Gateway |
| Ubuntu Server | 10.0.20.20 | VM 100 (tag=20) |

---

## Traffic Flow Examples

### Gaming PC → Internet
```
Gaming PC (10.0.10.10)
 → Switch Port 2 (untagged, PVID 10)
  → Switch tags as VLAN 10
   → Port 1 trunk (tagged VLAN 10)
    → enp2s0f0 → vmbr1
     → OPNsense vtnet1 (trunk)
      → OPNsense Trusted interface (10.0.10.1)
       → Firewall check: pass
        → NAT → WAN (192.168.0.135)
         → Cox Router → Internet
```

### Gaming PC → Ubuntu Server (Inter-VLAN)
```
Gaming PC (10.0.10.10)
 → Default gateway: 10.0.10.1 (OPNsense)
  → OPNsense: route 10.0.20.20 → Lab interface
   → Firewall check: pass
    → OPNsense sends via VLAN 20
     → vmbr1 delivers to VM 100 (tag=20)
      → Ubuntu Server (10.0.20.20)
```

---

## Proxmox Virtual Networking

### Bridge Configuration
| Bridge | Physical NIC | Purpose | VLAN Aware |
|--------|-------------|---------|------------|
| vmbr0 | nic0/eno1 | WAN — Cox router uplink | No |
| vmbr1 | enp2s0f0/f0 | LAN — VLAN trunk to switch | Yes |
| vmbr2 | enp2s0f1/f1 | OPT1 — unused | No |

### vmbr1 Configuration
```
auto vmbr1
iface vmbr1 inet manual
        bridge-ports enp2s0f0
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 1-4094
        bridge-pvid 1
```

### VM Port Assignments
| VM | Port Type | Proxmox Config | Behavior |
|----|-----------|---------------|----------|
| OPNsense (net1) | Trunk | trunks=10;20;30;40 | Receives all VLANs tagged |
| Ubuntu Server (net0) | Access | tag=20 | Receives VLAN 20 only, untagged |

---

## Switch Configuration (JGS516PE)

| Port | Connected To | VLAN Config | PVID |
|------|-------------|-------------|------|
| 1 | EliteDesk f0 | T: 10,20,30,40 / U: VLAN 1 | 1 |
| 2 | Gaming PC | U: VLAN 10 | 10 |
| 3-16 | Available | Default (VLAN 1) | 1 |

---

## Design Decisions

**Why two subnets?** Proxmox management (vmbr0) and OPNsense WAN both need to reach the Cox router on 192.168.0.0/24. Placing LAN traffic on a separate 10.0.x.0/24 range avoids routing conflicts and provides clean separation.

**Why router-on-a-stick?** A single trunk carries all VLANs between OPNsense and the switch. This is simpler than dedicating a physical NIC per VLAN and mirrors how enterprise networks handle inter-VLAN routing.

**Why VLAN-aware on vmbr1?** Without VLAN-aware, the bridge passes all traffic transparently but cannot assign VMs to specific VLANs. With VLAN-aware enabled, Proxmox acts as a virtual managed switch — OPNsense gets a trunk port, and other VMs get access ports on specific VLANs.

**Why PVID 1 on vmbr1?** The switch management interface uses VLAN 1 (untagged). Without `bridge-pvid 1`, the VLAN-aware bridge drops untagged traffic, making the switch unreachable.
