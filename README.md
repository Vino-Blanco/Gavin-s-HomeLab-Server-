# My HomeLab Server

A production-style homelab built on Proxmox VE, featuring network segmentation with VLANs, a virtualized OPNsense firewall/router, a full observability stack (metrics + logs), and containerized services — designed to mirror real-world enterprise infrastructure.

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

              ┌──────── Ubuntu Server VM ────────┐
              │   tag=20 on vmbr1                │
              │   10.0.20.20 (VLAN 20 — Lab)     │
              │                                  │
              │   Metrics:                       │
              │   ├── Prometheus (9090)           │
              │   ├── Node Exporter (9100)        │
              │   ├── SNMP Exporter (9116)        │
              │   └── Grafana (3000)              │
              │                                  │
              │   Logs:                          │
              │   ├── Loki (3100)                │
              │   └── Grafana Alloy (12345)      │
              │                                  │
              │   Availability:                  │
              │   └── Uptime Kuma (3001)          │
              │                                  │
              │   Management:                    │
              │   └── Portainer (9000)            │
              └──────────────────────────────────┘
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

## Firewall Policy

| VLAN | Internet | Trusted | Lab | IoT | Guest | Firewall Mgmt |
|------|----------|---------|-----|-----|-------|---------------|
| Trusted (10) | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| Lab (20) | ✅ | ✅ | — | ❌ | ❌ | ✅ |
| IoT (30) | ✅ | ❌ | ❌ | — | ❌ | ❌ |
| Guest (40) | ✅ | ❌ | ❌ | ❌ | — | ❌ |

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
| 100 | ubuntu-server | Ubuntu Server 24.04 LTS | 20 (Lab) | 10.0.20.20 | Docker host, monitoring stack |
| 101 | OPNsense | OPNsense 25.1 | Trunk (10,20,30,40) | 10.0.10.1 (Trusted) | Firewall, router, DHCP, DNS |

## Monitoring & Observability Stack

The monitoring stack implements two of the three pillars of observability (metrics and logs), running as Docker containers on the Ubuntu Server VM.

### Metrics Pipeline
| Component | Port | Purpose |
|-----------|------|---------|
| Prometheus | 9090 | Metrics scraping and storage (30-day retention) |
| Node Exporter | 9100 | Linux host metrics (CPU, RAM, disk, network) |
| SNMP Exporter | 9116 | OPNsense network interface metrics via SNMP v2c |
| Grafana | 3000 | Dashboard visualization for metrics and logs |

### Logs Pipeline
| Component | Port | Purpose |
|-----------|------|---------|
| Loki | 3100 | Log aggregation and storage (7-day retention) |
| Grafana Alloy | 12345 | Log collection agent — Docker discovery + OPNsense syslog (UDP 5514) |

### Availability
| Component | Port | Purpose |
|-----------|------|---------|
| Uptime Kuma | 3001 | Service availability monitoring (7 monitors) |

### Grafana Dashboards
- **Node Exporter Full** — imported (ID 1860), Linux host metrics
- **OPNsense Monitoring** — custom, 4 panels: per-interface traffic (in/out), interface count, SNMP scrape time. Interface names relabeled from raw IDs to friendly names (WAN, Trusted, Lab, IoT, Guest) via Prometheus metric_relabel_configs
- **Docker Logs** — custom, 3 panels: live container log stream, filtered error logs, log volume by container

### SNMP Monitoring
- OPNsense Net-SNMP plugin (v2c, community string configured)
- SNMP Exporter uses official default config (61k lines, if_mib module)
- Metrics collected: ifHCInOctets, ifHCOutOctets, ifNumber per interface

### Syslog Integration
- OPNsense forwards firewall logs via UDP syslog to Grafana Alloy (port 5514)
- Alloy parses RFC3164 (BSD) format and ships to Loki
- Enables searching firewall block/pass events in Grafana with LogQL

### Design Decision: Alloy over Promtail
Grafana's original log collector (Promtail) reached end-of-life in March 2026. This project uses Grafana Alloy — the current, actively maintained replacement — for both Docker log collection (via Docker socket discovery) and syslog reception.

## Build Progress

- [x] Proxmox VE installation and configuration
- [x] Ubuntu Server VM with static networking
- [x] Docker and Portainer deployment
- [x] OPNsense VM — firewall, routing, DHCP, DNS
- [x] VLAN segmentation (802.1Q) on switch and OPNsense
- [x] VLAN-aware bridge with trunk/access port config
- [x] Inter-VLAN routing verified
- [x] Firewall rule lockdown (per-VLAN restrictive policies)
- [x] Monitoring stack (Prometheus + Node Exporter + Grafana + Uptime Kuma)
- [x] SNMP monitoring — OPNsense interface metrics via SNMP Exporter
- [x] Log aggregation — Loki + Grafana Alloy (Docker logs + OPNsense syslog)
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
- Per-VLAN security policies with inter-VLAN traffic control
- DHCP and DNS server administration
- Linux server administration (Ubuntu Server 24.04)
- Docker containerization and Portainer management
- Infrastructure monitoring with Prometheus and Grafana
- SNMP network monitoring (v2c, OID mapping, MIB configuration)
- Centralized log aggregation with Grafana Loki
- Telemetry collection with Grafana Alloy (Docker discovery + syslog)
- Syslog integration for firewall log visibility
- LogQL and PromQL query writing
- Managed switch configuration (trunk/access ports, PVID)
- Two-subnet network architecture (WAN/LAN separation)
- Hypervisor-level virtual networking (VLAN-aware bridges, trunk/access VM ports)
- Systematic troubleshooting and root cause analysis

## About

Built by [Gavin White](https://www.linkedin.com/in/gavin-white-812345315) — aspiring Network Engineer with CompTIA Security+ (CE), currently studying for CCNA.
