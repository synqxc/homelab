# SynQX Homelab POC — Backup and Restore Validation Report

## Summary

Backup and restore operations were successfully tested for the SynQX Homelab POC environment.

The initial backup attempts for the BIND9 DNS containers failed due to an incorrect PBS namespace configuration. The existing PBS storage configuration could not be cleared directly, so it was recreated without a namespace. After that change, backups completed successfully and were validated through restore testing.

## Environment

| Component | Detail |
|---|---|
| Hypervisor | Proxmox VE |
| Backup server | Proxmox Backup Server |
| Backup storage | Cloudflare R2 via PBS S3-compatible datastore |
| Backup target | LXC containers |
| Tested containers | `dns1` (CT 503), `dns2` (CT 504) |
| Backup mode | Snapshot |
| Compression | Zstandard |
| Retention | Keep all for initial validation |

## Backup configuration

Backups were configured from Proxmox VE to a Cloudflare R2-backed Proxmox Backup Server datastore.

```text
Proxmox VE
└── LXC containers
    ├── dns1 — CT 503
    └── dns2 — CT 504
        ↓
Proxmox Backup Server
        ↓
Cloudflare R2 — S3-compatible datastore
```

The backup job used the following settings:

| Setting | Value |
|---|---|
| Backup mode | Snapshot |
| Compression | Zstandard |
| Storage | `pbs1` |
| Notes template | `{{guestname}}` |
| Fleecing | Disabled |
| Retention | `keep-all=1` for initial validation |
| Containers | `dns1`, `dns2` |

## Initial backup failure

### Symptom

The first backup attempts for both DNS containers failed immediately after starting the backup protocol.

Relevant log output:

```text
INFO: creating Proxmox Backup Server archive 'ct/503/2026-10-18T11:28:59Z'
INFO: Starting backup: [root]:ct/503/2026-10-18T11:28:59Z
INFO: Error: namespace not found
ERROR: Backup of VM 503 failed - exit code 255
```

The same error occurred for `dns2`:

```text
INFO: creating Proxmox Backup Server archive 'ct/504/2026-10-18T11:29:00Z'
INFO: Starting backup: [root]:ct/504/2026-10-18T11:29:00Z
INFO: Error: namespace not found
ERROR: Backup of VM 504 failed - exit code 255
```

### Root cause

The Proxmox VE PBS storage configuration had a namespace value of `root`.

Proxmox Backup Server was sending backups to:

```text
--ns root
```

However, PBS does not automatically create a namespace literally named `root`. The implicit root namespace is represented by an empty namespace value.

### Resolution

The namespace field could not be cleared from the existing PBS storage configuration in Proxmox VE.

To resolve this, the PBS storage entry was recreated without a namespace:

1. Remove the existing PBS storage configuration from Proxmox VE.
2. Re-add the PBS storage.
3. Leave the **Namespace** field empty.
4. Reconfigure the datastore, credentials, and backup job as required.
5. Rerun the backup job.

After recreating the storage without a namespace, Proxmox VE no longer attempted to send backups to a non-existent namespace.

## Successful backup

After correcting the namespace configuration, backups for both DNS containers completed successfully.

Validated backups:

| Container | CT ID | Backup result |
|---|---:|---|
| `dns1` | 503 | Successful |
| `dns2` | 504 | Successful |

The backups were stored in the Cloudflare R2-backed PBS datastore.

## Restore validation

A restore test was performed to confirm that the backups were usable and not merely present in the datastore.

### Restore method

The container backup was restored from the Proxmox VE web interface.

Two restore approaches were validated:

1. **Full container restore**
   - Used to recover an entire LXC container from a PBS backup.
   - Can restore to the original CT ID, replacing the existing container.
   - Can also restore to a new CT ID for non-destructive testing.

2. **File-level restore**
   - Used to browse the contents of an LXC backup.
   - Allows individual files or directories to be downloaded, such as BIND configuration files and DNS zone files.

### Full container restore

The restore process was tested by restoring a DNS container backup.

The restore confirmed that:

- The container configuration was recovered.
- The container root filesystem was recovered.
- BIND9 configuration
