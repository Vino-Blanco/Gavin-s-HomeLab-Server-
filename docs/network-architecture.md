# Network Architecture

This document describes the complete network design, including the two-subnet layout, VLAN segmentation, firewall policies, and monitoring data flows.

---

## Two-Subnet Design

The network is split into two distinct subnets:

- **WAN (192.168.0.0/24):** The Cox router side. Proxmox management (192.168.0.50), OPNsense WAN interface (192.168.0.135 via DHCP), and switch management (192.168.0.239) live here.
- **LAN (10.0.x.0/24):** Behind OPNsense. All lab devices are segmented into VLANs on the 10.0.x.0/24 address space.

This separation exists because the Cox router and OPNsense WAN are on the same physical network. Putting the LAN on the same subnet caused routing conflicts in Proxmox.

---

## Physical Topology

```
Internet
    |
Cox Router (192.168.0.1)
    |
eno1 → vmbr0 — Proxmox Node (192.168.0.50)
    |
OPNsense WAN — vtnet0/net0 (192.168.0.135 via DHCP)

OPNsense LAN — vtnet1/net1 (trunks=10;20;30;40)
    → vmbr1 (VLAN aware, PVID 1)
    → enp2s0f0 (I350-AM2 NIC)
    |
JGS516PE Switch (802.1Q enabled, mgmt 192.168.0.239 on VLAN 1)
    ├── Port 1 — Trunk (T on VLANs 10,20,30,40 / U on VLAN 1)
    ├── Port 2 — Main PC (VLAN 10 Trusted — 10.0.10.10)
    └── Port 3+ — future devices

Ubuntu Server VM — net0 (tag=20) → vmbr1 → VLAN 20 (10.0.20.20)
    ├── Docker: prometheus (9090)
    ├── Docker: node-exporter (9100)
    ├── Docker: snmp-exporter (9116)
    ├── Docker: grafana (3000)
    ├── Docker: uptime-kuma (3001)
    ├── Docker: loki (3100)
    ├── Docker: alloy (12345, 5514/udp)
    └── Docker: portainer (9000)
```

---

## VLAN Layout

| VLAN ID | Name | Subnet | Gateway | Purpose |
|---------|------|--------|---------|----------|
| 1 | Switch Mgmt | 192.168.0.0/24 | N/A | Untagged switch management (firmware limitation) |
| 10 | Trusted | 10.0.10.0/24 | 10.0.10.1 | Personal devices (full access) |
| 20 | Lab | 10.0.20.0/24 | 10.0.20.1 | Lab VMs and monitoring stack |
| 25 | Production | 10.0.25.0/24 | 10.0.25.1 | Future production services |
| 30 | IoT | 10.0.30.0/24 | 10.0.30.1 | IoT devices (internet only) |
| 40 | Guest | 10.0.40.0/24 | 10.0.40.1 | Guest devices (fully isolated) |
| 50 | Security Cams | 10.0.50.0/24 | 10.0.50.1 | Future camera network |

---

## OPNsense Interface Assignments

| ifName | OPNsense Name | Role | VLAN Tag |
|--------|--------------|------|----------|
| vtnet0 | WAN | WAN uplink to Cox router | — |
| vtnet1 | LAN | LAN trunk to switch (VLAN parent) | — |
| vtnet2 | OPT1 | Unused | — |
| vlan01 | Trusted | VLAN 10 sub-interface | 10 |
| vlan02 | Lab | VLAN 20 sub-interface | 20 |
| vlan03 | IoT | VLAN 30 sub-interface | 30 |
| vlan04 | Guest | VLAN 40 sub-interface | 40 |

---

## Firewall Rules

### Trusted (VLAN 10)
1. Pass — Trusted net → any — Allow all Trusted traffic

### Lab (VLAN 20)
1. Pass — Lab net → OPNsense GUI (TCP 443, This Firewall)
2. Pass — Lab net → This Firewall (ICMP)
3. Pass — Lab net → Trusted net
4. Block — Lab net → IoT net
5. Block — Lab net → Guest net
6. Pass — Lab net → any (internet access)

### IoT (VLAN 30)
1. Block — IoT net → Trusted net
2. Block — IoT net → Lab net
3. Block — IoT net → Guest net
4. Block — IoT net → This Firewall
5. Pass — IoT net → any (internet access)

### Guest (VLAN 40)
1. Block — Guest net → Trusted net
2. Block — Guest net → Lab net
3. Block — Guest net → IoT net
4. Block — Guest net → This Firewall
5. Pass — Guest net → any (internet access)

---

## Monitoring Data Flows

### Metrics Flow
```
Node Exporter (host metrics) ──┐
                               ├──→ Prometheus (scrapes every 15s) ──→ Grafana
SNMP Exporter ←── SNMP v2c ──→ OPNsense
```

Prometheus scrapes three jobs: itself, Node Exporter, and OPNsense via SNMP Exporter. SNMP queries use the Lab gateway IP (10.0.20.1) since the Ubuntu Server is on VLAN 20.

### Logs Flow
```
Docker containers ──→ Alloy (Docker socket discovery) ──┐
                                                        ├──→ Loki ──→ Grafana
OPNsense ──→ UDP syslog (5514) ──→ Alloy (RFC3164) ─────┘
```

Grafana Alloy collects logs from two sources: Docker containers (auto-discovered via the Docker socket) and OPNsense (via UDP syslog on port 5514, parsed as RFC3164/BSD format). Both streams are shipped to Loki and queryable in Grafana using LogQL.

### Availability Monitoring
```
Uptime Kuma ──→ HTTP checks every 60s
    ├── Grafana (10.0.20.20:3000)
    ├── Node Exporter (10.0.20.20:9100)
    ├── Prometheus (10.0.20.20:9090)
    ├── OPNsense (10.0.20.1:80)
    ├── Proxmox (192.168.0.50:8006)
    ├── Loki (10.0.20.20:3100)
    └── Alloy (10.0.20.20:12345)
```
