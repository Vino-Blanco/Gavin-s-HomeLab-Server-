# My HomeLab Server

A production-style homelab built on Proxmox VE, featuring network segmentation with VLANs, a virtualized OPNsense firewall/router, a full observability stack (metrics + logs), self-hosted services, and a modded Minecraft game server -- all accessible locally via clean URLs and publicly via a custom domain. Designed to mirror real-world enterprise infrastructure.

**Live:** [gavinwhite.dev](https://gavinwhite.dev) | **Status:** [status.gavinwhite.dev](https://status.gavinwhite.dev)

## Project Overview

This homelab serves as a hands-on learning environment for network engineering, systems administration, and security operations. Every component is documented from scratch, and the network is designed with the same principles used in production data center environments.

## Key Accomplishments

- Designed and implemented a segmented network with 4 VLANs, each with its own subnet, DHCP scope, and firewall policy -- isolating trusted, lab, IoT, and guest traffic
- Built and enforced per-VLAN firewall rules on OPNsense, restricting inter-VLAN access based on trust level while maintaining internet connectivity for all segments
- Deployed a full metrics pipeline (Prometheus, Node Exporter, SNMP Exporter, Grafana) to monitor host performance and network interface traffic across all OPNsense interfaces in real time
- Integrated SNMP v2c monitoring for OPNsense and relabeled raw interface identifiers to human-readable names (WAN, Trusted, Lab, IoT, Guest) using Prometheus metric_relabel_configs
- Built centralized log aggregation with Loki and Grafana Alloy, collecting Docker container logs via socket discovery and OPNsense firewall logs via UDP syslog -- searchable in Grafana with LogQL
- Deployed Pi-hole for network-wide DNS ad blocking across all VLANs, with DNS forwarding through OPNsense Unbound (DNSSEC enabled)
- Built a custom portfolio website served via Nginx and exposed it publicly through a Cloudflare Tunnel at gavinwhite.dev -- no port forwarding, no exposed home IP
- Configured Nginx Proxy Manager with 10 local proxy hosts, enabling clean internal URLs (*.homelab.local) backed by Pi-hole local DNS records
- Set up Tailscale VPN mesh for secure remote access to the entire homelab from any device, anywhere
- Deployed a modded Minecraft server (Monifactory, 191 mods) with tiered storage architecture -- NVMe for OS/Docker, HDD for game data -- accessible to friends via Tailscale
- Chose Grafana Alloy over Promtail after identifying that Promtail had reached end-of-life, demonstrating awareness of tool lifecycle and the ability to evaluate alternatives

## Troubleshooting Highlights

Real problems I encountered and solved during this build:

**Subnet conflict broke all routing.** Both Proxmox bridges (vmbr0 and vmbr1) were on the same 192.168.0.0/24 subnet, causing the host to route traffic to the wrong bridge. Resolved by redesigning the network into two subnets -- WAN on 192.168.0.0/24 and LAN on 10.0.x.0/24.

**VLAN-aware bridge silently dropped traffic.** Enabling VLAN-aware mode on Proxmox's vmbr1 caused OPNsense to stop receiving tagged traffic because no trunk configuration existed on the VM NIC. Fixed by adding explicit trunk definitions for VLANs 10, 20, 30, and 40.

**Locked myself out of the switch.** While configuring VLANs on the JGS516PE, I accidentally removed the management port from VLAN 1, cutting off all access. Required a physical factory reset to recover. Learned to always verify management access before applying VLAN changes.

**SNMP Exporter returned only scrape-health metrics.** A hand-written 14-line SNMP config started and scraped successfully but produced no real interface data. The fix was extracting the full 61,000-line official default config from the container image, which contained the proper OID-to-metric mappings.

**Syslog logs arrived but were silently discarded.** OPNsense was sending syslog to Alloy, but no logs appeared in Loki. Alloy's logs showed repeated parse errors -- it was expecting RFC5424 format while OPNsense sends BSD/RFC3164. Adding `syslog_format = "rfc3164"` to the Alloy config resolved the issue immediately.

**Pi-hole container failed to build gravity database.** Docker's internal DNS resolver couldn't reach the internet during Pi-hole's first startup, preventing blocklist downloads. Fixed by adding explicit bootstrap DNS servers (1.1.1.1, 8.8.8.8) via the `dns:` directive in docker-compose, separate from Pi-hole's upstream DNS config.

**Root filesystem hit 100% during Minecraft deployment.** Ubuntu Server's installer only allocated 15GB of a 32GB virtual disk by default. Freed space with `docker system prune`, then expanded the logical volume to use the full disk with `lvextend` and `resize2fs`. Added a separate 200GB HDD-backed virtual disk for game data to implement tiered storage.

## Architecture

```
                        Internet
                           |
                    Cox Router (192.168.0.1)
                           |
                 eno1 -> vmbr0 (WAN Bridge)
                           |
              +-------------+-------------+
              |                           |
     Proxmox Node                  OPNsense VM
    (192.168.0.50)              WAN: 192.168.0.135
                                LAN: VLAN Trunk
                                         |
                               vtnet1 -> vmbr1
                             (VLAN-aware bridge)
                                         |
                              enp2s0f0 (I350-AM2)
                                         |
                            JGS516PE Managed Switch
                           +------+------+
                      Port 1          Port 2
                     (Trunk)        (Access)
                   T: 10,20,30,40   PVID: 10
                   U: VLAN 1
                                         |
                                     Main PC
                                  (10.0.10.10)

              +-------- Ubuntu Server VM (16GB RAM) -------+
              |   tag=20 on vmbr1                          |
              |   10.0.20.20 (VLAN 20 -- Lab)              |
              |   scsi0: 32GB NVMe (OS/Docker)             |
              |   scsi1: 200GB HDD  (game data)            |
              |                                            |
              |   Metrics:                                 |
              |   |-- Prometheus (9090)                    |
              |   |-- Node Exporter (9100)                 |
              |   |-- SNMP Exporter (9116)                 |
              |   +-- Grafana (3000)                       |
              |                                            |
              |   Logs:                                    |
              |   |-- Loki (3100)                          |
              |   +-- Grafana Alloy (12345, 5514/udp)      |
              |                                            |
              |   Availability:                            |
              |   +-- Uptime Kuma (3001)                   |
              |                                            |
              |   Services:                                |
              |   |-- Pi-hole (53, 8080)                   |
              |   |-- Homepage (3002)                      |
              |   |-- Portfolio (8090)                     |
              |   |-- Nginx Proxy Manager (80, 443, 81)   |
              |   |-- Cloudflare Tunnel                    |
              |   +-- Minecraft (25565)                    |
              |                                            |
              |   Management:                              |
              |   +-- Portainer (9000)                     |
              +--------------------------------------------+

              Remote Access: Tailscale VPN (100.89.129.32)
              Public Access: gavinwhite.dev via Cloudflare Tunnel
```

## VLAN Segmentation

| VLAN ID | Name | Subnet | Gateway | Purpose | Status |
|---------|------|--------|---------|---------|--------|
| 1 | Switch Mgmt | 192.168.0.0/24 | N/A | Switch management (untagged) | Working |
| 10 | Trusted | 10.0.10.0/24 | 10.0.10.1 | Personal devices | Working |
| 20 | Lab | 10.0.20.0/24 | 10.0.20.1 | Lab VMs and services | Working |
| 30 | IoT | 10.0.30.0/24 | 10.0.30.1 | Smart home devices | Working |
| 40 | Guest | 10.0.40.0/24 | 10.0.40.1 | Guest/untrusted devices | Working |
| 50 | Security Cams | 10.0.50.0/24 | 10.0.50.1 | Future camera network | Planned |

## Firewall Policy

| VLAN | Internet | Trusted | Lab | IoT | Guest | Firewall Mgmt |
|------|----------|---------|-----|-----|-------|---------------|
| Trusted (10) | Allow | -- | Allow | Allow | Allow | Allow |
| Lab (20) | Allow | Allow | -- | Block | Block | Allow |
| IoT (30) | Allow | Block | Block | -- | Block | Block |
| Guest (40) | Allow | Block | Block | Block | -- | Block |

## DNS Architecture

All network devices use Pi-hole as their DNS server. Pi-hole blocks ads and trackers at the DNS level, then forwards legitimate queries to OPNsense's Unbound resolver (which validates with DNSSEC) before reaching the internet.

```
Client Device --> Pi-hole (10.0.20.20:53) --> OPNsense Unbound (10.0.20.1, DNSSEC) --> Internet
```

Pi-hole also serves local DNS records for *.homelab.local domains, which point to Nginx Proxy Manager for clean internal URLs.

## Hardware

| Component | Details |
|-----------|---------|
| Server | HP EliteDesk G4 600 (Intel i5-8500, 32GB RAM) |
| Hypervisor | Proxmox VE 9.1.7 |
| NIC | 10Gtek Intel I350-AM2 dual-port PCIe |
| Storage | Samsung 970 Evo NVMe (256GB, OS + VMs) + 1TB WD HDD (VM disks + game data) |
| Switch | Netgear JGS516PE 16-port managed PoE |
| UPS | CyberPower CP1500AVR |

## Virtual Machines

| VM ID | Name | OS | VLAN | IP | RAM | Storage | Purpose |
|-------|------|----|------|----|----|---------|--------|
| 100 | ubuntu-server | Ubuntu Server 24.04 LTS | 20 (Lab) | 10.0.20.20 | 16 GB | 32GB NVMe + 200GB HDD | Docker host, monitoring + services + Minecraft |
| 101 | OPNsense | OPNsense 25.1 | Trunk (10,20,30,40) | 10.0.10.1 (Trusted) | 2 GB | 16GB NVMe | Firewall, router, DHCP, DNS |

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
| Grafana Alloy | 12345 | Log collection agent -- Docker discovery + OPNsense syslog (UDP 5514) |

### Availability
| Component | Port | Purpose |
|-----------|------|---------|
| Uptime Kuma | 3001 | Service availability monitoring with public status page |

### Grafana Dashboards
- **Node Exporter Full** -- imported (ID 1860), Linux host metrics
- **OPNsense Monitoring** -- custom, 4 panels: per-interface traffic (in/out), interface count, SNMP scrape time. Interface names relabeled from raw IDs to friendly names (WAN, Trusted, Lab, IoT, Guest) via Prometheus metric_relabel_configs
- **Docker Logs** -- custom, 3 panels: live container log stream, filtered error logs, log volume by container

## Self-Hosted Services

| Service | Port | Purpose |
|---------|------|---------|
| Pi-hole | 53, 8080 | Network-wide DNS ad blocking for all VLANs |
| Homepage | 3002 | Internal services dashboard with live status |
| Portfolio | 8090 | Custom HTML/CSS personal website (public at gavinwhite.dev) |
| Nginx Proxy Manager | 80, 443, 81 | Reverse proxy -- 10 local proxy hosts for *.homelab.local |
| Cloudflare Tunnel | -- | Exposes portfolio and status page to the internet via gavinwhite.dev |
| Minecraft | 25565 | Monifactory modded server (191 mods, Forge 1.20.1) -- Tailscale access for friends |
| Portainer | 9000 | Docker container management UI |

## Storage Architecture

| Disk | Proxmox Storage | Size | VM Mount | Purpose |
|------|----------------|------|----------|---------|
| scsi0 | local-lvm (NVMe) | 32GB | / (30GB LV) | OS, Docker engine, container configs |
| scsi1 | hdd-storage (1TB HDD) | 200GB | /mnt/gamedata | Minecraft world data |

NVMe handles OS and Docker for fast random I/O. The HDD handles game server data, which is primarily sequential chunk I/O and doesn't need SSD performance.

## Access Methods

| Method | What it provides | Example |
|--------|-----------------|--------|
| Local IP:port | Direct access from home network | http://10.0.20.20:3000 |
| Local DNS (*.homelab.local) | Clean URLs via Pi-hole + NPM | http://grafana.homelab.local |
| Tailscale | Remote access from anywhere | http://100.89.129.32:3000 |
| Cloudflare Tunnel | Public internet access | https://gavinwhite.dev |

## Build Progress

- [x] Proxmox VE installation and configuration
- [x] Ubuntu Server VM with static networking
- [x] Docker and Portainer deployment
- [x] OPNsense VM -- firewall, routing, DHCP, DNS
- [x] VLAN segmentation (802.1Q) on switch and OPNsense
- [x] VLAN-aware bridge with trunk/access port config
- [x] Inter-VLAN routing verified
- [x] Firewall rule lockdown (per-VLAN restrictive policies)
- [x] Monitoring stack (Prometheus + Node Exporter + Grafana + Uptime Kuma)
- [x] SNMP monitoring -- OPNsense interface metrics via SNMP Exporter
- [x] Log aggregation -- Loki + Grafana Alloy (Docker logs + OPNsense syslog)
- [x] Pi-hole DNS ad blocking (network-wide, all VLANs)
- [x] Homepage internal dashboard
- [x] Custom portfolio website (gavinwhite.dev)
- [x] Nginx Proxy Manager with 10 local proxy hosts
- [x] Cloudflare Tunnel for public access
- [x] Tailscale VPN for remote access
- [x] Public status page (status.gavinwhite.dev)
- [x] Minecraft Monifactory game server (HDD-backed, Tailscale access)
- [ ] Active Directory lab rebuild
- [ ] Jellyfin media server
- [ ] Ansible infrastructure automation
- [ ] Wazuh SIEM

## Documentation

| Document | Description |
|----------|-------------|
| [Build Phases](docs/build-phases.md) | Detailed walkthrough of each build phase |
| [Network Architecture](docs/network-architecture.md) | Complete network topology and design decisions |
| [Lessons Learned](docs/lessons-learned.md) | Troubleshooting experiences and key takeaways |
| [Screenshots](screenshots/) | 31 annotated screenshots of every major component |

## Skills Demonstrated

- Bare-metal hypervisor deployment (Proxmox VE)
- Virtual machine provisioning and management
- Network segmentation with 802.1Q VLANs
- Firewall configuration and stateful rule management (OPNsense/pf)
- Per-VLAN security policies with inter-VLAN traffic control
- DHCP and DNS server administration
- DNS sinkhole deployment (Pi-hole) with upstream DNSSEC validation
- Linux server administration (Ubuntu Server 24.04)
- Docker containerization and Portainer management
- Infrastructure monitoring with Prometheus and Grafana
- SNMP network monitoring (v2c, OID mapping, MIB configuration)
- Centralized log aggregation with Grafana Loki
- Telemetry collection with Grafana Alloy (Docker discovery + syslog)
- Syslog integration for firewall log visibility
- LogQL and PromQL query writing
- Reverse proxy configuration (Nginx Proxy Manager)
- Secure remote access (Tailscale VPN mesh)
- Public hosting via Cloudflare Tunnel (zero-trust, no port forwarding)
- Custom web development (HTML/CSS portfolio site)
- Domain management and DNS (Cloudflare)
- Managed switch configuration (trunk/access ports, PVID)
- Two-subnet network architecture (WAN/LAN separation)
- Hypervisor-level virtual networking (VLAN-aware bridges, trunk/access VM ports)
- Game server hosting and modpack deployment (Minecraft Forge)
- Tiered storage architecture (NVMe for OS/Docker, HDD for game data)
- LVM disk management and filesystem expansion
- Systematic troubleshooting and root cause analysis

## About

Built by [Gavin White](https://www.linkedin.com/in/gavin-white-812345315) -- aspiring Network Engineer with CompTIA Security+ (CE), currently studying for CCNA.

Portfolio: [gavinwhite.dev](https://gavinwhite.dev) | Status: [status.gavinwhite.dev](https://status.gavinwhite.dev)
