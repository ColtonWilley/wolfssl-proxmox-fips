# wolfSSL FIPS for Proxmox VE — Moving guests from Proxmox VE

Move guests by backup and restore onto a separate, freshly deployed wolfSSL-configured target.
This does not convert the source node or join a stock node to a FIPS cluster.

## 1. Prepare the destination

Complete [Installation](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md) and the selected
storage guide. Confirm that the destination storage is unlocked, has enough capacity, and is
available on the node performing the restore. Use an unused destination guest ID.

The [Backup guide](../_shared/WOLFSSL-PROXMOX-FIPS-BACKUP.md) covers the storage and key settings
used below. Backups from Proxmox Backup Server require the original client encryption key;
a datastore login or a successful snapshot listing does not supply that key.

A passphrase-protected PBS key file must use PBKDF2 rather than scrypt. Before using such a
file on the target, rewrap it on the source or another system that can still read it:

```sh
proxmox-backup-client key change-passphrase <keyfile> --kdf pbkdf2
```

Enter the current passphrase, then a new passphrase of at least 14 bytes. Rewrapping preserves
the encryption key and its fingerprint, so existing backups remain decryptable. Keys used
by Proxmox VE backup jobs must have no passphrase; see the Backup guide for registration
and off-cluster key custody.

## 2. Back up the source

Inventory the source volumes before backing up. Detached VM disks (`unusedN`) are omitted
from ordinary backups; container bind/device mounts are excluded, and additional container
volumes need their backup option enabled. Attach disks that must be backed up or transfer
the excluded data separately. Compare the restored volume inventory with the source before
retiring it.

Record application checks or file checksums before the backup so the restored data can be
compared with the source. For a Linux guest, run inside the source guest:

```sh
sha256sum <data-file> > /root/migration.sha256
```

For an existing backup, use the recovery records belonging to that snapshot instead.
Follow [Proxmox Backup and Restore](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)
for job configuration and the appropriate consistency method. For a stopped-mode backup,
run on the source node:

```sh
vzdump <source-vmid> --storage <backup-storage> --mode stop
```

**Important:** stopped-mode backup restarts a previously running source guest when it
finishes. Shut down the source again before starting its replacement. Keep the source and
backup until the destination has passed acceptance.

## 3. Restore onto encrypted storage

Commands run as root on the destination. `<encrypted-storage>` is the Proxmox storage ID
configured by the destination's storage guide. Restore an archive by its full path:

```sh
# Virtual machine
qmrestore <archive> <new-vmid> --storage <encrypted-storage>

# Container
pct restore <new-vmid> <archive> --storage <encrypted-storage>
```

For PBS, register the datastore with the source encryption key using the Backup guide.
Select the snapshot in the Proxmox restore interface and explicitly select the encrypted
destination storage. The same destination requirement applies to archives copied to the
node: their location does not determine where restored disks are written.

The APT packages supply the provider configuration and backup wrappers. No additional
`OPENSSL_CONF` prefix or manual wrapper is needed for web-interface or command-line restores.

A restore retains guest settings. Remove SPICE display, audio and USB redirection settings
before starting an affected VM; the packaged verifier lists incompatible settings and their
removal commands:

```sh
wolfssl-fips-pve-verify
```

Continue with [Migration acceptance](WOLFSSL-PROXMOX-FIPS-MIGRATION-ACCEPTANCE.md).
