# SynQX Homelab POC

A documented, internal-first Proxmox private-cloud proof of concept built to explore infrastructure, networking, storage, DNS, and secure remote access.

This repository documents the design, build, and operation of a single-node Proxmox platform. It is private by default: workloads run in segmented SDN networks and are not exposed directly to the internet.

## Objectives

- Build a private cloud from the ground up
- Learn and document Proxmox VE, ZFS, SDN, DNS, and VPN
- Provide a stable platform for nested labs and infrastructure experimentation
- Practise production-style design decisions, documentation, and operational runbooks
- Develop skills relevant to hybrid infrastructure, private cloud, and platform engineering

## Platform overview

The platform uses Proxmox VE with local ZFS storage and Proxmox SDN for workload networking.

The Proxmox host uses the existing home LAN for management only. Workloads are isolated within SDN VNets.

```text
Internet
   │
   └── No inbound access to Proxmox, DNS, or internal services

Home LAN — 192.168.0.0/24
└── Proxmox host management
    └── VLAN-aware bridge / SDN underlay
        └── svc — 10.0.5.0/26
```

## Current services

| Service | Status | Purpose |
|---|---|---|
| Proxmox VE | Deployed | Hypervisor and platform foundation |
| ZFS | Deployed | Local VM and container storage |
| Proxmox SDN | Deployed | Segmented workload networking |
| BIND9 | Deployed | Internal and recursive DNS |
| Tailscale | Deployed | Secure remote access to internal networks |

## Network design

The network uses a predictable VLAN and subnet allocation model.

VLANs are allocated in increments of five, starting at VLAN 5.

The subnet is derived deterministically from the VLAN ID:

```text
VLAN n → 10.(n / 256).(n mod 256).0/24
```

### Deployed networks

| VNet | VLAN | Subnet | Purpose | Status |
|---|---:|---|---|---|
| `v5` | 5 | `10.0.5.0/26` | Services | Deployed |

Workload VNets use Proxmox SDN SNAT for outbound access:

```text
VM / LXC → VNet gateway → Proxmox SNAT → home LAN → internet
```

Inter-VNet access is denied by default unless explicitly required.

## DNS

Internal DNS is provided by a redundant BIND9 pair.

| Server | FQDN | IP | Role |
|---|---|---:|---|
| `dns1` | `dns1.internal.synqx.cloud` | `10.0.5.3` | BIND9 primary |
| `dns2` | `dns2.internal.synqx.cloud` | `10.0.5.4` | BIND9 secondary |

DNS design:

- Internal authoritative zone: `internal.synqx.cloud`
- Reverse zone: `5.0.10.in-addr.arpa`
- Public domain: `synqx.cloud`, managed in Cloudflare
- Split-horizon DNS: private records remain internal
- TSIG-protected zone transfers using HMAC-SHA256
- Restricted recursion for trusted networks
- External forwarding to Cloudflare and OpenDNS
- Query logging enabled

Internal names are answered authoritatively from `internal.synqx.cloud`. External names are forwarded to upstream resolvers.

Example:

```text
dns1.internal.synqx.cloud → 10.0.5.3
```

## Tailscale

Tailscale provides secure remote access to the services VNet.

| Server | FQDN | IP | Role |
|---|---|---:|---|
| `vpn1` | `vpn1.internal.synqx.cloud` | `10.0.5.6` | Tailscale subnet router |
| `vpn2` | `vpn2.internal.synqx.cloud` | `10.0.5.7` | Tailscale subnet router |

Both subnet routers advertise the same route for high availability:

```text
10.0.5.0/26   Services
```

These are subnet routers, not exit nodes. They provide access to internal networks only.

## Security model

- No inbound internet access to Proxmox, DNS, or internal services
- Tailscale access is restricted using ACLs
- Internal DNS recursion is restricted to trusted networks
- BIND zone transfers require a TSIG key
