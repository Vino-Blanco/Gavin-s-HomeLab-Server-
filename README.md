# Gavin's HomeLab Server

A production-style homelab built on Proxmox VE, featuring network segmentation with VLANs, a virtualized OPNsense firewall/router, and containerized services — designed to mirror real-world enterprise infrastructure.

## Project Overview

This homelab serves as a hands-on learning environment for network engineering, systems administration, and security operations. Every component is documented from scratch, and the network is designed with the same principles used in production data center environments.

## Architecture

```
                        Internet
                           |
                    Cox Router (192.168.0.1)
                           |
                 eno1 → vmbr0 (WAN Bridge)
                           |
              ┌─────────────┴─────────────┐
              │                           │
     Proxmox Node                  OPNsense VM
    (192.168.0.50)              WAN: 192.168.0.135
                                LAN: VLAN Trunk
                                         │
                               vtnet1 → vmbr1
                             (VLAN-aware bridge)
                                         │
                              enp2s0f0 (I350-AM2)
                                         │
                            JGS516PE Managed Switch
                           ┌──────┴──────┐
                      Port 1          Port 2
                     (Trunk)        (Access)
                   T: 10,20,30,40   PVID: 10
                   U: VLAN 1
                                         │
                                    Gaming PC
                                  (10.0.10.10)

              ┌──── Ubuntu Server VM ────┐
              │   tag=20 on vmbr1        │
              │   10.0.20.20 (VLAN 20)   │
              │   Docker + Portainer     │
              └──────────────────────────┘
```

## VLAN Segmentation

| VLAN ID | Name | Subnet | Gateway | Purpose | Status |
|---------|------|--------|---------|---------|--------|
| 1 | Switch Mgmt | 192.168.0.0/24 | N/A | Switch management (untagged) | ✅ Working |
| 10 | Trusted | 10.0.10.0/24 | 10.0.10.1 | Personal devices | ✅ Working |
| 20 | Lab | 10.0.20.0/24 | 10.0.20.1 | Lab VMs and services | ✅ Working |
| 30 | IoT | 10.0.30.0/24 | 10.0.30.1 | Smart home devices | ✅ Working |
| 40 | Guest | 10.0.40.0/24 | 10.0.40.1 | Guest/untrusted devices | ✅ Working |
| 50 | Security Cams | 10.0.50.0/24 | 10.0.50.1 | Future camera network | ⏳ Planned |

## Hardware

| Component | Details |
|-----------|---------|
| Server | HP EliteDesk G4 600 (Intel i5-8500, 32GB RAM, 68GB NVMe + 1TB HDD) |
| Hypervisor | Proxmox VE 9.1.7 |
| NIC | 10Gtek Intel I350-AM2 dual-port PCIe |
| Switch | Netgear JGS516PE 16-port managed PoE |
| UPS | CyberPower CP1500AVR |

## Virtual Machines

| VM ID | Name | OS | VLAN | IP | Purpose |
|-------|------|----|------|----|---------|
| 100 | ubuntu-server | Ubuntu Server 24.04 LTS | 20 (Lab) | 10.0.20.20 | Docker host, containerized services |
| 101 | OPNsense | OPNsense 25.1 | Trunk (10,20,30,40) | 10.0.10.1 (Trusted) | Firewall, router, DHCP, DNS |

## Key Technical Details

### Network Design
- **Two-subnet architecture:** WAN (192.168.0.0/24) and LAN (10.0.x.0/24 VLANs)
- **Router-on-a-stick:** OPNsense handles all inter-VLAN routing through a single trunk
- **VLAN-aware bridge:** Proxmox vmbr1 acts as a virtual managed switch with trunk and access port assignments
- **OPNsense trunk:** `trunks=10;20;30;40` — receives all tagged VLAN traffic
- **VM access ports:** Ubuntu Server uses `tag=20` for VLAN 20 isolation
- **PVID 1 passthrough:** `bridge-pvid 1` on vmbr1 enables switch management via VLAN 1 untagged

### Services Running
- **OPNsense:** Stateful firewall, inter-VLAN routing, DHCP (all VLANs), Unbound DNS with DNSSEC
- **Docker + Portainer:** Container management on Ubuntu Server (http://10.0.20.20:9000)

## Build Progress

- [x] Proxmox VE installation and configuration
- [x] Ubuntu Server VM with static networking
- [x] Docker and Portainer deployment
- [x] OPNsense VM — firewall, routing, DHCP, DNS
- [x] VLAN segmentation (802.1Q) on switch and OPNsense
- [x] VLAN-aware bridge with trunk/access port config
- [x] Inter-VLAN routing verified
- [ ] Firewall rule lockdown (restrict inter-VLAN traffic)
- [ ] Monitoring stack (Grafana + InfluxDB + Telegraf)
- [ ] Active Directory lab rebuild
- [ ] GNS3/EVE-NG for routing protocol labs

## Documentation

| Document | Description |
|----------|-------------|
| [Build Phases](docs/build-phases.md) | Detailed walkthrough of each build phase |
| [Network Architecture](docs/network-architecture.md) | Complete network topology and design decisions |
| [Lessons Learned](docs/lessons-learned.md) | Troubleshooting experiences and key takeaways |

## Skills Demonstrated

- Bare-metal hypervisor deployment (Proxmox VE)
- Virtual machine provisioning and management
- Network segmentation with 802.1Q VLANs
- Firewall configuration and stateful rule management (OPNsense/pf)
- DHCP and DNS server administration
- Linux server administration (Ubuntu Server 24.04)
- Docker containerization and Portainer management
- Managed switch configuration (trunk/access ports, PVID)
- Two-subnet network architecture (WAN/LAN separation)
- Hypervisor-level virtual networking (VLAN-aware bridges, trunk/access VM ports)
- Systematic troubleshooting and root cause analysis

## About

Built by [Gavin White](https://www.linkedin.com/in/gavin-white-812345315) — aspiring Network Engineer with CompTIA Security+ (CE), currently studying for CCNA.
