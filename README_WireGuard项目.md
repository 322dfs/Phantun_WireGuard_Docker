<div align="center">

# 🌐 Phantun WireGuard Docker

**Enterprise-Grade Cross-Border VPN Solution**

[![WireGuard](https://img.shields.io/badge/WireGuard-Latest-brightgreen?logo=wireguard&logoColor=white)](https://www.wireguard.com/)
[![Docker](https://img.shields.io/badge/Docker-Containerized-blue?logo=docker&logoColor=white)](https://www.docker.com/)
[![Phantun](https://img.shields.io/badge/Phantun-UDP%20over%20TCP-orange)](https://github.com/dndx/phantun)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

</div>

---

## Overview

A production-ready **cross-border networking solution** that solves UDP blocking, packet loss, and connection instability issues in international VPN scenarios. Built with **WireGuard + Phantun + Docker** for enterprise deployment.

### 🎯 Problem Solved

| Challenge | Solution |
|:----------|:---------|
| UDP blocked by ISP | Phantun encapsulates UDP → TCP 443 |
| Connection success rate ≤50% | Achieved **100%** connectivity |
| High latency (200ms+) | Optimized to **45-60ms** |
| Frequent disconnections | Container auto-restart, 7×24h stable |

---

## Architecture

**Core Concept: Dual-container shared network namespace + TCP encapsulation**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        North America Client                              │
│                                                                          │
│   WireGuard ──(UDP 51820)──► Phantun Client ──(TCP 443)──► Internet    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ TCP 443
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Internet (Cross-border)                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Shanghai HQ (Docker Host)                           │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              Shared Network Namespace (127.0.0.1)                │   │
│   │                                                                  │   │
│   │  ┌────────────────┐              ┌────────────────┐             │   │
│   │  │ Phantun Server │   (UDP)      │ wg-access-server│             │   │
│   │  │   Port 443     │ ──────────► │   Port 51820    │             │   │
│   │  │  (TCP → UDP)   │  127.0.0.1   │  (Decrypt)      │             │   │
│   │  └────────────────┘              └───────┬────────┘             │   │
│   └──────────────────────────────────────────┼──────────────────────┘   │
│                                              │                           │
│                                    ┌─────────┴─────────┐                │
│                                    │  Web Management   │                │
│                                    │    Port 8888      │                │
│                                    └───────────────────┘                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                         ┌──────────────────┐
                         │  Internal LAN    │
                         │ 192.168.0.0/24   │
                         └──────────────────┘
```

---

## Tech Stack

| Component | Technology | Purpose |
|:----------|:-----------|:--------|
| VPN Tunnel | WireGuard | High-speed encrypted tunnel |
| Protocol Conversion | Phantun | UDP → TCP to bypass ISP blocking |
| Containerization | Docker | Production deployment, auto-restart |
| Management | wg-access-server | Web UI, multi-client auth |
| Security | FortiGate | Firewall, IP whitelist |

---

## Port Configuration

| Port | Protocol | Purpose | Exposure |
|:-----|:---------|:--------|:---------|
| **443** | TCP | Public tunnel entry (disguised as HTTPS) | Global |
| **8888** | TCP | Web management interface | Internal only |
| **51820** | UDP | WireGuard internal communication | Localhost only (127.0.0.1) |

---

## Traffic Flow

```
North America Office PC
    │
    ▼ (WireGuard generates UDP 51820 traffic)
Phantun Client (UDP → TCP encapsulation)
    │
    ▼ (TCP 443 packet, destination: 222.71.32.173:443)
═══════════════════════════════════════
           Internet (Cross-border)
═══════════════════════════════════════
    │
    ▼
FortiGate Firewall (allow TCP 443)
    │
    ▼
Docker Host (222.71.32.173)
    │
    ▼
Container B (phantun-server) listening on 0.0.0.0:443
    │  Decapsulates TCP → UDP, target: 127.0.0.1:51820
    ▼ (Shared network namespace, localhost)
Container A (wg-access-server) listening on 127.0.0.1:51820
    │  Decrypts WireGuard traffic
    ▼
Internal LAN Resources
```

---

## Solution Evolution (6 Iterations)

| Version | Approach | Protocol | Success Rate | Issue |
|:--------|:---------|:---------|:-------------|:------|
| v1 | WireGuard + Mimic (MT3000) | UDP | ≤50% | Poor UDP penetration, dynamic IP |
| v2 | WireGuard + UDPTunnel | UDP | ≤70% | Still UDP limited |
| v3 | WireGuard + Phantun (MT3000) | TCP | 100% | Router performance, not enterprise-ready |
| v4 | Single wg-access-server | UDP | ≤60% | Pure UDP blocked by ISP |
| v5 | Single container (port 52000) | UDP | ≤75% | Cannot solve UDP本质问题 |
| **v6** | **Dual Container (WG + Phantun)** | **TCP** | **100%** | ✅ Final solution |

**Key Insight:** Shift from "optimizing UDP" to "TCP encapsulation" using Phantun.

---

## Quick Start

### Server Side (Shanghai HQ)

```bash
# 1. Start WireGuard container
docker run -d \
  --name wg-access-server \
  --restart always \
  --cap-add NET_ADMIN \
  --cap-add SYS_MODULE \
  -p 8888:8000/tcp \
  -v /etc/wireguard:/etc/wireguard \
  -e WG_ADMIN_PASSWORD=your_password \
  -e WG_LISTEN_PORT=51820 \
  -e WG_VPN_CIDR=10.8.0.0/24 \
  weejewel/wg-access-server

# 2. Start Phantun container (shared network namespace)
docker run -d \
  --name phantun-server \
  --restart always \
  --cap-add NET_ADMIN \
  --network container:wg-access-server \
  ghcr.io/dndx/phantun:latest \
  phantun-server -l 0.0.0.0:443 -r 127.0.0.1:51820 -k "your_key" --mtu 1400
```

### Client Side (North America)

```bash
# 1. Run Phantun client
phantun-client -l 127.0.0.1:51820 -r YOUR_SERVER_IP:443 -k "your_key"

# 2. Configure WireGuard
[Interface]
PrivateKey = <your-private-key>
Address = 10.8.0.2/24
MTU = 1400

[Peer]
PublicKey = <server-public-key>
Endpoint = 127.0.0.1:51820
AllowedIPs = 192.168.0.0/24, 10.8.0.0/24
PersistentKeepalive = 20
```

---

## Performance Metrics

| Metric | Value |
|:-------|:------|
| **Connectivity** | 100% |
| **Latency** | 45-60ms |
| **Packet Loss** | ≤0.05% |
| **Uptime** | 7×24h continuous |
| **Bandwidth** | Supports test data, web tools, file transfer |

---

## Project Outcomes

### Business Impact
- ✅ North America team can access internal network **7×24h** stable
- ✅ Productivity improved by **30%**
- ✅ Optical module test visualization web tool: **100%** cross-network usage
- ✅ Annual savings: **~80,000 CNY** vs commercial VPN solutions

### Technical Deliverables
- ✅ Reusable enterprise cross-border networking template
- ✅ 3 documents archived: Deployment Manual, Troubleshooting Guide, Extension Guide
- ✅ Validated key techniques: TCP encapsulation, dual-container shared network stack

### Security & Compliance
- ✅ Dual encryption: Phantun transport + WireGuard tunnel
- ✅ Firewall strict source/port restrictions
- ✅ Passed internal security compliance audit

---

## Key Learnings

### Technology Selection
- Use lightweight carriers (MT3000) for rapid hypothesis validation, then invest in containerization

### Cross-border Networking Best Practices
- Avoid relying on UDP; prioritize TCP encapsulation or mature VPN protocols
- Expose common ports (443, 8888) to reduce blocking probability
- Containerized deployment preferred over hardware deployment

### Project Management
- Small iterative steps
- Strengthen cross-team collaboration
- Emphasize documentation output

---

## Future Roadmap

- [ ] Multi-path redundancy (backup public line + DNS round-robin)
- [ ] High availability cluster (multi-node + Keepalived VIP)
- [ ] Enhanced monitoring (Prometheus + Grafana, ELK logging)
- [ ] RBAC permission control, operation audit, automatic key rotation

---

## Use Cases

- 🏢 **Enterprise HQ ↔ Overseas Branch** connectivity
- 🔬 **Cross-border R&D** and testing
- 🚫 **UDP-restricted environments** (ISP blocking)
- ⚡ **Low-latency requirements** (< 100ms)
- 🏭 **SMB production environments**

---

## Team

| Role | Responsibility |
|:-----|:---------------|
| Network Architect | Architecture design, container deployment, port planning, network debugging, connectivity verification |

---

## License

MIT License

---

<div align="center">

**Shanghai Beipu Semiconductor Technology Co., Ltd.**

*Secure · Fast · Reliable Cross-Border Networking*

</div>
