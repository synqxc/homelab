# Design

## Domain model

| Zone                   | Hosted by               | Purpose                       |
| :--------------------- | :---------------------- | :---------------------------- |
| `synqx.cloud`          | Cloudflare              | Public DNS only               |
| `internal.synqx.cloud` | BIND9 primary/secondary | Private homelab services      |
| `5.0.10.in-addr.arpa`  | BIND9 primary/secondary | Reverse DNS for services VNet |

Public DNS is kept minimal in Cloudflare. Internal services use
`internal.synqx.cloud`.

Example:

```text
vpn1.internal.synqx.cloud → 10.0.5.6
```

## DNS servers

| Server | IP         | Role            | OS        |
| :----- | :--------- | :-------------- | :-------- |
| `dns1` | `10.0.5.3` | BIND9 primary   | AlmaLinux |
| `dns2` | `10.0.5.4` | BIND9 secondary | AlmaLinux |

Both servers:

- Serve recursive DNS to trusted networks

- Forward unknown external names to Cloudflare and OpenDNS

- Use `forward first`

- Use TSIG-protected zone transfers

- Enable query logging

- Restrict recursion to trusted networks

## Trusted networks

```text
acl "trusted-networks" {
    localhost;
    10.0.5.0/26;
};
```

Additional VNets will be added to this ACL as they are created.

## Forwarders

```text
forwarders {
    1.1.1.1;
    1.0.0.1;
    208.67.222.222;
    208.67.222.220;
};
forward first;
```

`forward first` means BIND tries the forwarders first, then falls back to
direct recursive resolution if they are unavailable.

## Zone transfer

Zone transfers from `dns1` to `dns2` are secured using TSIG.

| Item      | Value                          |
| :-------- | :----------------------------- |
| Key file  | `/etc/named/`example-key`.key` |
| Key name  | example-key                    |
| Algorithm | HMAC-SHA256                    |
| Primary   | `dns1` — `10.0.5.3`            |
| Secondary | `dns2` — `10.0.5.4`            |

The primary permits transfers only using the TSIG key:

```text
allow-transfer {
    key "example-key";
};
```

The primary also sends signed NOTIFY messages to the secondary:

```text
also-notify {
    10.0.5.4 key "example-key";
};
```

## Zone files

### dns1 — primary

```text
/var/named/zones/internal.synqx.cloud.zone
/var/named/zones/5.0.10.in-addr.arpa.zone
```

These files are manually maintained on `dns1`.

### dns2 — secondary

```text
/var/named/zones/internal.synqx.cloud.zone
/var/named/zones/5.0.10.in-addr.arpa.zone
```

These files are automatically created and maintained by BIND after zone
transfers from `dns1`. They must not be edited manually.

## Current service records

| Service | FQDN                        | IP         |
| :------ | :-------------------------- | :--------- |
| DNS 1   | `dns1.internal.synqx.cloud` | `10.0.5.3` |
| DNS 2   | `dns2.internal.synqx.cloud` | `10.0.5.4` |
| VPN 1   | `vpn1.internal.synqx.cloud` | `10.0.5.6` |
| VPN 2   | `vpn2.internal.synqx.cloud` | `10.0.5.7` |

## Validation tests

```bash
# Internal resolution
dig @10.0.5.3 dns2.internal.synqx.cloud +short
dig @10.0.5.4 dns1.internal.synqx.cloud +short

# External recursion
dig @10.0.5.3 cloudflare.com +short
dig @10.0.5.4 cloudflare.com +short

# Reverse DNS
dig @10.0.5.3 -x 10.0.5.6 +short
dig @10.0.5.4 -x 10.0.5.7 +short

# Zone serial comparison
dig @10.0.5.3 internal.synqx.cloud SOA +short
dig @10.0.5.4 internal.synqx.cloud SOA +short

# Unauthorised transfer should fail
dig @10.0.5.3 internal.synqx.cloud AXFR
```

