# Tailscale Design

Tailscale provides secure remote access to the SynQX services VNet.

It is used as a private overlay network for administrative access. It does not replace the home LAN and is not configured as an exit node.

## Purpose

Tailscale allows trusted devices to reach internal services without exposing them to the internet.

Current access scope:

```text
Tailscale client
└── vpn1 / vpn2 subnet router
    └── svc — 10.0.5.0/26
        ├── dns1 — 10.0.5.3
        └── dns2 — 10.0.5.4
```

## Architecture

Two Tailscale subnet routers provide redundancy.

| Server | FQDN | IP | Role |
|---|---|---:|---|
| `vpn1` | `vpn1.internal.synqx.cloud` | `10.0.5.6` | Tailscale subnet router |
| `vpn2` | `vpn2.internal.synqx.cloud` | `10.0.5.7` | Tailscale subnet router |

Both routers advertise the same route:

```text
10.0.5.0/26
```

This allows either router to provide access to the services VNet. If one router is unavailable, the other can continue serving the route.

## Network placement

Both Tailscale containers are attached to the `svc` VNet.

```text
svc VNet — 10.0.5.0/26
├── dns1 — 10.0.5.3
├── dns2 — 10.0.5.4
├── vpn1 — 10.0.5.6
└── vpn2 — 10.0.5.7
```

The subnet routers are not exposed to the internet. They connect to Tailscale outbound and advertise the internal `svc` subnet to approved Tailscale clients.

## Container configuration

Tailscale runs in unprivileged LXC containers.

The containers require access to `/dev/net/tun` to create the Tailscale network interface.

In Proxmox:

1. Create an unprivileged LXC container.
2. Attach it to the `svc` VNet.
3. Assign a static IP address.
4. Add a device passthrough for `/dev/net/tun`.
5. Restart the container.
6. Install and configure Tailscale.

The container configuration is intentionally kept minimal:

- No privileged container mode
- No direct inbound internet exposure
- No unnecessary packages
- Static internal IP address
- Access to `/dev/net/tun` only

## Installation

Install Tailscale on each subnet router:

```bash
curl -fsSL [https://tailscale.com/install.sh](https://tailscale.com/install.sh) | sh
```

Enable and start the service:

```bash
sudo systemctl enable --now tailscaled
```

Authenticate the node:

```bash
sudo tailscale up
```

## Subnet router configuration

On each subnet router, advertise the services VNet:

```bash
sudo tailscale up \
  --advertise-routes=10.0.5.0/26
```

The route must then be approved in the Tailscale admin console.

To verify the advertised routes:

```bash
tailscale status
```

To verify that IP forwarding is enabled:

```bash
sysctl net.ipv4.ip_forward
```

If required, enable IPv4 forwarding:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

## Access policy

Access is controlled using Tailscale ACLs.

The intended policy is:

- Only approved user devices may access the `v5` VNet
- Access is limited to required services and ports
- The subnet routers cannot be used as exit nodes
- Internal services remain unreachable from the public internet

Example ACL intent:

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["group:admin"],
      "dst": ["10.0.5.0/26:*"]
    }
  ]
}
```

This should be narrowed over time so that devices only receive access to the specific services they need.

## DNS access

The Tailscale subnet routers provide access to the internal BIND9 servers:

```text
dns1.internal.synqx.cloud — 10.0.5.3
dns2.internal.synqx.cloud — 10.0.5.4
```

Trusted Tailscale devices can use these servers for internal name resolution when connected to the tailnet.

## Verification

From an approved Tailscale client, verify connectivity to the services VNet:

```bash
ping 10.0.5.3
ping 10.0.5.4
```

Verify DNS resolution:

```bash
dig @10.0.5.3 dns1.internal.synqx.cloud
dig @10.0.5.4 dns1.internal.synqx.cloud
```

Check that both subnet routers are online:

```bash
tailscale status
```

## Operational notes

- Both subnet routers should remain online where possible.
- Route approval is required after a new subnet router is added.
- Tailscale ACLs are the primary access-control mechanism.
- The subnet routers should not be used as exit nodes.
- No Tailscale auth keys or sensitive configuration should be committed to this repository.
